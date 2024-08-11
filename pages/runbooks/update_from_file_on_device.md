---
title: Update from file on device
parent: Runbooks
---

# Update from a file on the device

From the Nerves application project directory, build the firmware.

```
MIX_TARGET=<target> mix firmware
```

Connect to the device with SFTP and transfer the firmware bundle (`.fw`) onto the device. Modify the lines below to match your project.

```
sftp <device ip address>
cd /data
lcd _build/<target>_dev/nerves/images/
lls
put <app>.fw
quit
```

Connect to the device's console and run `fwup` from IEx to apply the firmware update. Reboot once the update is complete. The device will come back up with the new firmware running.

```elixir
cmd "fwup -a -t upgrade -d /dev/<disk> -i /data/<app>.fw"
reboot
```

The firmware bundle can be removed from the device if it is no longer needed.

```elixir
File.rm "/data/<app>.fw"
```

> 🛈 Tip
>
> `scp` is an alternative to `sftp`. However, on some systems `scp` responds with `lost connection` and fails to transfer the file. The tool used for the file transfer is not important as long as the firmware file makes it onto the device.

## Finding the disk

The fwup `-d` flag specifies the disk to apply the update to. This is documented by the `NERVES_FW_DEVPATH` property in your Nerves system fwup config.

If you need to find the disk without being able to reference the config, run fwup on the device without the `-d` flag.

```elixir
cmd "fwup -a -t upgrade -i /data/<app>.fw"
```

This will attempt to auto-detect the disk and will print what it has discovered. However, the console will hang when doing so.

```
Use 7.95 GB memory card found at /dev/mmcblk0? [y/N]
```

Break out of the command with Ctrl+C. Copy the path to the disk and paste it after the `-d` flag in the full fwup command above.

## Background

A Nerves device doesn't have to receive a firmware update command remotely. [fwup](https://github.com/fwup-home/fwup?tab=readme-ov-file#overview) is used to manage firmware updates, and can be run directly on a Nerves device. This is actually how the OTA update process works under the hood.

In this runbook we manually place a firmware file on the device, and then call `fwup` on the device to perform the firmware update. Once the device is rebooted, it will come up with the new firmware.
