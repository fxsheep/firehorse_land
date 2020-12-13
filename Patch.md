# Bootchain Patching  
The Secureboot bypass allows us to boot an arbitrary SBL1, in other words, a patched SBL1.  
In order to make use of secureboot bypass, we need to patch the bootchain (i.e. disable later checks).  

## Intro  
Some MSM8937 bootchain basics here: https://alephsecurity.com/2018/01/22/qualcomm-edl-1/
In these boot chains, each component is packed as an ELF file, but with an additional hash table of all segments, and a signature chain that signs the table itself.  
Now that we can patch SBL1, it's now possible to apply patches to disable both checks (hash table integrity and signature verification).  
After disabling sigchecks in SBL1, we can now modify things loaded and authed by SBL1.  
For now, we patch TZ(trustzone,QSEE) and use a custom aboot (LK) image, to achieve our convenience.  
RPM firmware is also patchable if you want to.  

## Procedure  
1.Download https://bigota.d.miui.com/9.3.28/miui_HM3S_9.3.28_4a8a385746_6.0.zip and extract firmware-update  
2.Find and patch the TZ:
> bspatch tz.mbn tz_patched.mbn tz_9.3.28.patch  

3.Download http://bigota.d.miui.com/6.6.16/miui_HM3S_6.6.16_eb411d8524_6.0.zip and extract firmware-update  
4.Find and patch the SBL1:  
> bspatch sbl1.mbn sbl1_patched.mbn sbl1_6.6.16.patch  

5.Flash 9.3.28 firmware you downloaded before to the device.  
6.Clone phone EMMC contents to SDCard except userdata partition. ALL DATA ON THE CARD WILL BE DESTROYED.  
(Below is only an example, adjust yourself)
> land:/ # dd if=/dev/block/mmcblk0 of=/dev/block/mmcblk1 bs=1024M count=6

7.Boot into TWRP, adb push emmc_appsboot.mbn(lk1st in this repo), tz_patched.mbn and sbl1_patched.mbn to root directory.
8.adb shell and flash patched boot chains:  
> land:/ # cat sbl1_patched.mbn >/dev/block/platform/soc/7864900.sdhci/by-name/sbl1  
> land:/ # cat tz_patched.mbn >/dev/block/platform/soc/7864900.sdhci/by-name/tz  
> land:/ # cat emmc_appsboot.mbn >/dev/block/platform/soc/7864900.sdhci/by-name/aboot  

9.Prepare a ROM and a TWRP image folloing guides below.  
10.Boot tethered with cmd scripts, (documented in README), while holding volume down till it vibrates and fastboot USB enumerated.  
11.Now only SDCard will be visible, as mmcblk0. FORMAT /data with new TWRP, then flash ROM prepared.
12.Boot tethered again but let aboot boot normally, you'll eventually be greeted with Android bootanim and UI
13.Enjoy SoC-wide freedom.  

## Note  
Only SBL1 is taken from an older firmware version, other images are all from MIUI China 9.3.28.  

## SBL1 patching  

As of now, only some (seems to be older) SBL1s are bootable via patched PBL. Thus an older SBL1 patch is supplied.


## TZ patching  
The TZ known as QSEE, is actually a combined image with a secure monitor(EL3) and QSEE OS (s-EL1).  
It's responsible for secure services such as PIL, trustzone apps environment and more.  
This patch does the following:  
1.Patch all secureboot fuse checks, trick TZ into thinking that it's running on an unfused device, which allows:  
-Loading arbitrary/unsigned subsystem images (except with SSA, i.e. modem) and QSEE apps.
-Demoting the JTAG by default, allows you to attach to the AP processor and other subsystems.(untested)  
2.Disable all XPUs except modem critical regions (modem itself in DDR is still disabled)  
-XPU is a hardware block in Snapdragons that acts as both TZASC and TZPC with more advanced features.  
-Disabling allows you to access the DDR memory across subsystems, and do unauthorized accesses such as patching TZ code region directly from linux.  
3.Remove scm_io_read(write) address limitations.  
-scm_io_read(write) are two SMC calls on most qcom platforms that's used for altering secure peripherals that does not have to be.  
-Patching it allows you to execute readl/writel in TZ conveniently.  
4.Disable qfprom write functions in TZ.  
-This is for safety.All qfprom writings, whether invoked from SMC, or from tzapps, or QSEE itself, are ignored.  
-This lowers the possibility of bricking your device unintentially via blowning any fuses, e.g. antirollback.  

## ADSP patching
Assuming you have applied TZ patch, to patch out ADSP userspace applications signature checks, enable /dev/mem, use the following:  
> busybox devmem 0x8BB44668 32 0x5800C03C  
This offset will work on 9.3.28, and probably all versions. 

## lk1st
Stock aboot does not support booting from SDCard. So I did a custom one with source from CAF.  
Currently display is not working, contributions welcome :)  
I also added some hacks to boot kernel for KVM, offset hardcoded for 9.3.28 TZ. Revert before running another firmware.  
Binary build included in this repo, the emmc_appsboot.mbn  
You can download and compile yourself from https://github.com/fxsheep/lk1st_land  


## Hint
Symbols of ADSP and modem firmwares can be acquired from Xiaomi factory firmwares, and it seems that the factory build of these Q6 FWs are the same as production builds.  
keyword: SW_S88537BA1_V036_M20_MP_XM-MIUI-FACTORY20
