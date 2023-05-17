# Auto-Tournament Technical Design

Authors: @domino14

Date: May 15, 2023

# Purpose

This design is for automatically running Swiss-style OMGWords tournaments, and making these easily discoverable to our users.


# Background

We have a well-designed tournament module, but it has a learning curve for directors. If we could make more tournaments automatic, we believe it will increase the number of regular players we have and make the experience more fun.

Additionally, tournament discoverability right now is tough. It relies upon checking a calendar on another page on the site (/clubs); this calendar also needs to be updated manually. For recurring tournaments, the director needs to make a tournament every time, tell us to update the calendar with the new link, etc. 

Another approach is to create a separate club page with the sole purpose of having a constant link; the tournament goers then are told via this page where the tournament room for this week is. The Philly and Madison clubs use this method, among other clubs.

These approaches are subideal*. It would be great to have tournaments show up in the front page at least a few hours in advance; allowing users to register for these tournaments. We could then notify registrants when their tournaments are about to begin.

## Product Design

### Registering and checking in to a tournament

First, we want to add a "Sign up" button to a tournament. This will allow the user to sign up for a specific division. Signing up for a tournament should add the user to the list of registrants. This list should be publicly visible.

We want to allow the user to withdraw from a tournament after signing up, but before the tournament starts. This can simply remove them from the list when they do this.

A few minutes (10?) before the tournament starts, we should send a notification to all users online that their tournament is about to begin. We can re-use the tournament status widget for this. Because no-shows are a pain in Swiss (they create byes), we would like a way to confirm a registration -- i.e., "check in" to the tournament. A check-in widget should therefore become available as soon as the notification is sent.

You can still withdraw your check-in if something comes up, until the moment that the tournament actually starts.

Players not checked in before the tournament starts will not be paired in the first round! 

### Starting a tournament

The tournament will automatically start at the given time. This will open all rounds, and give players a Start button like it is now, except with an additional timer. There will be a timer (1 minute?) before the game is auto-started. Once the game is auto-started, or manually started, players will be automatically redirected to the game screen. They will then be able to play their game.

### Removing no-shows

If a player never shows up to one of their games, their clock will tick down and they'll flag. We will apply a penalty to the player via moderation (disallowing them from signing up for another automatic tournament for some period of time). The player should also be removed from future rounds of the tournament at this time.

### Auto-starting round

Once a round is over, there should be a small countdown (30 sec to 1 minute) before the next round opens. 

### Auto-ending tournament

Once the final round is over, the tournament automatically ends.

### Leaderboard scoring

We can track total points instead of W-L record, which might be a little more meaningless.

- One point for a win or bye
- Half a point for a tie
- Byes are only given for players who joined on time but were not paired due to oddness or other reasons.

Tiebreaks will be done by number of games actually played, and then spread. For example, someone who only played 2 games and won them both should never place ahead of someone who went 2-3, because the latter person played 5 games.

### Calendar

Automatic Swiss tournaments should be set on a repeating schedule. Today's tournaments should be added to the front page. These can also be updated in the Google calendar automatically. The tournament room should be created several hours in advance of the tournament actually beginning. A new widget with ongoing/past/upcoming tournaments should be put on the front page.

# Glossary of terms

**Swiss** - a pairing format used by tournaments.

# Technical Design

## Modifications to existing tournament module
---

Our existing tournament module will handle a lot of what we need, but we want to make several modifications.

### Registrants

We need to refactor the existing "registrants" table. This is not really currently used. It looks like the following:

```sql
CREATE TABLE public.registrants (
    user_id text,
    tournament_id text,
    division_id text
);
```

This can be enhanced by the following:

- A `BOOLEAN` column `checked_in`. If this is false, the user has registered, but not checked-in before the tournament begins.
- A `timestamptz` column `updated_at`.

Once a tournament is over, the registrations for it should be deleted; they're not necessary anymore.

### Tournament type flag

We should add a new Tournament Type:

