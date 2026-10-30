# xcc-fan-controller

Lightweight Zsh fan controller for Lenovo XClarity Controller (XCC) using local IPMI.

**Tested and working on:**
- Lenovo ThinkSystem SR650 V1 (7X06)
- Lenovo XClarity Controller (XCC) 9.95
- UEFI 4.13
- Linux with local IPMI support (`ipmitool -I open`)

Other models, firmware versions and raw fan commands have **not** been tested.

## Run

Requires Zsh, ipmitool, GNU coreutils (`timeout`) and root access for hardware control.

```sh
sudo ./fan-control
```

The script selects **low** for CPU temperatures below 30 °C, **medium** at 30–59 °C, and hands control back to **XClarity AUTO** when a safety limit is reached or a required sensor cannot be validated. Hysteresis prevents repeated switching around thresholds.

## Configuration

Defaults: [config/default.conf](config/default.conf). To override just the desired settings, place a trusted Zsh file at `/etc/xcc-fan-controller.conf`, or supply `--config /path/to/file`. An explicit path replaces the optional system override; it does not replace the defaults.

Example custom configuration:

```zsh
cfg[cpu_low_below]=32
cfg[cpu_low_return]=29
cfg[ambient_limit]=38
cfg[poll_interval]=3

low_command=(raw 0x3a 0x07 0xff 0x56 0x01)
medium_command=(raw 0x3a 0x07 0xff 0x8a 0x01)
auto_command=(raw 0x3a 0x07 0xff 0xff 0x00)
```

`ipmi_command=(ipmitool -I open)` is also configurable. The `*_command` arrays contain arguments appended to `ipmi_command`; they are never passed through `eval`. Safety limits and expected sensor counts (CPU1, CPU2, DIMM, PCH, ambient, exhaust and overtemperature alarms) can all be adjusted through `cfg`. See `config/default.conf` for all supported keys.

Configuration files are **executable Zsh code**: keep them root-owned and writable only by trusted administrators. Different XClarity models may require different raw commands; do not copy unverified fan commands to other hardware.

To validate configuration without running IPMI or requiring root:

```sh
./fan-control --check-config
./fan-control --config ./my-fans.conf --check-config
```

## Safety

The highest CPU temperature sets the normal fan curve. All monitored components have independent temperature limits. Any invalid/missing required sensor, overtemperature alarm, or sensor read failure causes a request to return to XClarity's automatic control.

The program also attempts to restore AUTO after a failed fan command and on normal exit or handled termination signals. This is **best effort**, not a guarantee after SIGKILL, power loss, kernel failure or BMC failure. Never depend on software alone to guarantee cooling.

## Development

The entry point is `fan-control`; each function is a separate autoloaded file in `functions/`. The controller owns state, the sensor functions read and validate values, `select_mode` decides the fan mode and `set_mode` sends its configured IPMI command.
