# alienware-13-r3-ubuntu-16.04

This is guide where I have dumped my personal steps to getting Ubuntu 16.04 running smoothly with my Alienware 13 R3.

Please be aware that your mileage my vary. I am definitely no Linux/Ubuntu expert. Feedback is most welcome. Please raise an issue!

**I take no responsibility for any damage or problems that arise as a result from following this guide.**

# READ THIS FIRST
Running Linux on "hybrid" (Nvidia Optimus) is a little bit more tricky. It's important that you spend a little time understanding what this is about and what trade-offs you want to make.

I highly recommend that you take read the following before continuing:

* [Dell's KnowledgeBase article on Nvidia Optimus and Ubuntu](http://www.dell.com/support/Article/au/en/aubsdt1/SLN298431/EN)
* [A great summary of Nvidia Optimus (from one of the bumblebee devs)](http://askubuntu.com/questions/766745/do-i-need-to-install-bumblebee-for-hybrid-graphics-system-to-enable-optimus-on-u/775161)

**This guide will result in primarily using the integrated GPU (intel) with the option of invoking the discrete (NVIDIA GTX 1060) card via bumblebee.**

My personal reasons are:

* get the most out of the 76 watt-hour-battery by using the integrated GPU as much as possible
* able to explicitly leverage the discrete (NVIDIA) GPU with applications such as Blender
* primarily do my gaming in Windows
* currently the most stable configuration that I've been able to achieve

**Please be aware that `sudo prime-select nvidia` will not work as expected with this guide.** 

# Preparation

## BIOS
Turn off `Secure Boot` in the BIOS. This will make it easier to install several 3rd party drivers.

I recommend leaving `UEFI` on. Be aware that to get entries in your bios, you will probably need to manually add the appropriate *.efi file.

I believe that you need to switch the SATA interface from `RAID` to `AHCI` before you're able to install Ubuntu.
(I couldn't get the Ubuntu installer to detect the drive when using RAID but maybe this could be resolved with extra drivers). 

**WARNING: Switching the SATA interface will cause Windows to BSOD.**

## Dual-boot with Windows
I want to make this clear. **The easiest way to get dual-booting working is to re-install Windows.**

Be very wary of guides that claim that continue to get Windows after switching the SATA interface to `ACHI`. You can very easily stop Windows booting properly. Even with recovery points, don't expect this is easy to undo.  

I highly recommend that you backup everything and prepare to reinstall Windows 10 from scratch.
 
Installing Windows 10 is easy. Simply create a bootable USB using the Windows 10 Media Creation Tool.

You won't need a product key. (I believe this is stored on the machine somehow, perhaps in BIOS.)

Install Windows first. Get the drivers from Dell's website and get it all working. If necessary, shrink the partition to make room for Ubuntu.
   
## Ubuntu installer
Creating a bootable USB Ubuntu installer can be done using Rufus, using [these instructions](https://www.ubuntu.com/download/desktop/create-a-usb-stick-on-windows).

## Wifi not working after suspending in Ubuntu
I had an intermittent issue where Wifi wouldn't always come back after suspending. If you hit this problem, [this solution](http://askubuntu.com/questions/564556/cant-connect-to-wifi-after-suspend) seemed to fix it for me.

# Instructions

## Ubuntu installer
Install Ubuntu using the installer.

## Disable screen turn-off
Go into `System Settings...`. In `Brightness & Lock`, change `Turn off screen when inactive for:` to `Never`.

## Fix the DPI
Go into `System Settings...` again. In `Screen Display`, change `Scale for menu and title bars:` to `1.5`.

Otherwise you will crash when trying to turn the screen back on.
 
## Install Nvidia 367

````
sudo apt purge 'nvidia.*'
sudo add-apt-repository ppa:graphics-drivers/ppa
sudo apt update
sudo apt install nvidia-367 nvidia-prime
````

## Indicate that you want to primarily use the integrated graphics card

````
sudo prime-select intel
````

I can't confirm if a newer version works, but 367 worked for me.

## Do a general update/upgrade

````
sudo apt-get update
sudo apt-get upgrade
sudo apt-get dist-upgrade
````

## Intel Graphics Update Tool
Install v2.0.2 and run the [Intel Graphics Update Tool](https://01.org/linuxgraphics/downloads/update-tool).

## Reboot and verify that you can login correctly

Verify that you're using the integrated GPU. Run the following and confirm it returns `intel`:

````
prime-select query
````

## Modify grub defaults
We're going to fix 2 things:

* a bug which causes `lspci` or `lshw` to hang
* disable gpu-manager, which crashes resuming after suspending when using the integrated GPU

````
sudo vi /etc/default/grub
````

Modify `GRUB_CMD_LINUX_DEFAULT` and add options `nogpumanager` and `acpi_osi=! acpi_osi="Windows 2009"`.

For example, mine looks like this now:

````
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash nogpumanager acpi_osi=! acpi_osi=\"Windows 2009\""
````

Regenerate/update:

````
sudo update-grub
````

## Reboot
After rebooting, check

* that you can correctly suspend/resume.
* running `lspci` or `lshw` doesn't crash

## Install Bumblebee
These steps here were taken from [this guide.](http://www.webupd8.org/2016/08/how-to-install-and-configure-bumblebee.html).

### Add the Bumblebee testing repository and install bumblebee:

````
sudo add-apt-repository ppa:bumblebee/testing
sudo apt update
sudo apt install bumblebee
````

### Blacklist nvidia-367:

````
sudo vi /etc/modprobe.d/bumblebee.conf
````

### Insert the following lines:

````
# 367
blacklist nvidia-367
blacklist nvidia-367-updates
blacklist nvidia-experimental-367
````

## Configure Bumblebee:

````
sudo vi /etc/bumblebee/bumblebee.conf
````

Change to the following:

* Driver=nvidia
* KernelDriver=nvidia-367
* LibraryPath=/usr/lib/nvidia-367:/usr/lib32/nvidia-367
* XorgModulePath=/usr/lib/nvidia-367/xorg,/usr/lib/xorg/modules

## Reboot
Reboot and afterwards confirm the following returns `OFF`:

````
cat /proc/acpi/bbswitch
````

## optirun & nvidia-settings

You're able to configure the nvidia settings using the below command:

````
optirun -b none /usr/bin/nvidia-settings  -c :8
````

Whilst this is running `cat /proc/acpi/bbswitch` should now return `ON`.

## Install Nvidia Power Indicator
This is a great little indicator that helps you know if the discrete GPU is being used.

````
sudo add-apt-repository ppa:nilarimogard/webupd8
sudo apt update
sudo apt install nvidia-power-indicator
````

# Other tips

## Fix lag when resizing windows
Follow this guide. Personally, instead of `outline`, I selected `rectangle`.

## Fix flickering in Chrome
When the window is maximized and you scroll, you can sometimes see flicker.

The solution was to modify `/usr/share/applications/google-chrome.desktop` and update to the following:

````
Exec=/usr/bin/google-chrome-stable --disable-gpu-driver-bug-workarounds --enable-native-gpu-memory-buffers %U
````
## Adjusting brightness
There's some great scripts on a [reddit thread](https://www.reddit.com/r/Alienware/comments/5l57hu/aw13_r3_oled_ubuntu_everything_good_but_the/dc7wcvd/) which can continously monitor and restore brightness.

Personally, I just use a script that executes the following:

````
xrandr --output eDP1 --brightness 0.35
````



