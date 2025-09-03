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
>Things to know!
>
>You must MANUALLY enter your desired HOTEND / BED temps before running this macro:
>
>For example:
>
>I print ASA so my specific command is as follows:

```BELT_TENSION_TEST HOTEND=250 BED=110```

```
[gcode_macro BELT_TENSION_TEST]
# description: CoreXY belt test (A-only and B-only bars) with labels; supports HOTEND= and BED=
# ---------------- User-tunables ----------------
variable_margin: 10.0            # mm keep-away from bed edges
variable_len: 80.0               # requested bar stroke length along the diagonal (≤100 enforced)
variable_bar_width: 12.0         # mm bar width (perpendicular to stroke)
variable_layers: 3               # bar layers (first-layer style)
variable_layer_h: 0.25           # mm
variable_line_w: 0.60            # mm line width for bars
variable_line_f: 600             # mm/min drawing speed (10 mm/s)
variable_travel_f: 6000          # mm/min travel speed
variable_prime_e: 6.0            # mm gentle prime before each bar
variable_prime_f: 120            # mm/min (2 mm/s) prime speed
variable_fil_d: 1.75             # mm filament diameter
variable_flow: 1.00              # 1.0 = 100% flow

# Label style (drawn AFTER bars on a higher Z)
variable_label_h: 6.0            # mm text height
variable_label_w_frac: 0.70      # width = frac * height (0.7 looks good)
variable_label_lift: 0.30        # mm above last bar layer to draw labels
variable_label_f: 600            # mm/min text drawing speed (10 mm/s)
# ------------------------------------------------

gcode:
  # ---- Parse optional temps ----
  {% set bed_target = (params.BED|default(-1))|float %}
  {% set hot_target = (params.HOTEND|default(-1))|float %}

  # ---- Bed & safety ----
  {% set XMAX = printer.toolhead.axis_maximum.x|float %}
  {% set YMAX = printer.toolhead.axis_maximum.y|float %}
  {% set M = margin|float %}
  {% set L_req = len|float %}
  {% if L_req > 100.0 %}{% set L_req = 100.0 %}{% endif %}

  # Home if needed to avoid out-of-range moves
  {% set homed = printer.toolhead.homed_axes %}
  {% if 'x' not in homed or 'y' not in homed or 'z' not in homed %}
    G28
  {% endif %}

  # ---- Optional heating (bed first, then hotend) ----
  {% if bed_target >= 0 %}
    M140 S{bed_target}
    M190 S{bed_target}
  {% endif %}
  {% if hot_target >= 0 %}
    M104 S{hot_target}
    M109 S{hot_target}
  {% endif %}

  # ---- Geometry & flow (bars) ----
  {% set lw = line_w|float %}
  {% set lh = layer_h|float %}
  {% set fd = fil_d|float %}
  {% set flow = flow|float %}
  {% set fil_area = 3.1415926535 * (fd*0.5) * (fd*0.5) %}
  {% if fil_area <= 0 %}{% set fil_area = 2.405 %}{% endif %}
  {% set e_per_mm = (lw * lh * flow) / fil_area %}

  # If user didn’t heat and nozzle is cold, do motion-only (no extrusion)
  {% set min_ext = printer.configfile.settings.extruder.min_extrude_temp|float %}
  {% if hot_target < 0 and printer.extruder.temperature < (min_ext + 5.0) %}
    RESPOND PREFIX="BELT" MSG="Extruder cold: running motion-only. Pass HOTEND=xxx BED=yyy to print plastic."
    {% set e_per_mm = 0.0 %}
    {% set prime_e = 0.0 %}
  {% endif %}

  {% set spacing = lw * 0.90 %}
  {% set step = spacing / 1.41421356 %}     # perpendicular step components for 45° bars

  {% set layers = layers|int %}
  {% if layers < 1 %}{% set layers = 1 %}{% endif %}

  # ---- Start positions (kept safely in-bounds) ----
  {% set Ax = M + 25.0 %}
  {% set Ay = M + bar_width + 10.0 %}
  {% set Bx = Ax + 40.0 %}
  {% set By = Ay + bar_width + 20.0 %}

  # ---- Clamp stroke length to fit within bed for each bar ----
  # A bar goes +X +Y; stepping +X -Y
  {% set A_x_room = XMAX - M - Ax %}
  {% set A_y_room = YMAX - M - Ay %}
  {% if A_x_room < 0 %}{% set A_x_room = 0 %}{% endif %}
  {% if A_y_room < 0 %}{% set A_y_room = 0 %}{% endif %}
  {% set Lmax_A = 1.41421356 * (A_x_room if A_x_room < A_y_room else A_y_room) %}
  {% set L_A = (L_req if L_req <= Lmax_A else Lmax_A) %}
  {% if L_A < 5.0 %}{% set L_A = 5.0 %}{% endif %}

  # B bar goes +X -Y; stepping +X +Y
  {% set B_x_room = XMAX - M - Bx %}
  {% set B_y_room = By - M %}
  {% if B_x_room < 0 %}{% set B_x_room = 0 %}{% endif %}
  {% if B_y_room < 0 %}{% set B_y_room = 0 %}{% endif %}
  {% set Lmax_B = 1.41421356 * (B_x_room if B_x_room < B_y_room else B_y_room) %}
  {% set L_B = (L_req if L_req <= Lmax_B else Lmax_B) %}
  {% if L_B < 5.0 %}{% set L_B = 5.0 %}{% endif %}

  {% set e_move_A = e_per_mm * L_A %}
  {% set e_move_B = e_per_mm * L_B %}

  # Number of strokes across bar width
  {% set passes_req = (bar_width / spacing)|int + 1 %}

  # Limit passes so stepping stays in-bounds
  {% set passes_Ax = ((XMAX - M - Ax) / step)|int %}
  {% set passes_Ay = ((Ay - M) / step)|int %}
  {% set passes_A = passes_req if passes_req <= passes_Ax else passes_Ax %}
  {% set passes_A = passes_A if passes_A <= passes_Ay else passes_Ay %}
  {% if passes_A < 1 %}{% set passes_A = 1 %}{% endif %}

  {% set passes_Bx = ((XMAX - M - Bx) / step)|int %}
  {% set passes_By = ((YMAX - M - By) / step)|int %}
  {% set passes_B = passes_req if passes_req <= passes_Bx else passes_Bx %}
  {% set passes_B = passes_B if passes_B <= passes_By else passes_By %}
  {% if passes_B < 1 %}{% set passes_B = 1 %}{% endif %}

  # Respect current modes; only switch positioning (not extrusion)
  {% set was_abs = printer.gcode_move.absolute_coordinates %}
  {% set abs_e = printer.gcode_move.absolute_extrude %}

  # ---------------- Print Bars ----------------
  G90
  G92 E0
  G1 Z{lh} F{travel_f}

  # ----- BAR A (A-only: +45°, X+ Y+) -----
  G1 X{Ax} Y{Ay} F{travel_f}
  {% if abs_e %}
    G1 E{prime_e} F{prime_f}
    {% set eA = prime_e|float %}
  {% else %}
    G1 E{prime_e} F{prime_f}
  {% endif %}
  {% for L in range(layers) %}
    G91
    {% for i in range(passes_A) %}
      {% if abs_e %}
        {% set eA = eA + e_move_A %}
        G1 X{L_A/1.41421356} Y{L_A/1.41421356} E{eA} F{line_f}
      {% else %}
        G1 X{L_A/1.41421356} Y{L_A/1.41421356} E{e_move_A} F{line_f}
      {% endif %}
      G1 X{-L_A/1.41421356} Y{-L_A/1.41421356} F{travel_f}
      {% if i < (passes_A - 1) %}
        G1 X{step} Y{-step} F{travel_f}
      {% endif %}
    {% endfor %}
    G1 X{-step*(passes_A-1)} Y{step*(passes_A-1)} F{travel_f}
    G90
    G1 Z{lh*(L+2)} F{travel_f}
    G1 X{Ax} Y{Ay} F{travel_f}
  {% endfor %}

  ; Small clear move
  G91
  G1 X10 F{
```
