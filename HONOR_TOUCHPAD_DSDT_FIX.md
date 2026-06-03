# Honor MagicBook FMB-P: fix Linux touchpad with DSDT override

This guide documents the fix for Honor MagicBook / MagicBook X16 class laptops
where the touchpad is not detected at all on Linux. The tested machine was:

- DMI product: `HONOR FMB-P`
- BIOS: `1.13`, date `05/08/2025`
- Linux: Arch/Omarchy with Limine UKI boot

## Symptoms

The touchpad is missing from both input devices and Hyprland/libinput:

```bash
cat /proc/bus/input/devices
hyprctl -j devices
```

Kernel logs show ACPI/I2C/Intel LPSS problems:

```bash
journalctl -b -k --no-pager | rg -i "intel-lpss|i2c|acpi|PNP0C50|touch"
```

Typical lines:

```text
ACPI BIOS Error: Could not resolve symbol [\_SB.PC00.I2C3]
intel-lpss 0000:00:15.0: can't derive routing for PCI INT A
intel-lpss ... probe with driver intel-lpss failed
```

This means the problem is below Hyprland/libinput. The kernel does not expose
the touchpad as an input device because the firmware ACPI/DSDT table is broken.

## Reference Patch

The useful reference patch is:

https://github.com/denis-bb/honor-fmb-p-dsdt

Do not blindly install the ready-made `.aml` from that repo. On this machine,
the local DSDT differed from the repo's prebuilt AML. The safer approach is to
extract the current laptop's DSDT, apply the same kind of fixes, compile a local
AML, and load it through initramfs.

## Install Tools

```bash
sudo pacman -S --needed acpica git
```

## Download the Reference Patch

```bash
git clone --depth 1 https://github.com/denis-bb/honor-fmb-p-dsdt.git /tmp/honor-fmb-p-dsdt
```

## Extract Local DSDT

```bash
sudo cp /sys/firmware/acpi/tables/DSDT /tmp/local-dsdt.dat
sudo chmod 0644 /tmp/local-dsdt.dat
```

Disassemble it:

```bash
cd /tmp
iasl -d /tmp/local-dsdt.dat
```

This creates:

```text
/tmp/local-dsdt.dsl
```

## Patch Local DSDT

Create a working copy:

```bash
cp /tmp/local-dsdt.dsl /tmp/local-dsdt-honor-patched.dsl
```

Remove the problematic `EFUN.CRFI` externals, temporary `PS0X/PS3X` calls,
and bump the DSDT OEM revision:

```bash
perl -0pi -e 's/\n\s*External \([^\n]*EFUN\.CRFI[^\n]*\)//g; s/\n\s*External \(_SB_\.PC0[02]\.XHCI\._PS[03]\.PS[03]X, MethodObj\)\s*\/\/ 0 Arguments//g; s/PS0X \(\)//g; s/PS3X \(\)//g; s/DefinitionBlock \("", "DSDT", 2, "HONOR", "ARL", 0x00000002\)/DefinitionBlock ("", "DSDT", 2, "HONOR", "ARL", 0x0000000c)/' /tmp/local-dsdt-honor-patched.dsl
```

Remove the broken `NFC0` device block:

```bash
perl -0pi -e 's/\n\s*Device \(NFC0\)\s*\{\s*Name \(_ADR, Zero\).*?\n\s*\}\n\s*(?=Name \(_DSD, Package \(0x02\))/\n/s' /tmp/local-dsdt-honor-patched.dsl
```

Verify that only `DefinitionBlock` remains from the search below:

```bash
rg -n "EFUN\.CRFI|PS0X \(\)|PS3X \(\)|Device \(NFC0\)|DefinitionBlock" /tmp/local-dsdt-honor-patched.dsl
```

Expected output is similar to:

```text
21:DefinitionBlock ("", "DSDT", 2, "HONOR", "ARL", 0x0000000c)
```

## Compile AML

```bash
iasl -ve -tc /tmp/local-dsdt-honor-patched.dsl
```

The important part is `0 Errors`:

```text
Compilation successful. 0 Errors
AML Output: /tmp/local-dsdt-honor-patched.aml
```

Warnings and remarks can remain. Do not continue if there are compile errors.

## Install ACPI Override

```bash
sudo install -d /etc/initcpio/acpi_override
sudo cp /tmp/local-dsdt-honor-patched.aml /etc/initcpio/acpi_override/dsdt.aml
```

Add the `acpi_override` hook after `microcode`.

For standard Arch Linux:

```bash
sudo sed -i '/^HOOKS=/ { /acpi_override/! s/microcode /microcode acpi_override / }' /etc/mkinitcpio.conf
```

For Omarchy, the active hook list is also overridden by a drop-in, so patch it
too:

```bash
sudo cp /etc/mkinitcpio.conf.d/omarchy_hooks.conf /etc/mkinitcpio.conf.d/omarchy_hooks.conf.bak-honor-dsdt
sudo sed -i '/^HOOKS=/ { /acpi_override/! s/microcode /microcode acpi_override / }' /etc/mkinitcpio.conf.d/omarchy_hooks.conf
```

## Rebuild Initramfs / UKI

For Omarchy with Limine:

```bash
sudo limine-mkinitcpio
```

The output must include:

```text
Running build hook: [acpi_override]
```

If this line is missing, the override did not enter the active initramfs/UKI.

For a standard Arch setup without Omarchy/Limine, use the appropriate mkinitcpio
command for your kernel preset, for example:

```bash
sudo mkinitcpio -P
```

## Reboot

```bash
sudo reboot
```

## Verify After Reboot

Check kernel logs:

```bash
journalctl -b -k --no-pager | rg -i "ACPI table|DSDT|intel-lpss|i2c_hid|PNP0C50|touch|GXTP|SYNA|ELAN"
```

Check input devices:

```bash
cat /proc/bus/input/devices
hyprctl -j devices
```

If the fix worked, the touchpad should appear as an I2C HID device such as
`PNP0C50`, `SYNA*`, `GXTP*`, or `ELAN*`, and Hyprland/libinput should detect it.

## Rollback

If the system boots but you want to remove the override:

```bash
sudo rm /etc/initcpio/acpi_override/dsdt.aml
sudo sed -i 's/ acpi_override//' /etc/mkinitcpio.conf
sudo sed -i 's/ acpi_override//' /etc/mkinitcpio.conf.d/omarchy_hooks.conf
sudo limine-mkinitcpio
sudo reboot
```

If the system does not boot, use an older snapshot or fallback boot entry, then
remove the override from the installed system.

