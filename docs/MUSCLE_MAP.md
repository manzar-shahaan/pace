# The muscle map

Pace's home screen is a tappable front/back muscle map (`body-highlighter`).
Tap regions to narrow a shortlist of your own exercises for that area
(prioritized), plus unlogged suggestions sourced from wger
(de-emphasized), each with a preview image.

Regions you haven't trained in a while pulse (white → grey → black →
hold, repeating). That's deliberate: the map doubles as a "what needs
work" prompt, so the first thing you see every time you open Pace is the
area you've been neglecting — before you decide what to do today.

A Strength/Cardio switch in the map's header (`home.js`) jumps straight
to the add-exercise form pre-set to "Time & distance" + the Cardio tag,
since a timed activity has no regions to tap.

Region tags only apply to resistance exercises — endurance entries carry
no region tags, so a run never makes a muscle region look fresh. Tap
order sets region priority (1 = primary, 2 = secondary, ...), and the 17
regions are fixed vocabulary seeded from `utils/body_regions.py`, keyed
to the map's SVG.