```proto
enum TType {
  // A Standard tournament
  STANDARD = 0;
  // A new "clubhouse"
  CLUB = 1;
  // A club session or child tournament.
  CHILD = 2;
  // A legacy tournament
  LEGACY = 3;
  // Adding:
  STANDARD_AUTO = 4;  // if this is an AUTO tournament, there is no need for a director.
}
```

### Tournament timers

We need a number of new automatic timers that need to be resilient to app deploys/restarts.

First, the overall timer for the tournament. Let's say a tournament starts at 8 PM. We will re-use the "DefaultSettings" JSONB column in the tournament database model, which as far as we can tell, is not currently used for anything. We can store the UTC timestamp here for the start time (8 PM on whatever day), as well as other timer-related settings; for example, how many seconds to wait between rounds, how many seconds to wait until auto-start, etc.

**Phase 1**:

We will have an hourly timer running in the main bus thread. This timer spawns a goroutine that looks at the database for tournaments of type `STANDARD_AUTO`, and looks for their start time in the above column. Once the start time is within 12 hours (or some similar time), another goroutine will be spawned with an actual countdown. The front end should also show the time remaining as a countdown. There should be a map created on the backend with tournament ID + timer type as a key and a boolean as a value, indicating that a timer goroutine has started. When the hourly timer runs again, it will see that a timer has already started for this.

If a re-deploy is done after the countdown goroutine has started, we will need to start the timer again. In the initialization routine, we should check for tournament timers and start any ones that are missing. To avoid race conditions, we can start the timer if it is past due, and if the tournament hasn't started yet.


**Phase 2**:

For the MVP, we may not need to do this. Once we are multi-server we are going to want to use a more persistent timer solution. We will try this library, or a similar solution:

https://github.com/spy16/delayq

We can use Redis as a backing store and poll every second or so, with a dedicated scheduler 
kmicroservice. The microservice can then push out messages on NATS to communicate with the API service.

Or we can just use Redis as a map storing which servers have started Goroutine-based timers. There are a few possible solutions.

**Notes**:

Note that this still requires the tournament to be created ahead of time, even if it hasn't started. We can have a script or other method create a bunch of tournaments ahead of time. A better solution would be to have the concept of scheduled tournaments/club sessions/etc. Perhaps that can be done for Phase 2.

## Modifications to gameplay module
---
### Ready signal

When players join a game, the socket disconnects and reconnects in the new page. This is a legacy issue that we have not yet gotten around to fixing. Ideally, we would maintain a single socket connection as we navigate through the SPA, and just change the context/subscription as we move to different pages.

The front end then sends a "Ready" signal for each player via the socket. When the backend receives a Ready from all players, it starts the game.

For some people with bad connections, the socket never reconnects. The app will then cancel the game after some time (30 seconds to a minute). What this looks like to the user is a game that never starts. At some point they'll get sick of waiting, go back to the main tournament, and then have a "Start" button show up again. This is obviously unideal though for an automatic tournament scenario.

We propose to change this behavior in the following ways:

- For an automatic tournament, start the game anyway without a ready signal
- When a player joins the new table, show a pop-up that says Connecting... or similar.
- After a few seconds, the pop-up should prompt the user to refresh the page, with perhaps a button to refresh.


## Notification system
---

We currently have a tournament widget for users that can show the time remaining till the next round, etc. But it would be good to see something similar if you are not in the tournament.

A registered user should get a notification when a tournament they are registered for is about to start (when check-in is open). We can show the tournament widget at this time with an audio alert.

Upon refresh, the tournament widget should still show up. This can be handled via the backend finding immediately upcoming tournaments for users upon first load.

## Calendar
---

The frontend should show upcoming and past automatic tournaments in a widget on the front page, just below the lobby. This can be a simple "linear" calendar - i.e. just a list of events with times.


# Implementation Plan

A list of the “user stories” / tickets that are going to be created. Ideally these tickets can be copied straight into Github Issues for us to work on. 

# Appendix

Any other diagrams, etc that can help elucidate anything better
