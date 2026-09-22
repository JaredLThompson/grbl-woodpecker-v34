# GRBL for Woodpecker CNC CAMXTOOL v3.4

Custom GRBL 1.1h firmware for the **Woodpecker CNC CAMXTOOL v3.4**
controller commonly found on Genmitsu/SainSmart 3018-class CNC machines.

This fork fixes the limit-switch pin mapping on this particular controller
and supports a mixed limit-switch configuration:

- X: normally closed (NC)
- Y: normally closed (NC), two switches in series
- Z: normally open (NO), two switches
- Homing enabled
- Hard limits enabled
- Soft limits enabled
- Variable spindle PWM retained on D11

The firmware in `main` is based on the official GRBL release:

**GRBL v1.1h.20190825**

Upstream project:

https://github.com/gnea/grbl

---

## Why This Fork Exists

The Woodpecker CNC CAMXTOOL v3.4 PCB does not map its physical
`X_End`, `Y_End`, and `Z_End` connectors to the stock GRBL limit pins
the way the silkscreen suggests.

Testing the physical inputs showed:

| PCB connector | AVR / Arduino pin | Stock GRBL interpretation |
|---|---:|---|
| `X_End` | D12 | Z limit |
| `Y_End` | D10 | Y limit |
| `Z_End` | D9 | X limit |

In other words, **X and Z are swapped from GRBL's perspective**.

This fork remaps the GRBL limit bits so that the physical PCB labels once
again correspond to the logical GRBL axes:

| PCB connector | AVR / Arduino pin | Custom GRBL axis |
|---|---:|---|
| `X_End` | D12 | X |
| `Y_End` | D10 | Y |
| `Z_End` | D9 | Z |
| Spindle PWM | D11 | PWM |

The relevant modification is in `grbl/cpu_map.h`:

```c
// Woodpecker CNC CAMXTOOL v3.4 limit connector mapping.
// With VARIABLE_SPINDLE:
//   X_End -> D12
//   Y_End -> D10
//   Z_End -> D9
#ifdef VARIABLE_SPINDLE
  #define X_LIMIT_BIT    4  // Uno Digital Pin 12 - X_End
  #define Y_LIMIT_BIT    2  // Uno Digital Pin 10 - Y_End
  #define Z_LIMIT_BIT    1  // Uno Digital Pin 9  - Z_End
#else
  #define X_LIMIT_BIT    1  // Uno Digital Pin 9
  #define Y_LIMIT_BIT    2  // Uno Digital Pin 10
  #define Z_LIMIT_BIT    3  // Uno Digital Pin 11
#endif
```

D11 remains available for variable-spindle PWM.

---

## Mixed NC / NO Limit Switches

This machine uses different electrical conventions for its limit switches.

### X axis

One normally-closed microswitch connected to `X_End`.

The switch opens when triggered.

### Y axis

Two normally-closed microswitches, one at each end of travel.

The Woodpecker board's two `Y_End` connectors are electrically parallel,
so the two switches are wired **in series** and connected through a single
`Y_End` connector.

Therefore:

- neither switch pressed -> circuit closed
- either switch pressed -> circuit open

This allows either Y switch to trigger the Y limit input.

### Z axis

The Z axis uses a `Z-LIMIT V2.1` PCB containing two normally-open
microswitches.

The two Woodpecker `Z_End` connectors are electrically parallel.

Either Z switch therefore closes the Z limit input to ground when triggered.

---

## Limit Polarity

Because X/Y are NC while Z is NO, a single global GRBL `$5` setting cannot
correctly represent all three axes by itself.

This fork adds a compile-time inversion for Z in `grbl/config.h`:

```c
#define INVERT_LIMIT_PIN_MASK (1<<Z_LIMIT_BIT)
```

Combined with:

```text
$5=1
```

this produces the desired behavior for all three axes.

Verified status reports:

```text
all released -> no Pn field
X pressed    -> Pn:X
Y pressed    -> Pn:Y
Z pressed    -> Pn:Z
```

All five physical switches have been individually tested.

---

## Homing Configuration

Homing is configured as:

- X -> minimum / left
- Y -> maximum / rear
- Z -> maximum / up

GRBL homes Z first, followed by X and Y.

The required homing-direction mask is:

```text
$23=1
```

Only the X homing direction is inverted.

After a successful `$H`, with a 1 mm homing pull-off, the tested machine
reports approximately:

```text
MPos:-268.000,-1.000,-1.000
```

This is expected.

The machine-coordinate convention is approximately:

```text
X: -269 ... 0
Y: -180 ... 0
Z:  -38 ... 0
```

The homing switches are located at:

```text
X = -269   left
Y =    0   rear
Z =    0   top
```

The `$27=1` homing pull-off moves the machine 1 mm away from the switches
after homing.

---

## Measured Machine Travel

Travel was measured experimentally after installing and testing the limit
switches.

### X

Approximately 270 mm exists between the X home switch and the physical
right-side mechanical endpoint.

There is currently only one physical X limit switch.

The right-side mechanical endpoint is the carriage/coupler region, so the
configured soft limit intentionally leaves margin before it:

```text
$130=269
```

After the 1 mm homing pull-off, usable travel to machine X=0 is approximately
268 mm.

### Y

Measured switch-to-switch travel:

```text
180 mm
```

Configured:

```text
$131=180
```

### Z

Measured upper-to-lower switch envelope:

```text
38 mm
```

Configured:

```text
$132=38
```

The lower Z switch triggers approximately 3 mm before the physical
mechanical stop.

---

## Known-Good GRBL Configuration

The following configuration is currently tested on the machine:

