# XDS110 GPIO control

The stand-alone XDS110 has four user GPIO pins, on both the CTI-20 and the AUX connector. OpenOCD
can drive and read them with the `xds110 gpio` commands, for example to set a target's boot mode
pins before a reset. TI's `dbgjtag -Y gpiopins` does the same, but needs Code Composer Studio.

| Pin   | Direction       |
| ----- | --------------- |
| GPIO0 | output or input |
| GPIO1 | output or input |
| GPIO2 | input only      |
| GPIO3 | input only      |

## Behaviour

- All pins start as inputs (hi-Z).
- The probe keeps the pin state after OpenOCD exits.
- The probe ignores levels written to an input. Configure a pin as an output first.
- A pin switched to output drives high until a level is written. A pin set low therefore goes high
  briefly, between the two USB commands. A line that must never see a high level needs a pull-down
  or another hardware fix.
- A pin switched back to input keeps reading its last driven level for a while. Don't rely on
  GPIO0 or GPIO1 readings while they are inputs.

## Commands

| Command                                    | Use before `init` | Description                                     |
| ------------------------------------------ | ----------------- | ----------------------------------------------- |
| `xds110 gpio`                              | no                | Print the direction and the levels of the pins. |
| `xds110 gpio read [<mask>]`                | no                | Return the pin levels as a bitmask.             |
| `xds110 gpio config <direction> [<mask>]`  | yes               | Set pins as outputs or inputs.                  |
| `xds110 gpio write <level> [<mask>]`       | yes               | Set the level of the output pins.               |

Each argument is a 4-bit bitmask: a number from `0x0` to `0xf`, in hex or decimal, with bit 0 for
GPIO0 up to bit 3 for GPIO3.

| Argument      | A set bit (1) means         | A clear bit (0) means |
| ------------- | --------------------------- | --------------------- |
| `<direction>` | output                      | input (hi-Z)          |
| `<level>`     | high                        | low                   |
| `<mask>`      | the command affects the pin | the pin is left alone |

For example, `xds110 gpio write 0x1 0x3` sets GPIO0 high and GPIO1 low, and leaves GPIO2 and GPIO3
alone. `xds110 gpio read` returns the levels the same way: `0xd` means GPIO1 is low and the other
pins are high.

`config` and `write` used before `init` are stored and applied as soon as OpenOCD connects to the
probe, before it checks the target supply or scans JTAG. The pins are set even if `init` then fails,
for example because the target is off.

`config` rejects GPIO2 and GPIO3 as outputs. All commands reject a mask above `0xf`.

`interface/xds110.cfg` also defines two helpers. Unlike the commands above, they take single
values, not bitmasks: `<pin>` is a pin number from 0 to 3, `<level>` is 0 (low) or 1 (high), and
`<ms>` and `<timeout_ms>` are milliseconds.

| Proc                                                | Description                                                                         |
| --------------------------------------------------- | ----------------------------------------------------------------------------------- |
| `xds110_gpio_pulse <pin> <level> <ms>`              | Drive pin 0 or 1 to `<level>` for `<ms>`, then to the opposite level.               |
| `xds110_gpio_wait <pin> <level> <timeout_ms>`       | Wait until the pin reads `<level>`, polling every 10 ms. Fail after `<timeout_ms>`. |

## Examples

Set GPIO0 high and GPIO1 low, then exit:

```sh
openocd -f interface/xds110.cfg \
    -c 'transport select jtag; adapter speed 1000; init' \
    -c 'xds110 gpio config 0x3; xds110 gpio write 0x1; shutdown'
```

Print the pin state:

```sh
openocd -f interface/xds110.cfg \
    -c 'transport select jtag; adapter speed 1000; init; echo [xds110 gpio]; shutdown'
```

```text
XDS110: GPIO direction = 0x3, levels = 0xd
```

Set boot mode pins in a board config before OpenOCD touches the target:

```tcl
source [find interface/xds110.cfg]
xds110 gpio config 0x3
xds110 gpio write 0x2
```

Change one pin and leave the other alone:

```tcl
xds110 gpio write 0x2 0x2   ;# GPIO1 high, GPIO0 unchanged
xds110 gpio write 0x0 0x1   ;# GPIO0 low, GPIO1 unchanged
```

Test an input in a script:

```tcl
if {[xds110 gpio read 0x4]} {
    echo "GPIO2 is high"
}
```

Hold a reset line, active low on GPIO0, for 100 ms, then wait up to 2 s for the target to raise
its ready signal on GPIO3:

```tcl
xds110_gpio_pulse 0 0 100
xds110_gpio_wait 3 1 2000
```

Release both outputs:

```tcl
xds110 gpio config 0x0
```
