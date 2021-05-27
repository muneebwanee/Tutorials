# How To Install Kali Linux as a Portable Live USB for Pen-Testing & Hacking on Any Computer

Kali Linux is the go-to Linux distribution for penetration testing and ethical hacking. Still, it's not recommended for day-to-day use, such as responding to emails, playing games, or checking Facebook. That's why it's better to run your Kali Linux system from a bootable USB drive.

The hacker-friendly Debian-based distro did receive a major update by Offensive Security in late-2019 that changed the default desktop environment from the heavyweight Gnome to a more lightweight Xfce, making Kali more snappy and responsive overall. But we still can't recommend it as your daily driver unless you hack 24 hours a day, 7 days a week.

Using Kali in a dual-boot situation is the way to go if you have a dedicated machine, but for something more portable, the live version on a USB flash drive is what you want. If you have a spare computer that you're going to be using for your white-hat endeavors only, then yes, by all means, install Kali as the primary system so that you can take full use of the computer's hardware. 


## What You'll Need

To follow along, you'll need a USB flash drive. You could get away with using one as small as 4 GB for the basics or 8 GB if you want persistence, but a larger one may come in handy, especially if you want the ability to save data. When it comes to 4 GB USB sticks, you'll mostly only find USB 2.0 speeds unless you spend a crazy amount of money on an Kingston IronKey or iStorage datAshur Pro with 256-bit encryption.


You'll also need the Kali Live ISO file and an imaging program like Etcher or Rufus, which we'll outline below.


### Step 1

#### Download the Right Kali Linux Version

While there are many different types of Kali Linux images, the one we want for a portable live version is the "Live" download. You can choose between 64-Bit for AMD (for Intel chips), 64-Bit for ARM64 (such as the M1 chips in newer Macs), and 32-Bit for i386 (which you likely won't ever use because it's so outdated).

Visit kali.org/downloads and download the appropriate Live image. We'll be using the 64-Bit for ARM64 since we're using an M1 Mac. No matter which one you choose, note that you will only be able to use the Xfce desktop environment, and you won't be able to specify additional (meta)packages to install.

<p align="center">
<a href="https://1.bp.blogspot.com/-l2FZhC0MUR4/YK8MndZaj8I/AAAAAAAAA5c/wgtCr_g0f0AfnOsyw3Iyy2KvoqMjoyb7QCLcBGAsYHQ/s1280/1.jpg"></a>
</p>

### Step 2

#### Install a USB Formatting Tool

On Linux and macOS computers, you could use the dd command to copy over the Kali Live image to the USB drive, but you always run the risk of choosing the wrong drive and royally screwing yourself over by overwriting an important drive. Because of that, we recommend using a USB formatting tool. If you're on Windows, Rufus is a good tool, but Etcher works for Linux, macOS, and Windows, so we're using that.


Head to balena.io/etcher and his the "Download" button, which should detect which system you're on ad give you the right version. If not, hit the drop-down button next to it to choose your OS. Once downloaded, install it like you would any other app. For example, on macOS, open the DMG, drag balenaEtcher over to your Applications folder, then eject the DMG and delete it.

<p align="center">
<a href="https://1.bp.blogspot.com/-PUN-13sFl_0/YK8Ms4rkNlI/AAAAAAAAA5g/djLupFXMKi4o7rDH30xk1G-qXPGq30YuwCLcBGAsYHQ/s1280/2.jpg"></a>
</p>

### Step 3

#### Flash the Kali Live Image to the Flash Drive

Open Etcher, then click on "Select Image." You may see options for "Flash from file" and "Flash from URL" instead; choose "Flash from file" if you see that.

<p align="center">
<a href="https://1.bp.blogspot.com/-Pg2W6ITMtFE/YK8Mx0ZFGAI/AAAAAAAAA5k/Uv8O6eqOfUIXkhRZD0IyFuiP70UPV0sCgCLcBGAsYHQ/s1280/3.jpg"></a>
</p>

Navigate to the Kali image you downloaded in Step 1, and select it. Then, you need to click on "Select Target" to choose your USB flash drive. Double- and triple-check that you're selecting the right drive by looking at the drive name and space available.

<p align="center">
<a href="https://1.bp.blogspot.com/-5EO9Z7ujJtk/YK8M3eoBZDI/AAAAAAAAA5o/3q9WdZjN_7E_-ES1kPDvC_J__HpwB2-jQCLcBGAsYHQ/s1280/4.jpg"></a>
</p>

Now all that's left to do is click on "Flash!" This will reformat your flash drive so that everything will be erased and overwritten with the Kali Live image.

<p align="center">
<a href="https://1.bp.blogspot.com/-jVhBuhwRP50/YK8M7zjzsVI/AAAAAAAAA5s/rp9rwfYvot4CfO-fQhigqLOX1NVTQ8W4ACLcBGAsYHQ/s1280/5.jpg"></a>
</p>

You may be asked to input your admin password to let Etcher do its magic, so go ahead and do that if it happens. Then, Etcher will show a progress bar indicating how much time is left for flashing the content over. You'll see a "Flash Complete" message when it's done.

<p align="center">
<a href="https://1.bp.blogspot.com/-L3X36hE8ZlU/YK8NBNLikDI/AAAAAAAAA50/-bT_G20vJqQBDwLzafmOyKNJxSrUEDRoQCLcBGAsYHQ/s1280/6.jpg"></a>
</p>

### Step 4

#### Boot from the Kali Live USB

