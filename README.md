# klipper-macros

This is a collection of macros for the
[Klipper 3D printer firmware](https://github.com/Klipper3d/klipper). I
originally created this repo just to have a consistent set of macros shared
between my own 3D printers. But since I've found them useful, I thought other
people might as well.

## What can I do with these?

Most of these macros just improve basic functionality (e.g.
[selectable build sheets](#bed-surface)) and Klipper compatability
with g-code targeting Marlin printers. However, there are also some nice extras:

* **[Schedule commands at heights and layer changes](#layer-triggers)** -
  This is similar to what your slicer can already do, but I find it simpler, and
  you can schedule these commands while a print is active. As an example of
  usage, I added an [LCD menu item](#lcd-menus) to pause the print at the next
  layer change. This way the pause won't mar the print by e.g. pausing inside
  an external perimeter.
* **Dynamically scale [heaters](#heaters) and [fans](#fans)** - This makes it
  easy to do things like persistently adjust fan settings during a live print,
  or maintain simpler slicer profiles by moving things like a heater bump for a
  hardened steel nozzle into state stored on the printer.
* **Cleaner [LCD menu interface](#lcd-menus)** - I've simplified the menus and
  provided a much easier way to customize materials in the LCD menu (or at least
  I think so). I've also added confirmation dialogs for commands that would
  abort an active print.
* **[Optimized mesh bed leveling](#bed-mesh)** - Probes only within the printed
  area, which can save a lot of time on smaller prints.

## A few warnings...

* You must have `bed_heater` and `extruder` sections configured, otherwise the
  macros won't even load. The Klipper macro system makes it impossible to handle
  this without adding more end-user configuration, so I decided not to bother.
* The multi-extruder and chamber heater functionality is very under-tested and
  may have bugs, since I haven't used it much at all. Patches welcome.
* There's probably other stuff I haven't used enough to thoroughly, so use
  at your own risk.

# Installation

To install the macros first clone this repository inside of your
`klipper_config` directory like so.

```
git clone https://github.com/jschuh/klipper-macros.git
```

Then paste the below section into your `printer.cfg` to get started.
The settings are all listed in [globals.cfg](globals.cfg#L5), and can be
overridden by creating a corresponding variable with a new value in your
`[gcode_macro _km_options]` section.

# Klipper Setup

```
# All customizations are documented in globals.cfg. Just copy a variable from
# there into the section below, and change the value to meet your needs.

[gcode_macro _km_options]
# These are examples of some likely customizations:
# Any sheets in the below list will be available with a configurable offset.
#variable_bed_surfaces: ['smooth_1','texture_1']
# Length (in mm) of filament to load (bowden tubes will be longer).
#variable_load_length: 90.0
# Hide the Octoprint LCD menu since I don't use it.
#variable_menu_show_octoprint: 0
# Customize the filament menus (up to 10 entries).
#variable_menu_temperature: [
#  {'name' : 'PLA',  'extruder' : 200.0, 'bed' : 60.0},
#  {'name' : 'PETG', 'extruder' : 230.0, 'bed' : 85.0},
#  {'name' : 'ABS',  'extruder' : 245.0, 'bed' : 110.0, 'chamber' : 60}]
gcode: # This line required by Klipper.

# This line includes all the standard macros.
[include klipper-macros/*.cfg]
# Uncomment to include features that require specific hardware support.
# LCD menu support for features like bed surface selection and pause next layer.
#[include klipper-macros/optional/lcd_menus.cfg]
# Optimized bed leveling
#[include klipper-macros/optional/bed_mesh.cfg]

# The sections below here are required for the macros to work.
[idle_timeout]
gcode:
  _KM_IDLE_TIMEOUT

[pause_resume]

[respond]

[save_variables]
filename: ~/klipper_config/variables.cfg

[virtual_sdcard]
path: ~/gcode_files

```

## Slicer Configuration

### Prusa Slicer

Open "Printer Settings" → "Custom G-code" for your Klipper printer and
paste the below text into the relevant sections.

**Start G-code**

```
M190 S0
M109 S0
PRINT_START EXTRUDER={first_layer_temperature[initial_tool]} BED={first_layer_bed_temperature} MESH_MIN={first_layer_print_min[0]},{first_layer_print_min[1]} MESH_MAX={first_layer_print_max[0]},{first_layer_print_max[1]} LAYERS={total_layer_count}
; Any purge, intro lines, etc. go after this...
```

**End G-code**

```
PRINT_END
```

**Before layer change G-code**

```
;BEFORE_LAYER_CHANGE
;[layer_z]
BEFORE_LAYER_CHANGE HEIGHT=[layer_z] LAYER=[layer_num]
```

**After layer change G-code**

```
;AFTER_LAYER_CHANGE
;[layer_z]
AFTER_LAYER_CHANGE
```

# Command Reference

## Customization

All features are configured by setting `variable_` values in the 
`[gcode_macro _km_options]` section. All available variables and their purpose
are listed in [globals.cfg](globals.cfg#L5).

### Bed Surface

Provides a set of macros for selecting different bed surfaces with
correspdonding Z offset adjustments to compensate for their thickness. All
available surfaces must be listed in the `variable_bed_surfaces` array.
Corresponding LCD menus will also be added to *Setup* and *Tune* if
[`lcd_menus.cfg`](#lcd-menus) is included.

Lists all available surfaces. 

#### `SET_SURFACE_ACTIVE`

Sets the provided surface active and adjust the current Z offset to match the
offset for the surface. If no `SURFACE` argument is provided the available
surfaces are listed, with active surface preceded by a `*+*`.

* `SURFACE` - Bed surface with an associated offset.

* Sets the supplied surface active (from one listed in listed in
  `variable_bed_surfaces`). Any changes to the Z offset will be applied to and
  saved for the active surface.

#### `SET_SURFACE_OFFSET`

* `OFFSET` - New Z offset for the given surface.
* `SURFACE` *(default: current surface)* - Bed surface.

* Directly sets the the Z offset of `SURFACE` to the value of `OFFSET`. If no
  argument for `SURFACE` is provided the current active surface is used.

***Bed Surface Note:** The `SET_GCODE_OFFSET` macro is overridden to update the
offset for the active surface. This makes the bed surface work with adjustments
made in clients like Mainsail and Fluidd.*

### Beep

Implements the M300 command (if a corresponding `[output_pin beeper]` section is
present). This command causes the speaker to emit an audible tone.

#### `M300`

Emits an audible tone.

* `P` *(default: `variable_beep_duration`)* - Duration of tone.
* `S` *(default: `variable_beep_frequency`)* - Frequency of tone.

### Fans

Implements scaling parameters that alter the behavior of the M106 command. Once
set, these parameters apply to any fan speed until they are cleared.

#### `SET_FAN_SCALING`

Sets scaling parameters for the extruder fan.

* `BOOST` *(default: `0`)* - Added to the fan speed.
* `SCALE` *(default: `1.0`)* - The `BOOST` value is added an then the fan
  speed is multiplied by `SCALE`.
* `MAXIMUM` *(default: `255`)* - The fan speed is clamped to no larger
  than `MAXIMUM`.
* `MINIMUM` *(default: `0`)* - The fan speed is clamped to no less
  than `MINIMUM`; if this is a non-zero value the fan can be stopped only via
  the M107 command.
* `SPEED` *(optional)* - This specifies a new speed target, otherwise any new
  adjustments will be applied to the unadjusted value of the last set fan speed.

#### `RESET_FAN_SCALING`

* Clears all existing fan scaling factors.

### Filament

#### `LOAD_FILAMENT` and `UNLOAD_FILAMENT`

Loads or unloads filament to the nozzle.

* `TARGET` *(optional)* - If specified first preheats the extruder to the
  specified temperature.
* `LENGTH` *(default: `variable_load_length`)* - The length of filament to load or
  unload.

#### Marlin Compatibility

* The `M701` and `M702` commands are implemented.

### Heaters

Adds scaling parameters that can alter the behavior of the specified heater.
Once set, these parameters apply to any temperature target on that heater until
the scalaing parameters are cleared. A zero target temperature will turn the
heater off regardless of scaling parameters.

#### `SET_HEATER_SCALING`

Sets scaling parameters for the specified heater. If run without any arguments
any currently scaled heaters and thier scaling parameters will be listed.

* `HEATER` - The name of the heater to scale.
* `BOOST` *(default: `0.0`)* - Added to a non-zero target temperature.
* `SCALE` *(default: `1.0`)* - Multiplied with the boosted target
  temperature.
* `MAXIMUM` *(default: `max_temp`)* - The target temperature is clamped
  to no larger than `MAXIMUM`. This value must be between `min_temp` and
  `min_temp`, inclusive.
* `MINIMUM` *(default: `min_temp`)* - A non-zero target temperature is
  clamped to no less than `MINIMUM`. This value must be between `min_temp` and
  `min_temp`, inclusive.
* `TARGET` *(optional)* - This specifies a new target temperature, otherwise any
  new adjustments will be applied to the unadjusted value of the last set target
  temperature.
* *Note: a zero target temperature will turn the heater off regardless of
  scaling parameters.*

#### `RESET_HEATER_SCALING`

Clears current heater scaling.

* `HEATER` *(optional)* - The name of the heater to reset.
* *Note: if no HEATER argument is specified scaling parameters will be reset for
  all heaters.*

#### `SET_HEATER_TEMPERATURE_SCALED`

The scaled version of Klipper's `SET_HEATER_TEMPERATURE`. All arguments are the
same and the function is identical, except that scaling values are applied.

#### `TEMPERATURE_WAIT_SCALED`

The scaled version of Klipper's `TEMPERATURE_WAIT`. All arguments are the
same and the function is identical, except that scaling values are applied.

#### Marlin Compatibility

* The chamber heating commands `M141` and `M191` are implemented if a
  corresponding `[heater_generic chamber]` section is defined in the config.
* The `R` temperature parameter from Marlin is implemented for the `M109` and
  `M190` commands. This parameter will cause a wait until the target temperature
  stabilizes (i.e. the normal Klipper behavior for `S`).
* The `S` parameter for the  `M109` and `M190` commands is altered to behave as
  it does in Marlin. Rather than causing a wait until the temperature
  stabilizes, the wait will complete as soon as the temperature target is
  exceeded.
* The `M109`, `M190`, `M191`, `M104`, `M140`, and `M141` are all overridden to
  implement the heater scaling described above.

***Heater Scaling Note:** Both `SET_HEATER_TEMPERATURE` and `TEMPERATURE_WAIT`
are **not** overriden and will not scale values. This means that heater scaling
adjustments made in clients like Mainsail and Fluidd will not be scaled
(because that seemed like the clearest behavior). The
[custom LCD menus]](#lcd-menus) will also replace the temperature controls with
non-scaling versions. If you use the stock menus you'll get scaled values.*

### Kinematics

#### `G28`

Extends the `G28` command to add lazy homing by not re-homing already homed axes
when the `O` argument is included. See Klipper `G28` documentation for general
information and detail on the other arguments.

* `O` - Omits axes from the homing procedure if they are already homed.

### Layer Triggers

Provides the capability to run user-specified g-code commands at arbitrary layer
changes.

#### `GCODE_AT_LAYER`

Runs abritrary, user-provided g-code commands at the user-specified layer or
height.

* `HEIGHT` - Z height (in mm) to run the command. Exactly one of `HEIGHT` or
  `LAYER` must be specified.
* `LAYER` - Layer number (zero indexed) to run the command. Exactly one of
  `HEIGHT` or `LAYER` must be specified. The special value `next` may be
  specified run the command at the next layer change.
* `COMMAND` - The command to run at layer change. Take care to properly quote
  spaces and escape any special characters.
* `BEFORE` *(default: `0`)* - Set to 1 run the command before the layer
  change (i.e. immediately following completion of the previous layer). By
  default commands run after the layer change (i.e. immediately preceding the
  next layer). In most cases this distinction here doesn't matter, but it can
  be important when dealing with toolchangers or other multi-material printing.

#### `RESET_LAYER_GCODE`

Clears all gcode triggers and associated state. Called in the PRINT_END macro.

#### Convenient Layer Change Macros

* `PAUSE_NEXT_LAYER ...`
  * Schedules the current print to pause at the next layer change. See
    [`PAUSE`](#pause) macro for additional arguments.
* `PAUSE_AT_LAYER  { HEIGHT=<pos> | LAYER=<layer> } ...`
  * Schedules the current print to pause at the specified layer change.
    See [`PAUSE`](#pause) for additional arguments.
* `FAN_AT_LAYER { HEIGHT=<pos> | LAYER=<layer> } ...`
  * Schedules a fan adjustment at the specified layer change. See 
    [`SET_FAN_SCALING`](#set_fan_scaling) for additional arguments.
* `HEATER_AT_LAYER { HEIGHT=<pos> | LAYER=<layer> } ...`
  * Schedules a heater adjustment at the specified layer change. See 
    [`SET_HEATER_SCALING`](#set_heater_scaling) for additional arguments.

***Layer Triggers Note:** If any triggers cause an exception the current print
will abort. The convenience macros above validate their arguments as much as is
possible to reduce the chances of an aborted print, but they cannot entirely
eliminate the risk of a macro doig something that aborts the print.

### Park

Implements toolhead parking.

#### `PARK`

Parks the toolhead.

* `P` *(default: `2`)* - Parking mode
  * `P=0` - If current Z-pos is lower than Z-park then the nozzle will be raised
    to reach Z-park height
  * `P=1` - No matter the current Z-pos, the nozzle will be raised/lowered to
    reach Z-park height.
  * `P=2` - The nozzle height will be raised by Z-park amount but never going
    over the machine’s Z height limit.
* `X` *(default: `variable_park_x`)* - Absolute X parking coordinate.
* `Y` *(default: `variable_park_y`)* - Absolute Y parking coordinate.
* `Z` *(default: `variable_park_z`)* - Z parking coordinate applied according
  to the `P` parameter.
* Note: If a print is in progress the larger of the tallest printed layer or the
  current Z position will be used as the current Z position, to avoid collisions
  with already printed objects during a sequential print.

#### Marlin Compatibility

* The `G27` command is implemented with a default `P0` argument.

### Pause, Resume, Cancel

#### `PAUSE`

Pauses the current print.

* `X` *(default: `variable_park_x`)* - Absolute X parking coordinate.
* `Y` *(default: `variable_park_y`)* - Absolute Y parking coordinate.
* `Z` *(default: `variable_park_z`)* - Relative Z parking coordinate
* `E` *(default: `5`)* - Retraction length to prevent ooze.
* `B` *(default: `10`)* - Number of beeps to emit (if `M300` is enabled).

#### `RESUME`

* `E` *(default: `5`)* - Retraction length to prevent ooze.

#### `CANCEL_PRINT`

Cancels the print and performs all the same functions as `PRINT_END`.

#### Marlin Compatibility

* The `M24`, `M25`, `M600`, `M601`, and `M602` commands are all implemented by
  wrapping the above commands.


### Print Start and End

#### `PRINT_START`

Sets up the printer prior to strating a print (called from the slicer's print
start g-code). A target `CHAMBER` temperature may be provided if a 
`[heater_generic chamber]` section is present in the klipper config.
If `MESH_MIN` and `MESH_MAX` are provided, then `BED_MESH_CALIBRATE` will probe
only the area within the specified rectangle, and will scale the number of
probes to the appropriate density (this can dramatically reduce probe times for smaller prints).

* `BED` - Bed heater starting temperature.
* `EXTRUDER` - Extruder heater starting temperature.
* `CHAMBER` *(optional)* - Chamber heater starting temperature.
* `MESH_MIN` *(optional)* - Minimum x,y coordinate of the first layer.
* `MESH_MAX` *(optional)* - Maximum x,y coordinate of the first layer.
* `LAYERS` *(optional)* - Total number of layers in the print.

#### `PRINT_END`

Parks the printhead, shuts down heaters, fans, etc, and performs general state
housekeeping at the end of the print (called from the slicer's print end
g-code).

### Velocity

These are some basic wrappers for Klipper's analogs to some of Marlin's velocity
related commands, such as accelleration, jerk, and linear advance.

#### Marlin Compatibility

* The `M201`, `M203`, `M204`, and `M205` commands are implemented by calling
  Klipper's `SET_VELOCITY_LIMIT` command. For calls that set the `ACCEL`
  parameter, the `ACCEL_TO_DECEL` parameter is also set and scaled by
  `variable_velocity_decel_scale` *(default: `0.5`)*.
* The `M900` command is implemented by calling Klipper's `SET_PRESSURE_ADVANCE`
  command. The `K` factor is scaled by `variable_pressure_advance_scale`
  *(default: `-1.0`)*. If the scaling value is negative the `M900` command has no
  effect.

## Optional Configs

### Bed Mesh

`BED_MESH_CALIBRATE`

Wraps the equivalent Klipper command to scale and redistribute the probe points
so that only the appropriate area in `MESH_MIN` and `MESH_MAX` is probed. This
can dramatically reduce probing times for anything that doesn't fill the first
layer of the bed.

The following additional configuration options are available from
[globals.cfg](globals.cfg#L5).

* `variable_probe_mesh_padding` - Extra padding around the rectangle defined by
  `MESH_MIN` and `MESH_MAX`.
* `variable_probe_min_count` - Minimum number of probes for partial probing of a
  bed mesh.
* `variable_probe_count_scale` - Scaling factor to increase probe count for
   partial bed probes.

***Configuration:** `[include klipper-macros/optional/bed_mesh.cfg]`*

***Requirements:** A properly configured `bed_mesh` section.*

### LCD Menus

Adds relevant menu items to an LCD display and improves some existing
functionality. See the [customization](#customization) section above for more
information on how to configure specific behaviors.

* Confirmation added for cancelling the print or disabling steppers during a
  print.
* Several temperature menu changes:
  * Up to 10 filaments and their corresponding temperatures can be set via
   `variable_menu_temperature`.
  * Per filament chamber temperature controls are available if a 
    `[heater_generic chamber]` section is configured.
  * The cooldown commands are moved to the top level temperature menu.
* The filament loading commands are replaced with macros that use the lengths
  and speeds specified in `variable_load_length` and `variable_load_speed`,
  which includes a priming phase at the end of the load (controlled via
  `variable_load_priming_length` and `variable_load_priming_speed`).
* Setup and tuning menus are present for managing [bed surfaces.](#bed-surface)
* The SD card menu has been streamlined for printing and non-printing modes.
* You can hide the Octoprint or SD card menus if you don't use them 
  (via `variable_menu_show_octoprint` and `variable_menu_show_sdcard`,
  respectively).

***Configuration:** `[include klipper-macros/optional/lcd_menus.cfg]`*

***Requirements:** A properly configured `display` section.*
