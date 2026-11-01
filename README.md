LED Controller for UGREEN's DX/DXP NAS Series
==

UGREEN's DX/DXP NAS Series covers 2 to 8 bay NAS devices with a built-in system based on OpenWRT called `UGOS` or Debian called `UGOS-Pro`.  
Debian Linux or dedicated NAS operating systems and appliances are compatible with the hardware, but do not have drivers for the LED lights on the front panel to indicate power, network and hard drive activity.  
Instead, when using a 3rd party OS with e.g. DX 4600 Pro, only the power indicator light blinks, and the other LEDs are off by default.  
For the DXP series, all LEDs blink in rolling sequence when non-UGOS systems are running.

This repository
 - Describes the control logic of UGOS for the LED lights on the device front
 - Provides a command-line tool and a kernel module to control them  

For the process of understanding this control logic, please refer to [my blog (in Chinese)](https://blog.miskcoo.com/2024/05/ugreen-dx4600-pro-led-controller).

> [!NOTE]
> Only tested and working on the following devices:
> - [x] UGREEN DX4600 Pro
> - [x] UGREEN DX4700+
> - [x] UGREEN DXP2800 ([#19](https://github.com/miskcoo/ugreen_leds_controller/issues/19))
> - [x] UGREEN DXP2800 V1.2 (**Experimental**, available on [v0.4-beta](https://github.com/miskcoo/ugreen_leds_controller/releases/tag/v0.4-beta), [#128](https://github.com/miskcoo/ugreen_leds_controller/issues/128))
> - [ ] UGREEN DXP2800 GT (**WIP**, [#122](https://github.com/miskcoo/ugreen_leds_controller/issues/122), [#123](https://github.com/miskcoo/ugreen_leds_controller/issues/123))
> - [x] UGREEN DXP4800 ([#41](https://github.com/miskcoo/ugreen_leds_controller/issues/41))
> - [x] UGREEN DXP4800 Plus ([Gist](https://gist.github.com/Kerryliu/c380bb6b3b69be5671105fc23e19b7e8))
> - [x] UGREEN DXP4800 Pro ([#121](https://github.com/miskcoo/ugreen_leds_controller/issues/121))
> - [ ] UGREEN DXP4800 GT (**Experimental**, available on [v0.4-beta](https://github.com/miskcoo/ugreen_leds_controller/releases/tag/v0.4-beta), [#100](https://github.com/miskcoo/ugreen_leds_controller/pull/100))
> - [x] UGREEN DXP6800 Pro ([#7](https://github.com/miskcoo/ugreen_leds_controller/issues/7))
> - [x] UGREEN DXP8800 Plus ([#1](https://github.com/miskcoo/ugreen_leds_controller/issues/1), [Repo](https://github.com/meyergru/ugreen_dxp8800_leds_controller))
> - [ ] UGREEN DXP480T Plus ([#6](https://github.com/miskcoo/ugreen_leds_controller/issues/6#issuecomment-2156807225))
> - [ ] UGREEN iDX6011 (Pro) (**Experimental**, available on [v0.4-beta](https://github.com/miskcoo/ugreen_leds_controller/releases/tag/v0.4-beta), [#93](https://github.com/miskcoo/ugreen_leds_controller/issues/93), [#104](https://github.com/miskcoo/ugreen_leds_controller/pull/104))
>
>**I am not sure whether this module is compatible with other devices.  
>If you've tested it on other hardware, please feel free to update the list above!**
>
> Please follow the [Preparation](#Preparation) section to check if the protocol is compatible, and run `./ugreen_leds_cli all` to see which LEDs are supported by this tool.

For third-party systems, I am using Debian 12 "Bookworm", but you can find some manuals for other systems:
- **Debian**: see [the section below](#start-at-boot-for-debian-12-and-13)
- **DSM**: see [#8](https://github.com/miskcoo/ugreen_leds_controller/issues/8)
- **Proxmox**: you need to use the CLI tool in Proxmox, not in a VM
- **TrueNAS**:
  - see [#13](https://github.com/miskcoo/ugreen_leds_controller/issues/13) and [this repo](https://github.com/0x556c79/install-ugreen-leds-controller) (and maybe [here](https://github.com/miskcoo/ugreen_leds_controller/tree/truenas-build/build-scripts/truenas)) for how to build the module
  - look [here](https://gist.github.com/Kerryliu/c380bb6b3b69be5671105fc23e19b7e8) for a script using the CLI tool
  - check [here](https://github.com/miskcoo/ugreen_leds_controller/tree/gh-actions/build-scripts/truenas/build) for pre-build drivers. The default directories are built from `master`; tag builds, such as `v0.3` and `v0.4-beta`, are under `tags/<tag>/`.
- **unRAID**: there is a [plugin](https://forums.unraid.net/topic/168423-ugreen-nas-led-control/) ([related repo](https://github.com/ich777/unraid-ugreenleds-driver/tree/master/source/usr/bin))

Below is an example:  
![](https://blog.miskcoo.com/assets/images/dx4600-pro-leds.gif)

It can be achieved by the following commands:
```bash
ugreen_leds_cli all -off -status
ugreen_leds_cli power  -color 255 0 255 -blink 400 600
sleep 0.1
ugreen_leds_cli netdev -color 255 0 0   -blink 400 600
sleep 0.1
ugreen_leds_cli disk1  -color 255 255 0 -blink 400 600
sleep 0.1
ugreen_leds_cli disk2  -color 0 255 0   -blink 400 600
sleep 0.1
ugreen_leds_cli disk3  -color 0 255 255 -blink 400 600
sleep 0.1
ugreen_leds_cli disk4  -color 0 0 255   -blink 400 600
```

## Preparation

We communicate with the control chip of the LED via I2C, corresponding to the device with address `0x3a` on *SMBus I801 adapter*.  
Before proceeding, we need to load the `i2c-dev` module and install the `i2c-tools` tool.

```
$ apt install -y i2c-tools
$ modprobe -v i2c-dev
```

List the I2C adapters and identify the *SMBus I801 adapter*. The adapter number is system-dependent: on a tested DXP4800 Pro it was `i2c-0`, while the example below uses `i2c-1`. Avoid blindly scanning or registering a device on an assumed bus. The probe script below checks the controller at `0x3a` and registers it only if necessary.

```
$ i2cdetect -l
i2c-0   i2c             Synopsys DesignWare I2C adapter         I2C adapter
i2c-1   smbus           SMBus I801 adapter at efa0              SMBus adapter
i2c-2   i2c             Synopsys DesignWare I2C adapter         I2C adapter

```

## Build & Usage

> [!IMPORTANT]  
> The command-line tool and the kernel module do conflict.  
> To use the command-line tool, you must unload the `led_ugreen` module.

### The Command-line Tool

Use `cd cli && make` to build the command-line tool, and `ugreen_leds_cli` to modify the LED states (requires root permissions).

```
Usage: ugreen_leds_cli  [LED-NAME...] [-on] [-off] [-(blink|breath) T_ON T_OFF]
                    [-color R G B] [-brightness BRIGHTNESS] [-status]

       LED_NAME:    separated by white space, possible values are
                    { power, netdev, disk[1-8], all }.
       -on / -off:  turn on / off corresponding LEDs.
       -blink / -breath:  set LED to the blink / breath mode. This
                    mode keeps the LED on for T_ON millseconds and then
                    keeps it off for T_OFF millseconds.
                    T_ON and T_OFF should belong to [0, 65535].
       -color:      set the color of corresponding LEDs.
                    R, G and B should belong to [0, 255].
       -brightness: set the brightness of corresponding LEDs.
                    BRIGHTNESS should belong to [0, 255].
       -status:     display the status of corresponding LEDs.
```

Below is an example:

```bash
# turn on all LEDs
ugreen_leds_cli all -on

# query LEDs' status
ugreen_leds_cli all -status

# turn on the power indicator,
# and then set its color to blue,
# and then set its brightness to 128 / 256,
# and finally display its status
ugreen_leds_cli power -on -color 0 0 255 -brightness 128 -status
```

### The Kernel Module

For a numbered Debian 13 installation of the kernel module, LED initialization, and disk monitor, see [Debian 13 installation](debian13-installation.md). The documented DXP4800 Pro hardware and reboot results are in [the B5 test report](b5-reboot-test.md).

On Debian 13, install build tools, DKMS, I2C tools, and headers matching the running kernel before building the module:

```bash
apt install build-essential dkms i2c-tools "linux-headers-$(uname -r)"
```

Run installation commands as root. Use the matching headers for your kernel; custom kernels may require headers from their vendor.

There are three methods to install the module:

- **A)** Run `cd kmod && make` to build the kernel module, and then load it with `sudo insmod led-ugreen.ko`.

- **B)** Alternatively, you can install it with DKMS. For a fresh manual installation from the repository root:

  ```bash
  cp -r kmod /usr/src/led-ugreen-0.3
  dkms add -m led-ugreen -v 0.3
  dkms build -m led-ugreen -v 0.3
  dkms install -m led-ugreen -v 0.3
  ```

- **C)** You can also directly install the `.deb` package [here](https://github.com/miskcoo/ugreen_leds_controller/releases).

For a DKMS installation, check `dkms status led-ugreen` and `modinfo -k "$(uname -r)" led-ugreen`. DKMS is configured to rebuild the module for kernel updates, but the new kernel's matching headers and a successful build are required; verify DKMS status again after an update. With Secure Boot enabled, the module must be signed by a trusted key; enrolling a DKMS signing key may be necessary before it can load.

Loading the module alone does not register the LED controller. Run `scripts/ugreen-probe-leds` as root to locate the I801 adapter, register the controller at `0x3a` if needed, and verify its binding and disk LEDs. The adapter number printed by the script can be used for a read-only check, for example (replace `0` with the detected bus number):

```bash
readlink -f /sys/bus/i2c/devices/0-003a/driver
ls -l /sys/class/leds/disk*
```

The driver link should end in `led-ugreen`. The number of disk LEDs depends on the model; the tested DXP4800 Pro has four. If the adapter is missing or ambiguous, the device belongs to another driver, or LEDs are missing, resolve the problem before starting monitoring. Do not manually re-register an existing device.

Below is an example of setting color, brightness, and blink of the `power` LED:

```bash
echo 255 > /sys/class/leds/power/brightness    # non-zero brightness turns it on
echo "255 0 0" > /sys/class/leds/power/color   # set the color to RGB(255, 0, 0)
echo "blink 100 100" > /sys/class/leds/power/blink_type  # blink at 10Hz
```

To blink the `netdev` LED when an NIC is active, you can use the `ledtrig-netdev` module (see `scripts/ugreen-netdevmon`):

```bash
led="netdev"
modprobe ledtrig-netdev
echo netdev > /sys/class/leds/$led/trigger
echo enp2s0 > /sys/class/leds/$led/device_name
echo 1 > /sys/class/leds/$led/link
echo 1 > /sys/class/leds/$led/tx
echo 1 > /sys/class/leds/$led/rx
echo 100 > /sys/class/leds/$led/interval
```

To blink the `disk` LED when a block device is active, you can use the `ledtrig-oneshot` module and monitor the changes of`/sys/block/sda/stat` (see `scripts/ugreen-diskiomon` for an example). If you are using zfs, you can combine this script with that provided in [#1](https://github.com/miskcoo/ugreen_leds_controller/issues/1) to change the LED's color when a disk drive failure occurs.  
To see how to map the disk LEDs to correct disk slots, please read the [Disk Mapping](#disk-mapping) section.

#### Start at Boot (for Debian 12 and 13)

For the complete Debian 13 disk-monitor-only sequence, including DKMS, I2C initialization and reboot checks, use [debian13-installation.md](debian13-installation.md). The following general example additionally configures network and power LED services.

The configure file of `ugreen-diskiomon` and `ugreen-netdevmon` is `/etc/ugreen-leds.conf`.  
Please see `scripts/ugreen-leds.conf` for an example.

- Add the following lines to `/etc/modules-load.d/ugreen-led.conf`
  ```
  cat > /etc/modules-load.d/ugreen-led.conf << EOF
  i2c-dev
  led-ugreen
  ledtrig-oneshot
  ledtrig-netdev
  EOF
  ```

- Install the monitoring dependencies: `apt install smartmontools python3 util-linux`. The disk monitor requires Python 3.9 or newer, `flock`, `timeout`, and Linux pidfd support (Linux 5.3 or newer).

- Install the kernel module by one of the three methods mentioned above.  
  For example, directly install [the deb package](https://github.com/miskcoo/ugreen_leds_controller/releases).

- A `led-ugreen-utils` Debian package built from this checkout installs the disk monitor, helper, and udev rule. Previously released packages may not include the hotplug changes. If installing the utilities manually, run the following commands as root from the repository root and stop if any command fails:
  ```bash
  # install the helper first; the disk monitor also uses it for cleanup
  install -m 0755 scripts/ugreen-diskiomon-notify /usr/bin/ugreen-diskiomon-notify || exit 1

  # copy the scripts
  scripts=(ugreen-{detect-{disks,network},diskiomon,netdevmon{,-multi},power-led,probe-leds})
  for f in ${scripts[@]}; do
      chmod +x "scripts/$f"
      cp "scripts/$f" /usr/bin
  done
  
  # copy the configuration file, you can change it if needed
  cp scripts/ugreen-leds.conf /etc/ugreen-leds.conf
  
  # copy the systemd services
  cp scripts/systemd/*.service /etc/systemd/system/

  # install and reload the SATA hotplug rule before starting the monitor
  install -D -m 0644 scripts/99-ugreen-diskiomon.rules /usr/lib/udev/rules.d/99-ugreen-diskiomon.rules || exit 1
  udevadm control --reload || exit 1
  
  systemctl daemon-reload

  # initialize and verify the controller before starting LED monitors
  systemctl start ugreen-probe-leds.service || exit 1
  systemctl status ugreen-probe-leds.service --no-pager
  
  # change enp2s0 to the network device you want to monitor
  systemctl start ugreen-netdevmon@enp2s0
  test -x /usr/bin/ugreen-diskiomon-notify || { echo "Disk monitor helper is missing" >&2; exit 1; }
  systemctl start ugreen-diskiomon
  systemctl start ugreen-power-led
  
  # if you confirm that everything works well, 
  # run the command below to make the service start at boot
  systemctl enable ugreen-netdevmon@enp2s0
  systemctl enable ugreen-diskiomon
  systemctl enable ugreen-power-led
  ```

  `ugreen-diskiomon.service` requires and starts after `ugreen-probe-leds.service`. If `/run/ugreen-diskiomon.cleanup-failed` exists, its start condition prevents automatic restarts; inspect the journal and remaining monitor processes before resolving the failure. The marker is in `/run` and does not persist across a reboot.

- (_Optional_) To reduce the CPU usage of blinking LEDs when disks are active, you can enter the `scripts` directory and do the following things:
  ```bash
  # compile the disk activities monitor
  g++ -std=c++17 -O2 blink-disk.cpp -o ugreen-blink-disk

  # copy the binary file (the path can be changed, see BLINK_MON_PATH in ugreen-leds.conf)
  cp ugreen-blink-disk /usr/bin
  ```

- (_Optional_) Similarly, to reduce the latency of the standby check, you can enter the `scripts` directory and do the following things:
  ```bash
  # compile the disk standby checker
  g++ -std=c++17 -O2 check-standby.cpp -o ugreen-check-standby

  # copy the binary file (the path can be changed, see STANDBY_MON_PATH in ugreen-leds.conf)
  cp ugreen-check-standby /usr/bin
  ```

## Disk Mapping

To make the disk LEDs useful, we should map the disk LEDs to correct disk slots. First of all, we should highlight that using `/dev/sdX` is never a smart idea, as it may change at every boot.  
In the script `ugreen-diskiomon` we provide three mapping methods: **by ATA**, **by HCTL** and **by serial**. 

The best mapping method is using serial numbers, but it needs to record them manually and fill the `DISK_SERIAL` array in `/etc/ugreen-leds.conf`. We use ATA mapping by default, and find that UGOS also uses a similar mapping method (see [#15](https://github.com/miskcoo/ugreen_leds_controller/pull/15)).  
See the comments in `scripts/ugreen-leds.conf` for more details.

The HCTL mapping depends on how the SATA controllers are connected to the PCIe bus and the disk slots. To check the HCTL order, you can run the following command, and check the serial of your disks:

```bash
# lsblk -S -x hctl -o name,hctl,serial
NAME HCTL       SERIAL
sda  0:0:0:0    XXKEREXX
sdc  1:0:0:0    XXKG2BXX
sdb  2:0:0:0    XXGMU6XX
sdd  3:0:0:0    XXKJEZXX
sde  4:0:0:0    XXKJHBXX
sdf  5:0:0:0    XXGT2ZXX
sdg  6:0:0:0    XXKH3SXX
sdh  7:0:0:0    XXJDB1XX
```
> [!NOTE]  
> As far as we know, the mapping between HCTL and the disk serial are stable at each boot (see [#4](https://github.com/miskcoo/ugreen_leds_controller/pull/4) and [#9](https://github.com/miskcoo/ugreen_leds_controller/issues/9)).  
> However, it has been reported that the exact order is model-dependent (see [#9](https://github.com/miskcoo/ugreen_leds_controller/issues/9)).  
> - For DX4600 Pro, DXP4800, and DXP8800 Plus, the mapping is `X:0:0:0 -> diskX`.  
> - For DXP6800 Pro, `0:0:0:0` and  `1:0:0:0` are mapped to `disk5` and `disk6`, and `2:0:0:0` to `6:0:0:0` are mapped to `disk1` to `disk4`.
>
> The script will use `dmidecode` to detect the device model, but I suggest to check the mapping outputed by the script manually.

## Communication Protocols

The IDs for the LED lights on the front panel of the NAS chassis are as follows: 

|     ID      | LED |
|-------------|--------------------------------|
| `0`         | Power indicator |
| `1`         | Network device indicator |
| `2`,`3`,... | Hard drive indicator "disk1", "disk2" etc. |

### Query Status

Reading 11 bytes from the address `0x81 + LED_ID` allows us to obtain the current status of the corresponding LED. The meaning of these 11 bytes is as follows:

| Address | Meaning of Corresponding Data |
|---------|--------------------------------|
| `0x00`  | LED status: 0 (off), 1 (on), 2 (blink), 3 (breath) |
| `0x01`  | LED brightness |
| `0x02`  | LED color (Red component in RGB) |
| `0x03`  | LED color (Green component in RGB) |
| `0x04`  | LED color (Blue component in RGB) |
| `0x05`  | Milliseconds needed to complete one blink/breath cycle (high 8 bits) |
| `0x06`  | Milliseconds needed to complete one blink/breath cycle (low 8 bits) |
| `0x07`  | Milliseconds the LED is on during one blink/breath cycle (high 8 bits) |
| `0x08`  | Milliseconds the LED is on during one blink/breath cycle (low 8 bits) |
| `0x09`  | Checksum of data in the range 0x00 - 0x08 (high 8 bits) |
| `0x0a`  | Checksum of data in the range 0x00 - 0x08 (low 8 bits) |

The checksum is a 16-bit value obtained by summing all the data at the corresponding positions as unsigned integers.

We can directly use `i2cget` to read from the relevant registers. For example, below is the status of the power indicator light (purple, blinking once per second, lit for 40% of the time, with a brightness of 180/256):

```
$ i2cget -y 0x01 0x3a 0x81 i 0x0b 0x02 0xb4 0xff 0x00 0xff 0x03 0xe8 0x01 0x90 0x04 0x30
```

### Change Status

By writing 12 bytes to the address `0x00 + LED_ID`, we can modify the current status of the corresponding LED. The meaning of these 12 bytes is as follows:

| Address | Meaning of Corresponding Data |
|---------|--------------------------------|
| `0x00`  | LED ID |
| `0x01`  | Constant: 0xa0 |
| `0x02`  | Constant: 0x01 |
| `0x03`  | Constant: 0x00 |
| `0x04`  | Constant: 0x00 |
| `0x05`  | If the value is 1, it indicates modifying brightness<br/> If the value is 2, it indicates modifying color<br/> If the value is 3, it indicates setting the on/off state<br/> If the value is 4, it indicates setting the blink state<br/> If the value is 5, it indicates setting the breath state |
| `0x06`  | First parameter |
| `0x07`  | Second parameter |
| `0x08`  | Third parameter |
| `0x09`  | Fourth parameter |
| `0x0a`  | Checksum of data in the range 0x01 - 0x09 (high 8 bits) |
| `0x0b`  | Checksum of data in the range 0x01 - 0x09 (low 8 bits) |

For the four different modification types at address `0x05`:
- If we need to modify **brightness**, the first parameter contains brightness information.
- If we need to modify **color**, the first three parameters represent RGB information.
- If we need to toggle the **on/off state**, the first parameter is either 0 (off) or 1 (on).
- If we need to set the **blink/breath state**, the first two parameters together form a 16-bit unsigned integer in big-endian order, representing the number of milliseconds needed to complete one blink/breath cycle. The next two parameters, also in big-endian order, represent the time in milliseconds the LED is on during one blink/breath cycle.

Below is an example for turning off and on the power indicator light using `i2cset`:

```
# turn off power LED
$ i2cset -y 0x01 0x3a 0x00  0x00 0xa0 0x01 0x00 0x00 0x03 0x01 0x00 0x00 0x00 0x00 0xa5 i

# turn on power LED
$ i2cset -y 0x01 0x3a 0x00  0x00 0xa0 0x01 0x00 0x00 0x03 0x00 0x00 0x00 0x00 0x00 0xa4 i
```

## Acknowledgement

ChatGPT, [this V2EX post](https://fast.v2ex.com/t/991429), Ghidra 
