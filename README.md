# xcc-fan-controller
Lightweight Zsh fan controller for Lenovo XClarity Controller (XCC) using IPMI.


Tested and working:

* Lenovo ThinkSystem SR650 V1 (7X06)
* Lenovo XClarity Controller (XCC) 9.95
* UEFI 4.13
* Linux with local IPMI support (ipmitool -I open)

Other models and firmware versions have not been tested.

# Install and use

As root

```sh
apt install ipmitool zsh
```
```sh
./fan-control
```
