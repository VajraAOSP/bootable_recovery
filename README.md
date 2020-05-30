**Team Win Recovery Project (TWRP)**

You can find a compiling guide [here](http://forum.xda-developers.com/showthread.php?t=1943625 "Guide").

For your convenience, here it is reproduced the above document:

All of TWRP 3.x source is public. You can compile it on your own. This guide isn't going to be a step-by-step, word-for-word type of guide. If you're not familiar with basic Linux commands and/or building in AOSP then you probably won't be able to do this.

You can currently use Omni 5.1, Omni 6.0, Omni 7.1, Omni 8.1, CM 12.1, CM 13.0, CM 14.1, or CM 15.1 source code. Omni 7.1 is recommended. You can build in CM but you may run into a few minor issues. If you don't know how to fix make file issues, then you should choose to use Omni instead.

If you are using CM, you'll need to place TWRP in the CM/bootable/recovery-twrp folder and set RECOVERY_VARIANT := twrp in your BoardConfig.mk file. TWRP source code can be found here:
https://github.com/omnirom/android_bootable_recovery
Select the newest branch available. This step is not necessary with Omni because Omni already includes TWRP source by default, however, if you are using an older version of Omni, you will probably want to pull from the latest branch (the latest branch will compile successfully in older build trees)

If you are only interested in building TWRP, you may want to try working with a smaller tree. You can try using this manifest. It should work in most cases but there may be some situations where you will need more repos in your tree than this manifest provides:
https://github.com/minimal-manifest-twrp

*BEFORE YOU COMPILE*
Note: If you add or change any flags, you will need to make clean or make clobber before recompiling or your flag changes will not be picked up.

Now that you have the source code, you'll need to set or change a few build flags for your device(s). Find the BoardConfig.mk for your device. The BoardConfig.mk is in your devices/manufacturer/codename folder (e.g. devices/lge/hammerhead/BoardConfig.mk).

Your board config will need to include architecture and platform settings. Usually these are already included if you're using device configs that someone else created, but if you created your own, you may need to add them. Without them, recovery may seg fault during startup and you'll just see the teamwin curtain flash on the screen over and over.

We usually put all of our flags at the bottom of the BoardConfig.mk under a heading of #twrp For all devices you'll need to tell TWRP what theme to use. This TW_THEME flag replaces the older DEVICE_RESOLUTION flag. TWRP now uses scaling to stretch any theme to fit the screen resolution. There are currently 5 settings which are: portrait_hdpi, portrait_mdpi, landscape_hdpi, landscape_mdpi, and watch_mdpi. For portrait, you should probably select the hdpi theme for resolutions of 720x1280 and higher. For landscape devices, use the hdpi theme for 1280x720 or higher.
TW_THEME := portrait_hdpi

Note that themes do not rotate 90 degrees and there currently is no option to rotate a theme. If you find that the touchscreen is rotated relative to the screen, then you can use some flags (discussed later in this guide) to rotate the touch input to match the screen's orientation.

In addition to the resolution, we have the following build flags:
RECOVERY_SDCARD_ON_DATA := true -- this enables proper handling of /data/media on devices that have this folder for storage (most Honeycomb and devices that originally shipped with ICS like Galaxy Nexus) This flag is not required for these types of devices though. If you do not define this flag and also do not include any references to /sdcard, /internal_sd, /internal_sdcard, or /emmc in your fstab, then we will automatically assume that the device is using emulated storage.
BOARD_HAS_NO_REAL_SDCARD := true -- disables things like sdcard partitioning and may save you some space if TWRP isn't fitting in your recovery patition
TW_NO_BATT_PERCENT := true -- disables the display of the battery percentage for devices that don't support it properly
TW_CUSTOM_POWER_BUTTON := 107 -- custom maps the power button for the lockscreen
TW_NO_REBOOT_BOOTLOADER := true -- removes the reboot bootloader button from the reboot menu
TW_NO_REBOOT_RECOVERY := true -- removes the reboot recovery button from the reboot menu
RECOVERY_TOUCHSCREEN_SWAP_XY := true -- swaps the mapping of touches between the X and Y axis
RECOVERY_TOUCHSCREEN_FLIP_Y := true -- flips y axis touchscreen values
RECOVERY_TOUCHSCREEN_FLIP_X := true -- flips x axis touchscreen values

TWRP_EVENT_LOGGING := true -- enables touch event logging to help debug touchscreen issues (don't leave this on for a release - it will fill up your logfile very quickly)
BOARD_HAS_FLIPPED_SCREEN := true -- flips the screen upside down for screens that were mounted upside-down

There are other build flags which you can locate by scanning the Android.mk files in the recovery source. Most of the other build flags are not often used and thus I won't document them all here.

*RECOVERY.FSTAB*
TWRP 2.5 and higher supports some new recovery.fstab features that you can use to extend TWRP's backup/restore capabilities. You do not have to add fstab flags as most partitions are handled automatically.

Note that TWRP only supports v2 fstabs in version 3.2.0 and higher. You will still need to use the "old" format of fstab for older TWRP (example of that format is below), and even TWRP 3.2.0 still supports the v1 format in addition to the v2 format. To maximize TWRP's compatibility with your build tree, you can create a twrp.fstab and use PRODUCT_COPY_FILES to place the file in /etc/twrp.fstab When TWRP boots, if it finds a twrp.fstab in the ramdisk it will rename /etc/recovery.fstab to /etc/recovery.fstab.bak and then rename /etc/twrp.fstab to /etc/recovery.fstab. Effectively this will "replace" the fstab 2 file that your device files are providing with the TWRP fstab allowing you to maintain compatibility within your device files and with other recoveries.
Code:
PRODUCT_COPY_FILES += device/lge/hammerhead/twrp.fstab:recovery/root/etc/twrp.fstab
The fstab in TWRP can contain some "flags" for each partition listed in the fstab.

Here's a sample TWRP fstab for the Galaxy S4 that we will use for reference:

/boot            emmc        /dev/block/platform/msm_sdcc.1/by-name/boot
/system          ext4        /dev/block/platform/msm_sdcc.1/by-name/system
/data            ext4        /dev/block/platform/msm_sdcc.1/by-name/userdata length=-16384
/cache           ext4        /dev/block/platform/msm_sdcc.1/by-name/cache
/recovery        emmc        /dev/block/platform/msm_sdcc.1/by-name/recovery
/efs             ext4        /dev/block/platform/msm_sdcc.1/by-name/efs                            flags=display="EFS";backup=1
/external_sd     vfat        /dev/block/mmcblk1p1    /dev/block/mmcblk1                            flags=display="Micro SDcard";storage;wipeingui;removable
/usb-otg         vfat        /dev/block/sda1         /dev/block/sda                                flags=display="USB-OTG";storage;wipeingui;removable
/preload         ext4        /dev/block/platform/msm_sdcc.1/by-name/hidden                         flags=display="Preload";wipeingui;backup=1
/modem           ext4        /dev/block/platform/msm_sdcc.1/by-name/apnhlos
/mdm		     emmc		 /dev/block/platform/msm_sdcc.1/by-name/mdm

Flags are added to the end of the partition listing in the fstab separated by white space (spaces or tabs are fine). The flags affect only that partition but not any of the others. Flags are separated by semicolons. If your display name is going to have a space, you must surround the display name with quotes.

/external_sd  vfat  /dev/block/mmcblk1p1  flags=display="Micro SDcard";storage;wipeingui;removable

The flags for this partition give it a display name of "Micro SDcard" which is displayed to the user. wipeingui makes this partition available for wiping in the advanced wipe menu. The removable flag indicates that sometimes this partition may not be present preventing mounting errors from being displayed during startup. Here is a full list of flags:

removable -- indicates that the partition may not be present preventing mounting errors from being displayed during boot
storage -- indicates that the partition can be used as storage which makes the partition available as storage for backup, restore, zip installs, etc.
settingsstorage -- only one partition should be set as settings storage, this partition is used as the location for storing TWRP's settings file
canbewiped -- indicates that the partition can be wiped by the back-end system, but may not be listed in the GUI for wiping by the user
userrmrf -- overrides the normal format type of wiping and only allows the partition to be wiped using the rm -rf command
backup= -- must be succeeded by the equals sign, so backup=1 or backup=0, 1 indicates that the partition can be listed in the backup/restore list while 0 ensures that this partition will not show up in the backup list.
wipeingui -- makes the partition show up in the GUI to allow the user to select it for wiping in the advanced wipe menu
wipeduringfactoryreset -- the partition will be wiped during a factory reset
ignoreblkid -- blkid is used to determine what file system is in use by TWRP, this flag will cause TWRP to skip/ignore the results of blkid and use the file system specified in the fstab only
retainlayoutversion -- causes TWRP to retain the .layoutversion file in /data on devices like Sony Xperia S which sort of uses /data/media but still has a separate /sdcard partition
symlink= -- causes TWRP to run an additional mount command when mounting the partition, generally used with /data/media to create /sdcard
display= -- sets a display name for the partition for listing in the GUI
storagename= -- sets a storage name for the partition for listing in the GUI storage list
backupname= -- sets a backup name for the partition for listing in the GUI backup/restore list
length= -- usually used to reserve empty space at the end of the /data partition for storing the decryption key when Android's full device encryption is present, not setting this may lead to the inability to encrypt the device
canencryptbackup= -- 1 or 0 to enable/disable, makes TWRP encrypt the backup of this partition if the user chooses encryption (only applies to tar backups, not images)
userdataencryptbackup= -- 1 or 0 to enable/disable, makes TWRP encrypt only the userdata portion of this partition, certain subfuldes like /data/app would not be encrypted to save time
subpartitionof= -- must be succeeded by the equals sign and the path of the partition it is a subpartition of. A subpartition is treated as "part" of the main partition so for instance, TWRP automatically makes /datadata a subpartition of /data. This means that /datadata will not show up in the GUI listings, but /datadata would be wiped, backed up, restored, mounted, and unmounted anytime those operations are performed on /data. A good example of the use of subpartitions is the 3x efs partitions on the LG Optimus G:

/efs1         emmc   /dev/block/mmcblk0p12 flags=backup=1;display=EFS
/efs2         emmc   /dev/block/mmcblk0p13 flags=backup=1;subpartitionof=/efs1
/efs3         emmc   /dev/block/mmcblk0p14 flags=backup=1;subpartitionof=/efs1

This lumps all 3 partitions into a single "EFS" entry in the TWRP GUI allowing all three to be backed up and restored together under a single entry.

As of TWRP 3.2.0, TWRP now supports a version 2 fstab like those that have been found in Android devices for years. Yes, I know we're really slow to adopt this one, but I also saw no major advantage to v2 and the v2 fstab was being used in regular Android as well as recovery and I didn't want full ROM builds crashing or doing other weird things because of TWRP flags being present in the fstab. Version 2 fstab support is automatic. You don’t need to add any build flags. The regular version 1 fstab format is also still valid and it’s possible to use both v1 and v2 types in the same fstab. TWRP 3.2.0 also supports using wildcards via the asterisk in v1 format which can be useful for USB OTG and micro SD cards with multiple partitions. Note also that v2 fstab formats haven’t been extensively tested so developers should test their v2 fstabs before shipping to users (you should always be testing anyway!).

This is a v1 fstab line with a wildcard intended for a USB OTG drive. All partitions should show up in the list of available storage devices when the user plugs in a drive:

/usb-otg  vfat   /dev/block/sda*  flags=removable;storage;display=USB-OTG

This line is straight from the v2 fstab for the same device and also should work. In this case the kernel will notify us that new devices have been added or removed via uevents:

/devices/soc.0/f9200000.ssusb/f9200000.dwc3/xhci-hcd.0.auto/usb*    auto      auto    defaults      voldmanaged=usb:auto

In addition to the v2 fstab, you can include /etc/twrp.flags which uses the v1 fstab format. The twrp.flags file can be used to supplement the v2 fstab with TWRP flags, additional partitions not included in the v2 fstab, and to override settings in the v2 fstab. For example, I have a Huawei device with the following stock v2 fstab present as /etc/recovery.fstab

# Android fstab file.
#<src>                                                  <mnt_point>         <type>    <mnt_flags and options>                       <fs_mgr_flags>
# The filesystem that contains the filesystem checker binary (typically /system) cannot
# specify MF_CHECK, and must come before any filesystems that do specify MF_CHECK
/dev/block/bootdevice/by-name/system    /system    ext4    ro,barrier=1    wait,verify
/dev/block/bootdevice/by-name/cust    /cust    ext4    ro,barrier=1    wait,verify
/devices/hi_mci.1/mmc_host/mmc1/*                       auto                auto      defaults                                      voldmanaged=sdcard:auto,noemulatedsd
/devices/hisi-usb-otg/usb1/*                            auto                auto      defaults                                      voldmanaged=usbotg:auto
/dev/block/bootdevice/by-name/userdata         /data                f2fs     nosuid,nodev,noatime,discard,inline_data,inline_xattr wait,forceencrypt=footer,check
/dev/block/bootdevice/by-name/cache         /cache                ext4      rw,nosuid,nodev,noatime,data=ordered wait,check
/dev/block/bootdevice/by-name/splash2         /splash2                ext4      rw,nosuid,nodev,noatime,data=ordered,context=u:object_r:splash2_data_file:s0 wait,check
/dev/block/bootdevice/by-name/secure_storage         /sec_storage                ext4      rw,nosuid,nodev,noatime,discard,auto_da_alloc,mblk_io_submit,data=journal,context=u:object_r:teecd_data_file:s0 wait,check

In addition I have also included this in /etc/twrp.flags:

Code:
/boot         emmc       /dev/block/platform/hi_mci.0/by-name/boot
/recovery     emmc       /dev/block/platform/hi_mci.0/by-name/recovery   flags=backup=1
/cust         ext4       /dev/block/platform/hi_mci.0/by-name/cust       flags=display="Cust";backup=1
/misc         emmc       /dev/block/platform/hi_mci.0/by-name/misc
/oeminfo      emmc       /dev/block/platform/hi_mci.0/by-name/oeminfo    flags=display="OEMinfo";backup=1
/data         f2fs       /dev/block/dm-0
/system_image emmc       /dev/block/platform/hi_mci.0/by-name/system

The first 2 lines in twrp.flags adds the boot and recovery partitions which were not present at all in the v2 fstab. The /cust line in the twrp.flags file is added to tell TWRP to allow users to back up the cust partition and to give it a slightly better display name. The /misc partition is also only present in the twrp.flags file. Much like the /cust partition, the /oeminfo partition is in the twrp.flags file to tell TWRP to allow users to back it up and give a display name. The /data line is needed because this Huawei device, like many Huawei devices, is encrypted but the encryption uses some special Huawei binaries and is encrypted with some sort of default password that the user cannot change. We use the Huawei binaries to decrypt the device automatically in recovery. The /data line here tells TWRP to use /dev/block/dm-0 instead of /dev/block/bootdevice/by-name/userdata which is required for proper mounting, etc. Lastly we have the /system_image line so that TWRP will add a system image option for backup and restore.

As we add more new devices, we’ll add more example device trees to https://github.com/TeamWin/ which should help you find more ways to use this new fstab support. Please note that using the v2 fstab format at this point is completely optional, so feel free to continue using v1 if that is what is more comfortable or if you have trouble with the v2 format support.

If you have questions, feel free to stop by #twrp on Freenode. If you post here I may not see it for a while as I have lots of threads out there and there's no way for me to keep track of them all. If you successfully port TWRP to a new device, please let us know! We love to hear success stories!

If you have code changes that you'd like to submit, please submit them through the Omni Gerrit server. Guide is here.

Once you get Omni or CM sync'ed and your TWRP flags set, you should do a source ./build/envsetup.sh We usually lunch for the device in question, so something like "lunch omni_hammerhead-eng".

After you lunch successfully for your device this is the command used for most devices:

make clean && make -j# recoveryimage

Replace the # with the core count +1, so if you have a dual core it's -j3 and a quad core becomes -j5, etc. If you're dealing with a "typical" Samsung device, then you'll need to

make -j# bootimage

Most Samsung devices have the recovery included as an extra ramdisk in the boot image instead of a separate recovery partition as found on most other devices.