Now it's time to boot to your new Kali Live USB flash drive, but the process will vary based on the computer brand, operating system, and processor. In the Cyber Weapons Lab video above, Nick shows what happens when you boot into the Asus UEFI BIOS Utility on a Linux machine and change up the boot order if you want Kali Live to boot without selecting it.
On macOS, it's much easier. If you have an Apple M1 chip, just power down your Mac, then turn it on and hold the power button until you see the startup window. On Intel-based Macs, press the Option key immediately after turning on or restarting the computer until you see the startup window. Then, just select the Kali Live flash drive to boot from.


To find out how to enter BIOS and change the boot settings, and/or load the startup menu, google "boot from USB drive" for your computer model and OS and find the appropriate instructions. There are just too many to list here where there are plenty of good manuals online with instructions for booting from external disks.


### Step 5

#### Choose How You Want to Boot Kali

Once you've booted into the Kali Live USB flash drive, you should see a few different options for which version of Kali you want to load. They may include:

Live system: This will boot up Kali Live. In this mode, you cannot save any changes. No reports, no logs, no other data — none of these changes are saved. This way, you start with a fresh Kali Live every time you boot it up. Data is only saved to RAM, not the drive.

Live system (fail-safe mode): Same as the live system, but a more robust version in case the system fails. That way, the failed system won't ruin your flash drive. This is a good option if you need to troubleshoot a problematic computer.

Live system (forensic mode): Same as the live system with forensic-friendly tools so that you can recover files, gather evidence, and perform other forensic tasks on a host machine. However, the internal hard drive is never touched, and external devices and media will not auto-mount so that you have complete control over what you can connect to.

Live system (persistence): Same as the live system, only changes will be saved, and it will allow you to inspect the host system without worrying about running or locked processes.

Live system (encrypted persistence): Same as the live system (persistence), only encrypted with LUKS, so it's harder for others to access your data.

Start installer: This lets you start the installer to install Kali on an internal drive.

Start installer with speech synthesis: Same as start installer, only with speech instructions to help navigate.

Advanced options: Includes options such as MemTest, Hardware Detection, etc.

To boot it up without saving any changes, just choose the "Live system" option, which will take you right into the new Xfce desktop environment as a non-root user. However, if you want to save your changes, boot using "Live system (persistence)" or "Live system (encrypted persistence)" instead.


### Step 6

#### Set Up Persistence on Kali Live

Choosing one of the persistence options doesn't mean it'll work out of the box. There is some configuration to perform first, mainly creating a new partition to save all your data. Kali has some excellent instructions on doing so, which we are using.

To create a partition above the Kali Live partitions, ending at 7 GB, issue the following three commands in a terminal window as the kali user, which will keep the Live options in the 7 GB partition, freeing up the rest for your data storage. Make sure you're doing this from your Kali Live system you just booted into.

* `~$ end=7GiB
`
* `~$ read start _ < <(du -bcm kali-linux-2021.1-live-arm64.iso | tail -1); echo $start 
`
* `~$ parted /dev/sdb mkpart primary ${start}MiB $end
`

When that's done, you should now have a third partition labeled something like /dev/sdb3 or /dev/disk1s3. Check your volume identifier because you'll need it for the rest of this process.

If you want to add some encryption in case you think someone might get a hold of your drive, issue the following two commands as kali. Type "YES" in all caps if prompted to proceed with overwriting the new partition. When asked for a passphrase, choose one and enter it twice. You can skip encryption if you don't want it.

* `~$ cryptsetup --verbose --verify-passphrase luksFormat /dev/sdb3
`
* `~$ cryptsetup luksOpen /dev/sdb3 my_usb
`

Next, run the following two commands as kali to create an ext3 file system labeled "persistence." If you chose to skip encryption, use these:

* `~$ mkfs.ext3 -L persistence /dev/sdb3`
* `~$ e2label /dev/sdb3 persistence
If you encrypted the partition, use:`
* `~$ mkfs.ext3 -L persistence /dev/mapper/my_usb`
* `~$ e2label /dev/mapper/my_usb persistence`

Next, create a mount point, mount the new partition there, create a configuration file to enable persistence, then unmount the partition. If you didn't encrypt the partition, use:

* `~$ mkdir -p /mnt/my_usb`
* `~$ mount /dev/sdb3 /mnt/my_usb`
* `~$ echo "/ union" > /mnt/my_usb/persistence.conf`
* `~$ umount /dev/sdb3`

If you encrypted the partition, use:

* `~$ mkdir -p /mnt/my_usb/`
* `~$ mount /dev/mapper/my_usb /mnt/my_usb`
* `~$ echo "/ union" > /mnt/my_usb/persistence.conf`
* `~$ umount /dev/mapper/my_usb`

If you didn't encrypt the partition, you're done. If you did, you have one more command to issue as kali, which will close the encrypted channel for the new partition:

* `~$ cryptsetup luksClose /dev/mapper/my_usb`

Now, reboot into Kali Live and choose the appropriate system, and everything should be all set.


### Step7 

#### Add a Password for the Root User

On new versions of Kali, you're a non-root user by default, but you can issue root commands as a regular user by using:

* `~$ sudo su`
If you're asked for a password, use kali. However, Kali Live might not attach a password to the root user, so you'll have to create one if you don't want other people to get in and mess with your system. If it does have one, you can change it to something better. Use passwd root with root privileges to do that. Enter your chosen password, then verify it.

~# passwd root New password: Retype new password: passwd: password updated successfully

#### Explore Your New Kali Live System

Now all that's left to do is use Kali. You can check out the Cyber Weapons Lab video above for ideas on what you could configure, as well as our guide on installing Kali as your primary system, which goes over a lot of tools and settings to install and tweak.

### Thank you all for your attention!

### * This Article was written by : muneebwanee (https://muneb.rf.gd)
