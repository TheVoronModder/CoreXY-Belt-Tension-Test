# CoreXY-Belt-Tension-Test
Belt Tension Test utilizing printed methodology to ensure near-perfect belt tension for those with no so great hearing.


>[!IMPORTANT]
>zero-guesswork print test that isolates each CoreXY belt with real extrusion so the plastic tells you which belt needs more tension—no sound, no phone apps.
>It prints two short diagonal “bars”
><BR>
>Left bar = “A-only” moves (X+Y, +45°) → isolates the A belt/motor.
>Right bar = “B-only” moves (X−Y, −45°) → isolates the B belt/motor.
>Each bar is built from parallel extruded strokes in one diagonal direction only. Travel returns are non-extruding and don’t affect the test.
>Length is hard-capped to 100 mm, with auto-clamps so you won’t go out of range.
>Flow is computed from line width × layer height, so it won’t overfeed the hotend.

>[!NOTE]
>What to look for
>The bar that shows more wobble/ghosting, ragged ends, or shorter-than-commanded length (tip-to-tip along the stroke direction) points to the looser belt. Tighten that belt a little, reprint, and match the results.

>[!WARNING]
>THINGS TO ADJUST!
>HOTEND and BED temp

```

