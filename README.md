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
  {% set L_req = (len|float if (len|float) <= 100.0 else 100.0) %}

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

  # Start positions (front-left safe zone)
  {% set Ax = M + 25.0 %}
  {% set Ay = M + bar_width + 10.0 %}
  {% set Bx = Ax + 40.0 %}
  {% set By = Ay + bar_width + 20.0 %}

  # ---- Clamp stroke length per bar ----
  # A bar forward vector = (+X, +Y); usable room from start:
  {% set A_x_room = max(0.0, XMAX - M - Ax) %}
  {% set A_y_room = max(0.0, YMAX - M - Ay) %}
  {% set Lmax_A = 1.41421356 * (A_x_room if A_x_room < A_y_room else A_y_room) %}
  {% set L_A = (L_req if L_req <= Lmax_A else Lmax_A) %}
  {% set L_A_xy = L_A / 1.41421356 %}
  {% if L_A < 5.0 %}{% set L_A = 5.0 %}{% set L_A_xy = L_A/1.41421356 %}{% endif %}

  # B bar forward vector = (+X, -Y)
  {% set B_x_room = max(0.0, XMAX - M - Bx) %}
  {% set B_y_room = max(0.0, By - M) %}
  {% set Lmax_B = 1.41421356 * (B_x_room if B_x_room < B_y_room else B_y_room) %}
  {% set L_B = (L_req if L_req <= Lmax_B else Lmax_B) %}
  {% set L_B_xy = L_B / 1.41421356 %}
  {% if L_B < 5.0 %}{% set L_B = 5.0 %}{% set L_B_xy = L_B/1.41421356 %}{% endif %}

  {% set e_move_A = e_per_mm * L_A %}
  {% set e_move_B = e_per_mm * L_B %}

  # Required lanes across bar_width
  {% set passes_req = (bar_width / spacing)|int + 1 %}

  # ---- Pass limits that include forward reach (this fixes out-of-range) ----
  # A steps +X -Y between lanes; farthest X reached is Ax + step*(p-1) + L_A_xy
  {% set A_pass_xlim = ((XMAX - M - Ax - L_A_xy) / step)|int + 1 %}
  {% set A_pass_ylim = ((Ay - M) / step)|int + 1 %}
  {% set passes_A = passes_req %}
  {% if passes_A > A_pass_xlim %}{% set passes_A = A_pass_xlim %}{% endif %}
  {% if passes_A > A_pass_ylim %}{% set passes_A = A_pass_ylim %}{% endif %}
  {% if passes_A < 1 %}{% set passes_A = 1 %}{% endif %}

  # B steps +X +Y between lanes; farthest X is Bx + step*(p-1) + L_B_xy; farthest Y is By + step*(p-1)
  {% set B_pass_xlim = ((XMAX - M - Bx - L_B_xy) / step)|int + 1 %}
  {% set B_pass_ylim = ((YMAX - M - By) / step)|int + 1 %}
  {% set passes_B = passes_req %}
  {% if passes_B > B_pass_xlim %}{% set passes_B = B_pass_xlim %}{% endif %}
  {% if passes_B > B_pass_ylim %}{% set passes_B = B_pass_ylim %}{% endif %}
  {% if passes_B < 1 %}{% set passes_B = 1 %}{% endif %}

  # Modes
  {% set was_abs = printer.gcode_move.absolute_coordinates %}
  {% set abs_e = printer.gcode_move.absolute_extrude %}

  G90
  G92 E0
  G1 Z{lh} F{travel_f}

  # ===== BAR A (X+ Y+) =====
  G1 X{Ax} Y{Ay} F{travel_f}
  {% if abs_e %} G1 E{prime_e} F{prime_f} {% set eA = prime_e|float %}{% else %} G1 E{prime_e} F{prime_f} {% endif %}
  {% for L in range(layers) %}
    G91
    {% for i in range(passes_A) %}
      {% if abs_e %}{% set eA = eA + e_move_A %} G1 X{L_A_xy} Y{L_A_xy} E{eA} F{line_f}
      {% else %} G1 X{L_A_xy} Y{L_A_xy} E{e_move_A} F{line_f}{% endif %}
      G1 X{-L_A_xy} Y{-L_A_xy} F{travel_f}
      {% if i < (passes_A - 1) %} G1 X{step} Y{-step} F{travel_f} {% endif %}
    {% endfor %}
    G1 X{-step*(passes_A-1)} Y{step*(passes_A-1)} F{travel_f}
    G90
    G1 Z{lh*(L+2)} F{travel_f}
    G1 X{Ax} Y{Ay} F{travel_f}
  {% endfor %}

  # ===== BAR B (X+ Y-) =====
  G1 X{Bx} Y{By} F{travel_f}
  {% if abs_e %}{% set eB = 0.0 %}{% endif %}
  {% for L in range(layers) %}
    G91
    {% for i in range(passes_B) %}
      {% if abs_e %}{% set eB = eB + e_move_B %} G1 X{L_B_xy} Y{-L_B_xy} E{eB} F{line_f}
      {% else %} G1 X{L_B_xy} Y{-L_B_xy} E{e_move_B} F{line_f}{% endif %}
      G1 X{-L_B_xy} Y{L_B_xy} F{travel_f}
      {% if i < (passes_B - 1) %} G1 X{step} Y{step} F{travel_f} {% endif %}
    {% endfor %}
    G1 X{-step*(passes_B-1)} Y{-step*(passes_B-1)} F{travel_f}
    G90
    G1 Z{lh*(layers + L + 2)} F{travel_f}
    G1 X{Bx} Y{By} F{travel_f}
  {% endfor %}

  # ===== Labels (clamped to fit) =====
  {% set label_h = label_h|float %}
  {% set label_w = label_h * (label_w_frac|float) %}
  {% set label_z = (layers * lh) + label_lift %}
  {% set e_text_per_mm = e_per_mm %}
  {% set eV = e_text_per_mm * label_h %}
  {% set eW = e_text_per_mm * label_w %}

  # Place A label
  {% set Aox_raw = Ax %}
  {% set Aoy_raw = Ay + bar_width + 2.0 %}
  {% set Aox = Aox_raw if Aox_raw <= (XMAX - M - label_w) else (XMAX - M - label_w) %}
  {% set Aoy = Aoy_raw if Aoy_raw <= (YMAX - M - label_h) else (YMAX - M - label_h) %}
  G90
  G1 Z{label_z} F{travel_f}
  G1 X{Aox} Y{Aoy} F{travel_f}
  G91
  {% if abs_e %}
    {% set eAt = eV %} G1 Y{label_h} E{eAt} F{label_f}
    G1 X{label_w} F{travel_f}
    {% set eAt = eAt + eV %} G1 Y{-label_h} E{eAt} F{label_f}
    G1 Y{label_h/2.0} F{travel_f}
    {% set eAt = eAt + eW %} G1 X{-label_w} E{eAt} F{label_f}
  {% else %}
    G1 Y{label_h} E{eV} F{label_f}
    G1 X{label_w} F{travel_f}
    G1 Y{-label_h} E{eV} F{label_f}
    G1 Y{label_h/2.0} F{travel_f}
    G1 X{-label_w} E{eW} F{label_f}
  {% endif %}
  G90

  # Place B label
  {% set Box_raw = Bx %}
  {% set Boy_raw = By + bar_width + 2.0 %}
  {% set Box = Box_raw if Box_raw <= (XMAX - M - label_w) else (XMAX - M - label_w) %}
  {% set Boy = Boy_raw if Boy_raw <= (YMAX - M - label_h) else (YMAX - M - label_h) %}
  G1 Z{label_z} F{travel_f}
  G1 X{Box} Y{Boy} F{travel_f}
  G91
  {% if abs_e %}
    {% set eBt = eV %} G1 Y{label_h} E{eBt} F{label_f}
    {% set eBt = eBt + eW %} G1 X{label_w} E{eBt} F{label_f}
    {% set eBt = eBt + (e_text_per_mm * (label_h/2.0)) %} G1 Y{-label_h/2.0} E{eBt} F{label_f}
    G1 X{-label_w} F{travel_f}
    {% set eBt = eBt + eW %} G1 X{label_w} E{eBt} F{label_f}
    {% set eBt = eBt + (e_text_per_mm * (label_h/2.0)) %} G1 Y{-label_h/2.0} E{eBt} F{label_f}
    {% set eBt = eBt + eW %} G1 X{-label_w} E{eBt} F{label_f}
  {% else %}
    G1 Y{label_h} E{eV} F{label_f}
    G1 X{label_w} E{eW} F{label_f}
    G1 Y{-label_h/2.0} E{e_text_per_mm * (label_h/2.0)} F{label_f}
    G1 X{-label_w} F{travel_f}
    G1 X{label_w} E{eW} F{label_f}
    G1 Y{-label_h/2.0} E{e_text_per_mm * (label_h/2.0)} F{label_f}
    G1 X{-label_w} E{eW} F{label_f}
  {% endif %}
  G90

  G92 E0
  {% if was_abs %} G90 {% else %} G91 {% endif %}

```
