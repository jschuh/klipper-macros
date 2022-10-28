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
* **[Optimized mesh bed leveling](#bed-mesh-improvements)** - Probes only within
  the printed area, which can save a lot of time on smaller prints.

## A few warnings...

* You must have a `heater_bed`, `extruder`, and other [sections listed
  below](#klipper-setup) configured, otherwise the macros will ***force a
  printer shutdown at startup***. Unfortunately, the Klipper macro
  doesn't have a more graceful way of handling this sort of thing.
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

> **Note:** If you have a `[homing_override]` section you will need to update any
> `G28` commands in that section to use to `G28.6245197` instead (which is the
> renamed version of Klipper's built-in `G28`). Failure to do this will cause
> `G28` commands to error out with the message *"Macro G28 called recursively"*.

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
gcode: # This line is required by Klipper.
# Any code you put here will run at klipper startup, after the initialization
# for these macros. For example, you could uncomment the following line to
# automatically adjust your bed surface offsets to account for any changes made
# to your Z endstop or probe offset.
#  ADJUST_SURFACE_OFFSETS

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

[display_status]

# Uncomment the sections below if Fluidd complains (because it's confused).
#
#[gcode_macro CANCEL_PRINT]
#rename_existing: CANCEL_PRINT_BASE
#gcode: CANCEL_PRINT_BASE{% for k in params %}{' '~k~'='~params[k]}{% endfor %}
```

## Moonraker Configuration

Paste the following into your `moonraker.conf` if you want the macros to
automatically update directly from this repo.

```
[update_manager klipper-macros]
type: git_repo
origin: https://github.com/jschuh/klipper-macros.git
path: ~/klipper_config/klipper-macros
primary_branch: main
is_system_service: False
managed_services: klipper
```

## Slicer Configuration

### PrusaSlicer / SuperSlicer

PrusaSlicer and its variants are fairly easy to configure. Just open **Printer
Settings → Custom G-code** for your Klipper printer and paste the below text
into the relevant sections.

#### Start G-code

```
M190 S0
M109 S0
PRINT_START EXTRUDER={first_layer_temperature[initial_tool]} BED=[first_layer_bed_temperature] MESH_MIN={first_layer_print_min[0]},{first_layer_print_min[1]} MESH_MAX={first_layer_print_max[0]},{first_layer_print_max[1]} LAYERS={total_layer_count}
; Any purge, intro lines, etc. go after this...
```

#### End G-code

```
PRINT_END
```

#### Before layer change G-code

```
;BEFORE_LAYER_CHANGE
;[layer_z]
BEFORE_LAYER_CHANGE HEIGHT=[layer_z] LAYER=[layer_num]
```

#### After layer change G-code

```
;AFTER_LAYER_CHANGE
;[layer_z]
AFTER_LAYER_CHANGE
```

### Ultimaker Cura

Cura is a bit more difficult to configure, and it comes with the following known
issues:

- Cura doesn't have proper placeholders for before and after layer changes, so
  the before triggers all fire and are followed immediately by the after
  triggers, all of which happens inside the layer change. This probably doesn't
  matter, but it does mean that you can't use the before and after triggers to
  avoid running code in the layer change.
- Cura doesn't provide the Z-height of the current layer, so it's inferred from
  the current nozzle position, which will include the Z-hop if the nozzle is
  currently raised. This means height based gcode triggers may fire earlier than
  expected.
- Cura's **Insert at layer change** fires the `After` trigger and then the
  `Before` trigger (i.e before or after the *layer*, versus before or after the
  *layer change*). These macros and PrusaSlicer do the opposite, which is
  something to keep in mind if you're used to how Cura does it. Note that these
  macros do use an  **Insert at layer change** script to force `LAYER` comment
  generation, but that doesn't affect the trigger ordering.
- Cura does not provide the first layer bounding rectangle, only the model
  bounding volume. This means the XY bounding box used to speed up mesh probing
  may be larger than it needs to be, resulting in bed probing that's not as fast
  as it could be. 

Accepting the caveats, the macros work quite well with Cura if you follow the
configuration steps listed below.

#### Start G-code

```
M190 S0
M109 S0
PRINT_START EXTRUDER={material_print_temperature_layer_0} BED={material_bed_temperature_layer_0}
; Any purge, intro lines, etc. go after this...
```

#### End G-code

```
PRINT_END
```

#### Post Processing Plugin

Use the menu item for **Extensions → Post Processing → Modify G-Code** to
open the **Post Processing Plugin** and add the following four scripts. *The
scripts must be run in the order listed below and be sure to copy the strings
exactly, with no leading or trailing spaces.*

##### Search and Replace

- Search: `(\n;(MIN|MAX)X:([^\n]+)\n;\2Y:([^\n]+))`
- Replace: `\1\nPRINT_START_SET MESH_\2=\3,\4`
- Use Regular Expressions: ☑️

##### Search and Replace

- Search: `(\n;LAYER_COUNT:([^\n]+))`
- Replace: `\1\nINIT_LAYER_GCODE LAYERS=\2\nPRINT_START_SET LAYERS=\2`
- Use Regular Expressions: ☑️

##### Insert at layer change

- When to insert: `Before`
- G-code to insert: `;BEFORE_LAYER_CHANGE`

##### Search and Replace

- Search: `(\n;LAYER:([^\n]+))`
- Replace: `\1\nBEFORE_LAYER_CHANGE LAYER=\2\nAFTER_LAYER_CHANGE`
- Use Regular Expressions: ☑️

# Command Reference

## Customization

All features are configured by setting `variable_` values in the 
`[gcode_macro _km_options]` section. All available variables and their purpose
are listed in [globals.cfg](globals.cfg#L5).

### Bed Mesh Improvements

`BED_MESH_CALIBRATE_FAST`

Wraps the Klipper `BED_MESH_CALIBRATE` command to scale and redistribute the
probe points so that only the appropriate area in `MESH_MIN` and `MESH_MAX` is probed. This can dramatically reduce probing times for anything that doesn't
fill the first layer of the bed. `PRINT_START` will automatically use this for
bed mesh calibration if a `[bed_mesh]` section is detected in your config.

The following additional configuration options are available from
[globals.cfg](globals.cfg#L5).

* `variable_probe_mesh_padding` - Extra padding around the rectangle defined by
  `MESH_MIN` and `MESH_MAX`.
* `variable_probe_min_count` - Minimum number of probes for partial probing of a
  bed mesh.
* `variable_probe_count_scale` - Scaling factor to increase probe count for
   partial bed probes.

> **Note:** See the [optional section](#bed-mesh) for additional macros.

> **Note:** The bed mesh optimizations are silently disabled for delta printers
  (because jinja2 lacks the necessary math support) and when the mesh parameters
  include a [`RELATIVE_REFERENCE_INDEX`](https://www.klipper3d.org/Bed_Mesh.html#the-relative-reference-index)
  (which is cinompatible with dynamic mesh generation).

`BED_MESH_CHECK`

Checks the `[bed_mesh]` config and warns if `mesh_min` or `mesh_max` could allow
a move out of range during `BED_MESH_CALIBRATE`. This is run implictily at
Klipper startup and at the start of `BED_MESH_CALIBRATE`.

### Bed Surface

Provides a set of macros for selecting different bed surfaces with
correspdonding Z offset adjustments to compensate for their thickness. All
available surfaces must be listed in the `variable_bed_surfaces` array.
Corresponding LCD menus for sheet selection and babystepping will be added to
*Setup* and *Tune* if [`lcd_menus.cfg`](#lcd-menus) is included. Any Z offset
adjustments made in the LCD menus, console, or other clients (e.g. Mainsail,
Fluidd) will be applied to the current sheet and persisted across restarts.

Lists all available surfaces. 

#### `SET_SURFACE_ACTIVE`

Sets the provided surface active (from one listed in listed in
`variable_bed_surfaces`) and adjusts the current Z offset to match the
offset for the surface. If no `SURFACE` argument is provided the available
surfaces are listed, with active surface preceded by a `*`.

* `SURFACE` - Bed surface with an associated offset.

#### `SET_SURFACE_OFFSET`

Directly sets the the Z offset of `SURFACE` to the value of `OFFSET`. If no
argument for `SURFACE` is provided the current active surface is used. If no
argument for `OFFSET` is provided the current offset is displayed.

* `OFFSET` - New Z offset for the given surface.
* `SURFACE` *(default: current surface)* - Bed surface.

> **Note:** The `SET_GCODE_OFFSET` macro is overridden to update the
> offset for the active surface. This makes the bed surface work with Z offset
> adjustments made via any interface or client.

#### `ADJUST_SURFACE_OFFSETS`

Adjusts surface offsets to account for changes in the Z endstop position or
probe Z offset. A message to invoke this command will be shown in the console
when a relevant change is made to `printer.cfg`.

* `IGNORE` - Set to 1 to reset the saved offsets without adjusting the surfaces.

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

#### `LOAD_FILAMENT` / `UNLOAD_FILAMENT`

Loads or unloads filament to the nozzle.

* `LENGTH` *(default: `variable_load_length`)* - The length of filament to load
  or unload.
* `SPEED` *(default: `variable_load_speed`)* - Speed (in mm/m) to feed the
  filament.
* `MINIMUM` *(default: `min_extrude_temp` + 5)* - Ensures the extruder is heated
   to at least the specified temperature.

#### Marlin Compatibility

* The `M701` and `M702` commands are implemented with a default filament length
  of `variable_load_length`.

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

> **Note:** a zero target temperature will turn the heater off regardless of
> scaling parameters.

#### `RESET_HEATER_SCALING`

Clears current heater scaling.

* `HEATER` *(optional)* - The name of the heater to reset.

> **Note:** if no HEATER argument is specified scaling parameters will be reset
> for all heaters.

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

> **Note:** Both `SET_HEATER_TEMPERATURE` and `TEMPERATURE_WAIT` are **not**
> overriden and will not scale values. This means that heater scaling
> adjustments made in clients like Mainsail and Fluidd will not be scaled
> (because that seemed like the clearest behavior). The
> [custom LCD menus](#lcd-menus) will also replace the temperature controls
> with non-scaling versions. If you use the stock menus you'll get scaled
> values.

### Kinematics

#### `G28`

Extends the `G28` command to add lazy homing by not re-homing already homed axes
when the `O` argument is included (equivalent to the same argument in Marlin).
See Klipper `G28` documentation for general information and detail on the other
arguments.

* `O` - Omits axes from the homing procedure if they are already homed.

### Layer Triggers

Provides the capability to run user-specified g-code commands at arbitrary layer
changes.

#### `GCODE_AT_LAYER`

Runs abritrary, user-provided g-code commands at the user-specified layer or
height. If no arguments are specified it will display all currently scheduled
g-code commands along with their associated layer or height.

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

#### `CANCEL_ALL_LAYER_GCODE`

Cancels all g-code commands previously scheduled at any layer or height.

#### Convenient Layer Change Macros

* `PAUSE_NEXT_LAYER ...`
  * Schedules the current print to pause at the next layer change. See
    [`PAUSE`](#pause) macro for additional arguments.
* `PAUSE_AT_LAYER  { HEIGHT=<pos> | LAYER=<layer> } ...`
  * Schedules the current print to pause at the specified layer change.
    See [`PAUSE`](#pause) for additional arguments.
* `SPEED_AT_LAYER { HEIGHT=<pos> | LAYER=<layer> } SPEED=<percentage>`
  * Schedules a feedrate adjustment at the specified layer change. (`SPEED`
    parameter behaves the same as the `M220` `S` parameter.)
* `FLOW_AT_LAYER { HEIGHT=<pos> | LAYER=<layer> } FLOW=<percentage>`
  * Schedules a flow-rate adjustment at the specified layer change. (`FLOW`
    parameter behaves the same as the `M221` `S` parameter.)
* `FAN_AT_LAYER { HEIGHT=<pos> | LAYER=<layer> } ...`
  * Schedules a fan adjustment at the specified layer change. See 
    [`SET_FAN_SCALING`](#set_fan_scaling) for additional arguments.
* `HEATER_AT_LAYER { HEIGHT=<pos> | LAYER=<layer> } ...`
  * Schedules a heater adjustment at the specified layer change. See 
    [`SET_HEATER_SCALING`](#set_heater_scaling) for additional arguments.

> **Note:** If any triggers cause an exception the current print will
> abort. The convenience macros above validate their arguments as much as is
> possible to reduce the chances of an aborted print, but they cannot entirely
> eliminate the risk of a macro doig something that aborts the print.

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
* `LAZY` *(default: 1)* - Will home any unhomed axes if needed and will not
  move any axis if already homed and parked (even if `P=2`).

> **Note:** If a print is in progress the larger of the tallest printed layer or
> the current Z position will be used as the current Z position, to avoid
> collisions with already printed objects during a sequential print.

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

`BED_MESH_CALIBRATE` and `G20`

Overrides the default `BED_MESH_CALIBRATE` to use `BED_MESH_CALIBRATE_FAST`
instead, and adds the `G20` command.

***Configuration:***

```
[include klipper-macros/optional/bed_mesh.cfg]
```

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
* [Bed surface](#bed-surface) management is integrated into the setup and tuning
  menus.
* The SD card menu has been streamlined for printing and non-printing modes.
* The setup menu includes host shutdown, host restart, speed, and flow controls.
* You can hide the Octoprint or SD card menus if you don't use them 
  (via `variable_menu_show_octoprint` and `variable_menu_show_sdcard`,
  respectively).

***Configuration:***

```
[include klipper-macros/optional/lcd_menus.cfg]
```

***Requirements:** A properly configured `display` section.*