```text
$0=10
$1=25
$2=0
$3=2
$4=0
$5=1
$6=0
$10=19
$11=0.010
$12=0.002
$13=0
$20=1
$21=1
$22=1
$23=1
$24=25.000
$25=500.000
$26=250
$27=1.000
$30=11180
$31=0
$32=0
$100=800
$101=800
$102=800
$110=1000
$111=1000
$112=600
$120=30
$121=30
$122=30
$130=269.000
$131=180.000
$132=38.000
```

### Safety configuration

```text
$20=1    Soft limits enabled
$21=1    Hard limits enabled
$22=1    Homing enabled
```

Both soft-limit and hard-limit behavior have been deliberately tested.

A soft-limit violation correctly produces:

```text
ALARM:2
```

A physical hard-limit activation correctly produces:

```text
ALARM:1
```

After a hard-limit event, reset, controller reconnect, or any other event
that makes machine position uncertain, **re-home the machine with `$H` before
trusting machine coordinates.**

---

## Spindle Configuration

This machine has been upgraded to a 500 W spindle using an external
0-10 V spindle controller.

Control path:

```text
GRBL D11 PWM
    |
    v
PWM-to-0-10V converter
    |
    v
500 W spindle controller
```

The PWM converter is configured for a 5 V logic-level PWM input and was
calibrated to:

```text
50% PWM -> 5.00 V
100% PWM -> approximately 9.86 V
```

Measured maximum spindle speed is approximately:

```text
11180 RPM
```

Therefore GRBL is configured with:

```text
$30=11180
$31=0
```

Commands requesting higher spindle speeds are clamped by GRBL to `$30`.

---

## Building

The firmware has been successfully built with:

```text
AVR GCC 7.3.0
```

The source is based on:

```text
v1.1h.20190825
```

Upstream commit/tag point:

```text
40eb439
```

The Woodpecker-specific modification was initially committed as:

```text
84ff5cf Fix Woodpecker v3.4 limit mapping and mixed switch polarity
```

The resulting `grbl.hex` from that build had SHA-256:

```text
985E78F7E0AFE4E51FD95E9ED2DDC3D327B5065C0F491FB2C02417C90963A6D3
```

---

## Flashing the Woodpecker v3.4

This particular controller uses an ATmega328P and an older Nano-style
STK500v1 bootloader.

The bootloader baud rate was experimentally determined to be:

```text
57600
```

Example flash command on Windows, assuming the controller is `COM7`:

```powershell
avrdude `
  -p atmega328p `
  -c arduino `
  -P COM7 `
  -b 57600 `
  -U flash:w:.\grbl.hex:i
```

The known-good custom firmware programmed approximately:

```text
29766 bytes
```

After flashing, GRBL communicates normally at:

```text
115200 baud
```

A successful startup reports:

```text
[VER:1.1h.20190825:]
[OPT:V,15,128]
```

---

## Original Firmware Backup

Before modifying the controller, the original flash and EEPROM were backed
up with `avrdude`.

Original flash backup:

```text
woodpecker-v34-original-grbl-1.1f.hex
```

SHA-256:

```text
DDB991CD0C978BF11D506B9447FA8187D5FDA1E8CC5EF1C068CAC4D70DB88969
```

Original EEPROM backup:

```text
woodpecker-v34-original-eeprom.hex
```

SHA-256:

```text
8BF874484CC3D1FC90827A4CE286FCD0B774159E00BD6C3A154A0F0CC5AED9AC
```

**Important:** the original flash dump is a full 32768-byte readable flash
image and may include the bootloader. Restore it deliberately rather than
treating it as an ordinary application-only GRBL image.

---

## Machine Coordinates vs Work Coordinates

Homing establishes the GRBL **machine coordinate system**.

Work coordinate systems such as G54 are separate.

The normal operating workflow is:

```text
Power on / controller reset
        |
        v
       $H
        |
        v
Mount stock
        |
        v
Jog to desired work origin
        |
        v
Set G54
        |
        v
Run job
```

For the current Fusion 360 workflow, G54 is normally placed at the desired
stock origin with Z zero on the top surface.

G54 persists in EEPROM, but machine position does not survive a controller
reset in a trustworthy way.

Therefore:

> If the controller has been reset or reconnected and machine position is
> uncertain, home the machine before relying on G54 or soft limits.

---

## Tested Hardware

Development and testing were performed on:

- Genmitsu 3018-PRO CNC
- Woodpecker CNC CAMXTOOL v3.4 controller
- ATmega328P
- X NC microswitch
- Two Y NC microswitches wired in series
- Z-LIMIT V2.1 dual-switch Z assembly using NO contacts
- 500 W upgraded spindle
- External PWM-to-0-10 V converter
- Universal Gcode Sender (UGS) Platform 2.1.26

Other Woodpecker revisions may use different PCB routing.

**Verify your board electrically before assuming this pin mapping applies
to another revision.**

---

## Upstream GRBL

This project is a hardware-specific fork of GRBL.

Original GRBL project:

https://github.com/gnea/grbl

The upstream repository is retained locally as the `upstream` Git remote so
the original project history remains available.

GRBL itself is the work of the original GRBL authors and contributors.

---

## Status

The following have been physically verified on the target machine:

- [x] Custom firmware boots
- [x] X limit reports `Pn:X`
- [x] Both Y limits report `Pn:Y`
- [x] Both Z limits report `Pn:Z`
- [x] Homing succeeds
- [x] X/Y/Z homing directions verified
- [x] Machine travel measured
- [x] Soft limits tested
- [x] Hard limits tested
- [x] Spindle PWM calibrated
- [x] G54/G92 coordinate state verified
- [x] Real G-code streamed successfully

The PCB silkscreen and GRBL now agree on what **X, Y, and Z** mean.

Finally.
