# CoreXY-Belt-Tension-Test
Belt Tension Test utilizing printed methodology to ensure near-perfect belt tension for those with no so great hearing.


>[!IMPORTANT]
>zero-guesswork print test that isolates each CoreXY belt with real extrusion so the plastic tells you which belt needs more tension—no sound, no phone apps.
>It prints two short diagonal “bars”
><br>
>Left bar = “A-only” moves (X+Y, +45°) → isolates the A belt/motor.
><br>
>Right bar = “B-only” moves (X−Y, −45°) → isolates the B belt/motor.
><br>
>Each bar is built from parallel extruded strokes in one diagonal direction only. Travel returns are non-extruding and don’t affect the test.
><br>
>Length is hard-capped to 100 mm, with auto-clamps so you won’t go out of range.
>Flow is computed from line width × layer height, so it won’t overfeed the hotend.

>[!NOTE]
>What to look for
>The bar that shows more wobble/ghosting, ragged ends, or shorter-than-commanded length (tip-to-tip along the stroke direction) points to the looser belt. Tighten that belt a little, reprint, and match the results.

>[!WARNING]
>Things to know!
>
>You must MANUALLY enter your desired HOTEND / BED temps before running this macro:
><br>
>For example:
><br>
>I print ASA so my specific command is as follows:
><br>
>```BELT_TENSION_TEST HOTEND=245 BED=110```

```
[gcode_macro BELT_TENSION_TEST]
# description: CoreXY belt test (A-only and B-only bars) with labels; supports HOTEND= and BED=
# -------- User-tunables --------
variable_margin: 10.0
variable_len: 80.0                 # ≤100 enforced
variable_bar_width: 12.0
variable_layers: 3
variable_layer_h: 0.25
variable_line_w: 0.60
variable_line_f: 600
variable_travel_f: 6000
variable_prime_e: 6.0
variable_prime_f: 120
variable_fil_d: 1.75
variable_flow: 1.00
# Labels
variable_label_h: 6.0
variable_label_w_frac: 0.70
variable_label_lift: 0.30
variable_label_f: 600
# --------------------------------

gcode:
  {% set bed_target = (params.BED|default(-1))|float %}
  {% set hot_target = (params.HOTEND|default(-1))|float %}

  {% set XMAX = printer.toolhead.axis_maximum.x|float %}
  {% set YMAX = printer.toolhead.axis_maximum.y|float %}
  {% set M = margin|float %}
  {% set L_in = params.LEN|default(len)|float %}
  {% set L_req = (L_in if L_in <= 100.0 else 100.0) %}

  # Home if needed
  {% set homed = printer.toolhead.homed_axes %}
  {% if 'x' not in homed or 'y' not in homed or 'z' not in homed %}
    G28
  {% endif %}

  # Optional heating
  {% if bed_target >= 0 %} M140 S{bed_target} {% endif %}
  {% if bed_target >= 0 %} M190 S{bed_target} {% endif %}
  {% if hot_target >= 0 %} M104 S{hot_target} {% endif %}
  {% if hot_target >= 0 %} M109 S{hot_target} {% endif %}

  # Flow math
  {% set lw = line_w|float %}
  {% set lh = layer_h|float %}
  {% set fd = fil_d|float %}
  {% set flow = flow|float %}
  {% set fil_area = 3.1415926535 * (fd*0.5) * (fd*0.5) %}
  {% if fil_area <= 0 %}{% set fil_area = 2.405 %}{% endif %}
  {% set e_per_mm = (lw * lh * flow) / fil_area %}

  # Cold nozzle → motion-only
  {% set min_ext = printer.configfile.settings.extruder.min_extrude_temp|float %}
  {% if hot_target < 0 and printer.extruder.temperature < (min_ext + 5.0) %}
    RESPOND PREFIX="BELT" MSG="Extruder cold: motion-only. Pass HOTEND=xxx BED=yyy to print plastic."
    {% set e_per_mm = 0.0 %}
    {% set prime_e = 0.0 %}
  {% endif %}

  {% set spacing = lw * 0.90 %}
  {% set step = spacing / 1.41421356 %}
  {% set layers = (layers|int if (layers|int) > 0 else 1) %}

  # ---------- RAW start positions (front-left) ----------
  {% set Ax_raw = M + 25.0 %}
  {% set Ay_raw = M + bar_width + 10.0 %}
  {% set Bx_raw = Ax_raw + 40.0 %}
  {% set By_raw = Ay_raw + bar_width + 20.0 %}

  # ---------- Stroke lengths from remaining room ----------
  # A bar (+X,+Y): room clamps without max()
  {% set _tmp = XMAX - M - Ax_raw %}
  {% if _tmp < 0 %}{% set A_x_room = 0.0 %}{% else %}{% set A_x_room = _tmp %}{% endif %}
  {% set _tmp = YMAX - M - Ay_raw %}
  {% if _tmp < 0 %}{% set A_y_room = 0.0 %}{% else %}{% set A_y_room = _tmp %}{% endif %}
  {% set Lmax_A = 1.414213_

```
