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
# ---------------- User-tunables ----------------
variable_margin: 10.0
variable_len: 80.0
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
# Label style
variable_label_h: 6.0
variable_label_w_frac: 0.70
variable_label_lift: 0.30
variable_label_f: 600
# ------------------------------------------------

gcode:
  {% set bed_target = (params.BED|default(-1))|float %}
  {% set hot_target = (params.HOTEND|default(-1))|float %}

  {% set XMAX = printer.toolhead.axis_maximum.x|float %}
  {% set YMAX = printer.toolhead.axis_maximum.y|float %}
  {% set M = margin|float %}
  {% set L_req = len|float %}
  {% if L_req > 100.0 %}{% set L_req = 100.0 %}{% endif %}

  {% set homed = printer.toolhead.homed_axes %}
  {% if 'x' not in homed or 'y' not in homed or 'z' not in homed %}
    G28
  {% endif %}

  {% if bed_target >= 0 %}
    M140 S{bed_target}
    M190 S{bed_target}
  {% endif %}
  {% if hot_target >= 0 %}
    M104 S{hot_target}
    M109 S{hot_target}
  {% endif %}

  {% set lw = line_w|float %}
  {% set lh = layer_h|float %}
  {% set fd = fil_d|float %}
  {% set flow = flow|float %}
  {% set fil_area = 3.1415926535 * (fd*0.5) * (fd*0.5) %}
  {% if fil_area <= 0 %}{% set fil_area = 2.405 %}{% endif %}
  {% set e_per_mm = (lw * lh * flow) / fil_area %}

  {% set min_ext = printer.configfile.settings.extruder.min_extrude_temp|float %}
  {% if hot_target < 0 and printer.extruder.temperature < (min_ext + 5.0) %}
    RESPOND PREFIX="BELT" MSG="Extruder cold: running motion-only. Pass HOTEND=xxx BED=yyy to print plastic."
    {% set e_per_mm = 0.0 %}
    {% set prime_e = 0.0 %}
  {% endif %}

  {% set spacing = lw * 0.90 %}
  {% set step = spacing / 1.41421356 %}
  {% set layers = layers|int %}
  {% if layers < 1 %}{% set layers = 1 %}{% endif %}

  {% set Ax = M + 25.0 %}
  {% set Ay = M + bar_width + 10.0 %}
  {% set Bx = Ax + 40.0 %}
  {% set By = Ay + bar_width + 20.0 %}

  {% set A_x_room = XMAX - M - Ax %}
  {% set A_y_room = YMAX - M - Ay %}
  {% if A_x_room < 0 %}{% set A_x_room = 0 %}{% endif %}
  {% if A_y_room < 0 %}{% set A_y_room = 0 %}{% endif %}
  {% set Lmax_A = 1.41421356 * (A_x_room if A_x_room < A_y_room else A_y_room) %}
  {% set L_A = (L_req if L_req <= Lmax_A else Lmax_A) %}
  {% if L_A < 5.0 %}{% set L_A = 5.0 %}{% endif %}

  {% set B_x_room = XMAX - M - Bx %}
  {% set B_y_room = By - M %}
  {% if B_x_room < 0 %}{% set B_x_room = 0 %}{% endif %}
  {% if B_y_room < 0 %}{% set B_y_room = 0 %}{% endif %}
  {% set Lmax_B = 1.41421356 * (B_x_room if B_x_room < B_y_room else B_y_room) %}
  {% set L_B = (L_req if L_req <= Lmax_B else Lmax_B) %}
  {% if L_B < 5.0 %}{% set L_B = 5.0 %}{% endif %}

  {% set e_move_A = e_per_mm * L_A %}
  {% set e_move_B = e_per_mm * L_B %}

  {% set passes_req = (bar_width / spacing)|int + 1 %}

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

  {% set was_abs = printer.gcode_move.absolute_coordinates %}
  {% set abs_e = printer.gcode_move.absolute_extrude %}

  G90
  G92 E0
  G1 Z{lh} F{travel_f}

  ; ----- BAR A (A-only: +45°, X+ Y+) -----
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

  G91
  G1 X10 F{travel_f}
  G90

  ; ----- BAR B (B-only: -45°, X+ Y-) -----
  G1 X{Bx} Y{By} F{travel_f}
  {% if abs_e %}
    {% set eB = 0.0 %}
  {% endif %}
  {% for L in range(layers) %}
    G91
    {% for i in range(passes_B) %}
      {% if abs_e %}
        {% set eB = eB + e_move_B %}
        G1 X{L_B/1.41421356} Y{-L_B/1.41421356} E{eB} F{line_f}
      {% else %}
        G1 X{L_B/1.41421356} Y{-L_B/1.41421356} E{e_move_B} F{line_f}
      {% endif %}
      G1 X{-L_B/1.41421356} Y{L_B/1.41421356} F{travel_f}
      {% if i < (passes_B - 1) %}
        G1 X{step} Y{step} F{travel_f}
      {% endif %}
    {% endfor %}
    G1 X{-step*(passes_B-1)} Y{-step*(passes_B-1)} F{travel_f}
    G90
    G1 Z{lh*(layers + L + 2)} F{travel_f}
    G1 X{Bx} Y{By} F{travel_f}
  {% endfor %}

  ; ----- Labels on higher Z -----
  {% set label_h = label_h|float %}
  {% set label_w = label_h * (label_w_frac|float) %}
  {% set label_z = (layers * lh) + label_lift %}
  {% set e_text_per_mm = e_per_mm %}

  {% set Aox = Ax %}
  {% set Aoy = Ay + bar_width + 2.0 %}
  G90
  G1 Z{label_z} F{travel_f}
  G1 X{Aox} Y{Aoy} F{travel_f}
  G91
  {% set e_segV = e_text_per_mm * label_h %}
  {% set e_segH = e_text_per_mm * label_w %}
  {% if abs_e %}
    {% set eA_t = e_segV %}
    G1 Y{label_h} E{eA_t} F{label_f}
    G1 X{label_w} F{travel_f}
    {% set eA_t = eA_t + e_segV %}
    G1 Y{-label_h} E{eA_t} F{label_f}
    G1 Y{label_h/2.0} F{travel_f}
    {% set eA_t = eA_t + e_segH %}
    G1 X{-label_w} E{eA_t} F{label_f}
  {% else %}
    G1 Y{label_h} E{e_segV} F{label_f}
    G1 X{label_w} F{travel_f}
    G1 Y{-label_h} E{e_segV} F{label_f}
    G1 Y{label_h/2.0} F{travel_f}
    G1 X{-label_w} E{e_segH} F{label_f}
  {% endif %}
  G90

  {% set Box = Bx %}
  {% set Boy = By + bar_width + 2.0 %}
  G1 Z{label_z} F{travel_f}
  G1 X{Box} Y{Boy} F{travel_f}
  G91
  {% if abs_e %}
    {% set eB_t = e_segV %}
    G1 Y{label_h} E{eB_t} F{label_f}
    {% set eB_t = eB_t + e_segH %}
    G1 X{label_w} E{eB_t} F{label_f}
    {% set eB_t = eB_t + (e_text_per_mm * (label_h/2.0)) %}
    G1 Y{-label_h/2.0} E{eB_t} F{label_f}
    G1 X{-label_w} F{travel_f}
    {% set eB_t = eB_t + e_segH %}
    G1 X{label_w} E{eB_t} F{label_f}
    {% set eB_t = eB_t + (e_text_per_mm * (label_h/2.0)) %}
    G1 Y{-label_h/2.0} E{eB_t} F{label_f}
    {% set eB_t = eB_t + e_segH %}
    G1 X{-label_w} E{eB_t} F{label_f}
  {% else %}
    G1 Y{label_h} E{e_segV} F{label_f}
    G1 X{label_w} E{e_segH} F{label_f}
    G1 Y{-label_h/2.0} E{e_text_per_mm * (label_h/2.0)} F{label_f}
    G1 X{-label_w} F{travel_f}
    G1 X{label_w} E{e_segH} F{label_f}
    G1 Y{-label_h/2.0} E{e_text_per_mm * (label_h/2.0)} F{label_f}
    G1 X{-label_w} E{e_segH} F{label_f}
  {% endif %}
  G90

  G92 E0
  {% if was_abs %} G90 {% else %} G91 {% endif %}

```
