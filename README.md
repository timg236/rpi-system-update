# Raspberry Pi System Update

Just enough buildroot to run docker applications from a light weight
OS wrapper providing verified boot, A/B booting, updates and file system
encryption.

## Intro
The high level approach is to have a single file boot ramdisk that contains
a bootable Linux initramfs that is capable of updating itself and installing
an application component. This image can be signed for secure-boot and
launched using the Raspberry Pi 4 (or newer) network-install feature in the bootloader.

The application component is installed to a local block device (e.g. MMC). 
Typically, a self-updating service such as docker would be used as in this example
however, it should be straightforward to replace docker with other solutions.

See https://github.com/timg236/buildroot/tree/rpi-system-update
for the buildroot packages and defconfig.

The software update process is separate from the component which downloads
new updates to install.  For simplicity, this example uses scripts `curl` to download updates.

## Goals
* Runs on Raspberry Pi 4 and Pi 5 family.
* Minimal patches to core buildroot.
* Easy for users to customize the setup scripts as desired e.g. custom docker setup.
* Easy to verify / understand security model.
* Replacable FOTA model (for when wget isn't enough!)

## System software
The system-software in the boot ramdisk contains:-

* Linux Kernel
* Device-tree overlays
* initramfs containing modules and software required to install and launch
  the application component (e.g. Docker)
* A RSA public key used to verify updates.
* Raspberry Pi GPU firmware (aka start.elf) for Pi4 family.

### Secure-boot
This package shows how signed boot images can be used to securely provision and upgrade the system.
Please see the [secure-boot](https://github.com/timg236/usbboot/blob/master/secure-boot-example/README.md)
tutorial for more information about how to enable secure-boot in the bootloader.

The `rpi-system-update` package also implements an option (`BR2_PACKAGE_RPI_SYSTEM_UPDATE_ENCRYPT_APPLICATION_FS`)
to encrypt the application file-system using LUKS and a private key stored in the OTP.
The key must have already been programmed e.g using the `rpi-otp-private-key` utility in the usbboot repo.

**IMPORTANT:**
* Since this is an example UART login is enabled in order to debug / experiment with the update scripts.
  In a production system this should be disabled when deciding what if any remote access should be permitted.

## Architecture
The system consists of four main components:-

* The application image - installed to an SD card (or build time configurable block-device).
* A network install `boot.img` that can be launched via the Raspberry Pi bootloader. This downloads the full application image to a blank SD card.
* A bootloader EEPROM image configured to download the net-install release from the web server.
* A web-server hosting the latest application image updates (`boot.img` and `boot.sig`) files plus version manifest file.

### Kernel / Firmware
This system uses the latest (rpi-update master) kernel and firmware releases and no custom configuration is required.

### cmdline.txt / config.txt
To customize the firmware `config.txt` or `cmdline.txt` edit the files under `board/raspberrypi-system-update`

### Installation flow

1. The Raspberry Pi bootloader is updated with a custom EEPROM image containing the network install configuration.
2. User powers on the Raspberry Pi holding down a push-button attached to a GPIO.
   * This triggers [GPIO conditional filter](https://www.raspberrypi.com/documentation/computers/config_txt.html#the-gpio-filter) in the EEPROM config to go into network install mode.
3. The network install `boot.img` is downloaded into RAM and executed.
4. `rpi-system-update` is launched via systemd in `install mode`.
5. `rpi-system-update` waits for the block device specified in `/etc/default/rpi-system-update` to appear. It then creates the file-system.
6. `rpi-systme-update` downloads the `version` and `version.sig` files specified in `/etc/default/rpi-eeprom-update`.
7. `rpi-system-update` verifies the authenticity of `version` + `version.sig` using the public key specified in `/etc/default/rpi-eeprom-update`. If the verification fails then the version files are downloaded again.
8. `rpi-system-update` downloads the compressed `boot.img.gz` specified in the version file and the corresponding signature file.
9. `rpi-system-update` decompresses the `boot.img.gz`, verifies the signature. If the signature matches then the `boot.img` is decompressed and installed to either `/boot/boota` or `/boot/bootb` depending on whatever partition is the active partition (the one the system booted from. The inactive partition is referred to as the `TRYBOOT partition`.
10. `rpi-system-update` triggers a reboot.
11. The EEPROM bootloader boots from the specified block device (since the system-update GPIO button has been released).
12. The bootloader loads `boot.img`, checks the signature if secure-boot is enabled then starts booting.
13. `rpi-system-update` is run in `application mode`. If the local file-system is blank then it is formatted as Linux and configured for the specified docker service. Otherwise, it is just mounted and docker starts as normal.

### Update flow
* `rpi-system-update` is run in `download mode` from a systemd timer.
* If a newer version is available it is downloaded if an upgrade is NOT already pending
* After verifying the authenticity of the download `rpi-system-update` copies the new image to the `TRYBOOT partition`
* `rpi-system-update` sets a flag (a file) indicating that there is a pending update ready to be tested.
* When the system is next rebooted the `rpi-system-update` will apply the upgrade.

### Upgrade flow
* `rpi-system-update` is launched by systemd `upgrade mode` early during boot.
* If a `tryboot` upgrade is pending then `rpi-system-update` clears the update-pending flag and runs `sudo reboot "0 tryboot"` to reboot into `tryboot` mode.
* If the system is already in `tryboot` mode then the update was successful so `/boot/auto/autoboot.txt` is modified to swap the active/tryboot partitions.
* The system is rebooted into the new `active partition.`

###  Buildroot packages

### rpi-system-update
This package installs [rpi-system-update](https://github.com/timg236/rpi-system-update) scripts and systemd services.
The package config provides various configuration options so that `/etc/default/rpi-system-update` can be automatically generated from the buildroot defconfig.
See `BR2_PACKAGE_RPI_SYSTEM_UPDATE_*` options in "make menuconfig"

For DEBUG ONLY, this package also creates a user called 'admin' with SSH access. Password authentication is
not enabled, instead define the `authorized_keys` files to be written to `/home/admin/.ssh/authorized_keys`

### raspberrypi-secure-boot
This package installs the `secure-boot` scripts from the [usbboot](https://github.com/raspberrypi/usbboot) to support signing of boot images.

### rpi-firmware
There are some minor enhancements to this package.

* Make it easier to select a custom command line and `config.txt` file from the buildroot board config files.
* Install the overlays to the initrd image to support runtime dtoverlay configuration from the userspace in an initrd.

## Building

### Git clone

```
mkdir -p rpi-buildroot-dev
cd rpi-buildroot-dev
git clone --branch rpi-system-update https://github.com/timg236/buildroot buildroot
git clone --branch rpi-system-update https://github.com/timg236/buildroot buildroot-net-install
git clone --branch rpi-system-update https://github.com/timg236/rpi-system-update rpi-system-update
```

### Configuration
Before building the buildroot image the signing keys, and update server information must be defined.

```
cd buildroot
make raspberrypi-system-update_defconfig
make menuconfig
```

Go to `Target packages -->  System Tools --> rpi-system-update`

**Public key**  
The path of the public key .PEM file must be supplies so that it can be embedded in the image file. This is used to check the authenticity of newer versions of the image.

**Private key**  
If a private-key file is specified then the this will be used to automatically sign the image file. Alternatively, leave this blank to create an unsigned image. That might be desirable if the private is stored in a TPM and you want to sign the images on a separate secure server.

**Update URL**  
The URL of the `version` file that `rpi-system-update` will poll for updates.

**Version number**  
The version number is simply an integer defined by `BR2_PACKAGE_RPI_SYSTEM_UPDATE_VERSION` and embedded in `/etc/default/rpi-system-update`.
After updating the version number the `rpi-system-update` package must be rebuilt.

* The `rpi-eeprom-update` script never downgrades to a lower version number.
* Version numbers do not describe API compatibility.

### Build the image

```
cd buildroot
make raspberrypi-system-update_defconfig
make
```

Output files:-
* output/target/images/boot.img.gz
* output/target/images/boot.sig

The output images can be copied to the download server and the `verison` file update.

#### Configuration updates
**Rebuilding after a configuration change**
```
make raspberrypi-system-update_defconfig && make rpi-system-update-reconfigure && make
```

### Building the network-install image
The network-install image is optimized for size (32-bit kernel) and needs to be built in a different tree.
However, it's unlikely that this would need to be updated frequently because it's sole purpose is to
download and install the real `boot.img`.
```
cd buildroot-net-install
make raspberrypi-system-update-net-install_defconfig
make
```

#### Configuration
Configure the public key, private key and update URL. The version number should be set to 0 since the initial install just calls the update routine.

Output files:-
* output/target/images/boot.img
* output/target/images/boot.sig

### Bootloader EEPROM
The following EEPROM configuration shows how a GPIO conditional filter may be used to start an automated network install image.

Example `rpi-system-update/server/boot.conf`

#### SIGNED_BOOT
If `secure-boot` is enabled in OTP then the contents of the `boot.img` file is always verified using the EEPROM public key. If secure boot is not required then `SIGNED_BOOT` can be placed within the GPIO conditional section so that the signature is only checked for the network install step.

Replace `update.raspberrypi.com` with the domain name of your software update server.
```
[all]
BOOT_UART=1
HDMI_DELAY=0
WAKE_ON_GPIO=1
POWER_OFF_ON_HALT=0

[gpio2=0]
# There is currently no support for custom CA certs so HTTP only for now.
# version.sig and boot.sig verify authenticity.
SIGNED_BOOT=1
BOOT_ORDER=0xf7
HTTP_HOST=update.raspberrypi.com
HTTP_PATH=rpi-system-update/net-install
```

To generate `rpi-eeprom-recovery.zip` run
```
mkdir -p update-server
./rpi-system-update/build-eeprom update-server
```

To use a different EEPROM image specify the filename in the `RPI_EEPROM_IMAGE` environment variable.

### Adding a release to the download server
The `rpi-system-update/server/add-release` script can be used to create populate a new directory in the suitable structure from images built from buildroot.

Edit `rpi-system-update-server/config` to specify the base URL to include
in the `version` file.

Add release version 2 to the update-server directory.  
```
mkdir -p update-server
./rpi-system-update/server/add-release update-server 2
```

Update the net-install images from the buildroot-net-install  
```
mkdir -p update-server
./rpi-system-update/server/add-release update-server net-install
```

Example `version` file - replace with your HTTP (not HTTPS) server name. 
```
version: 2
signature-url: https://update.raspberrypi.com/releases/1/boot.sig
image-url: https://update.raspberrypi.com/releases/1/boot.img.gz
```

### Application configuration
The HomeAssistant docker-application is a real world example to verify the
installation and launching of applications in an encrypted disk. Docker 
applications also benefit from Docker's built in install / OTA mechanism
at the application.

The docker service is selected by defining `BR2_PACKAGE_RPI_SYSTEM_UPDATE_DOCKER_APPLICATION` e.g. `home-assistant`.
