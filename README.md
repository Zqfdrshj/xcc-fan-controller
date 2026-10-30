# xcc-fan-controller

Lightweight Zsh fan controller for Lenovo XClarity Controller (XCC), using local IPMI.

**Tested on:** Lenovo ThinkSystem SR650 V1 (7X06), XCC 9.95, UEFI 4.13 and Linux with local IPMI (`ipmitool -I open`). Other models and firmware have not been tested. Lenovo raw commands are undocumented.

## Architecture

The project follows the same layout as zpack: a central `autoload`, one function per file, folders by responsibility, and a `zMakefile` to compile, install and uninstall it.

```text
fan-control             Entry point
autoload                Zsh fpath and function registrations
zMakefile               all / clean / install / uninstall
config/default.conf     Default settings and fan commands
setup/                  Configuration loading and validation
sensors/                Sensor reading, parsing and safety checks
control/                Mode selection, IPMI commands and controller lifecycle
```

## Build and install

Requires `zsh`, `ipmitool`, GNU `timeout` and appropriate local IPMI support. To run from the checkout as root:

```sh
sudo ./fan-control
```

To compile the autoloaded functions and install in the Mouse userspace:

```sh
./zMakefile all
sudo ./zMakefile install
```

The installed program is `/usr/bin/fan-control`, with its sources and compiled function files (if built) in `/usr/share/mouse-lib/xcc-fan-controller`. `./zMakefile clean` removes local `.zwc` artifacts and `sudo ./zMakefile uninstall` removes the installation without deleting your personal configuration.

## Serman service

The included `etc/serman/services/fan-control.service` unit runs the controller as a
foreground service and restarts it if it exits unexpectedly. On systems with
`/etc/serman/services`, `./zMakefile install` also installs this unit, but does
not enable it or start the controller automatically.

For an already installed controller, install the unit directly:

```sh
sudo install -m 644 etc/serman/services/fan-control.service /etc/serman/services/fan-control.service
sermanctl reload
sermanctl fan-control enable
sermanctl fan-control start
```

To stop it: `sermanctl fan-control stop`. Serman writes controller output to
`/var/log/serman/fan-control.log`. The unit uses `Ready=spawn` because the
controller has no explicit readiness notification; this only confirms launch,
not sensor validation or successful manual fan control.

## Configuration

Defaults live in `config/default.conf`. A trusted administrator can override individual settings in `/etc/xcc-fan-controller.conf`, or pass `--config /path/to/override.conf`. Explicit `--config` replaces the optional system override, not the defaults.

```zsh
cfg[cpu_low_below]=32
cfg[cpu_low_return]=29
cfg[ambient_limit]=38
cfg[poll_interval]=3

low_command=(raw 0x3a 0x07 0xff 0x56 0x01)
medium_command=(raw 0x3a 0x07 0xff 0x8a 0x01)
auto_command=(raw 0x3a 0x07 0xff 0xff 0x00)
```

`ipmi_command=(ipmitool -I open)` configures the shared IPMI prefix. All command values are Zsh arrays; they are executed directly without `eval`. Sensor limits, expected sensor counts, sample period, timeouts and hysteresis settings are configurable. Consult `config/default.conf` for available settings.

Configuration files are executable **Zsh code** and must be writable only by trusted administrators.

Validate settings without root access or hardware commands:

```sh
./fan-control --check-config
./fan-control --config ./custom.conf --check-config
```

## Safety

The maximum CPU temperature selects LOW (< 30 °C) or MEDIUM (30–59 °C) by default. A missing/invalid required sensor, triggered overtemperature alarm or exceeded safety threshold makes the controller request XClarity AUTO. A hysteresis margin delays resumption of manual fan commands.

The controller explicitly restores XClarity AUTO when interrupted by Ctrl+C (SIGINT), SIGTERM or SIGHUP, and attempts recovery on a normal function return. Cleanup is guarded against duplicate calls. This is best effort; software cannot guarantee recovery from SIGKILL, power loss, kernel crashes or BMC failure. Do not use unverified raw commands on other servers.
