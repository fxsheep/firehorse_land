# firehorse_land
This is a fun project utilizing "peek and poke" function in Redmi 3S's EDL programmer,
which leads to [arbitary code execution](https://alephsecurity.com/2018/01/22/qualcomm-edl-1/) in highest privilege level of its Application Processor.

The ultimate goal of this project is to (re)load the rest of the boot chain from the LK, bypassing Secure Boot,
and use backup partitons (*bak) on the EMMC to store tampered firmwares and boot them in EDL while keeping the stock firmware,
making it a Semi-Tethered jailbreak.

## Current status:
Supports booting tethered with SBL1 patched on the fly.  
Supports booting tethered with SBL1 patched on the fly, from SDCard.  
Supports booting from EDL to a custom LK(little kernel, aboot),[source](https://github.com/fxsheep/lk4edl).
Nothing much is working at this point, as RPM firmware isn't loaded(yet).
### Working (lk4edl):
USB
### Not working (lk4edl):
Vibrator, Display, etc.
### Not working (tethered boot):
On board EMMC (unresponsive in Android kernel)  

## Usage_lk4edl (use at your own risk)
1. Download [firehorse research framework](https://github.com/alephsecurity/firehorse)
2. Download this repo and place all files under firehorse\host.(firehorse is the repo you downloaded in 1)
3. Setup firehorse framework properly ,you just need to install QPST, Python2.7 and configure constants.py.
4. Edit firehorse\host\exploit.cmd , modify port variable according to your EDL port number.
5. Put your Redmi 3S in normal mode, check if adb is working.
6. Run exploit.cmd , wait for the exploit to run.
7. Your device will enter EDL mode and the custom LK will be loaded and executed in less than a minute.
8. You'll see the 9008 serial port disappears and a Fastboot device shows up.
9. Use fastboot oem lk_log to see LK boot messages.

## Usage_SecureBoot_Bypass (use at your own risk)
Clone EMMC contents to a good enough SDCard, insert the card and boot with exploit_mota_boot_release.cmd.  
(To be continued)  
(Note that all offsets are hardcoded for firmware from MIUI China 9.3.28)  

## Notes
Based on the research done by alephsecurity,it's not difficult to control the PC with poke function,by changing the LR
in the stack, in our case @0x8057ee4. 

So this is done by mainly two parts, first is to load the arbitary code we want to run in a convenient and efficient way.

Alephsecurity wrote a XMLHunter that first send the binary encoded as XML and then upload a hunter to search for (and decode) it.
However I failed to port it to land for unknown reasons(causing a reboot at random stuffs).It's not that fast anyway,although
much faster than using poke.
Therefore instead of messing with the programmer , I just used an extremely easy way:
By sending a read storage command to the programmer, it will read the target data from the MMC to a fixed buffer in RAM (0x80100000).
However,the buffer isn't that big and the first 0x4000 bytes will be overwritten by the programmer as well.
So I appended the desired binary to a 0x4000 bytes padding, in order to avoid the overwrite.

Then it's time to run it, but before that you need to set DACR to 0xffffffff in order to execute on that page.
Therefore a fairly simple assembly (loader.S) is written to do that and jump to the image.It must be simple enough,
so that loading it with poke won't take too much time.

As for the custom LK,since it's running in pure AArch32 and there is no other stuffs like Secure Monitor etc, SMC will fail.
Commenting related codes out, and it should go far enough to fastboot mode. 

## Update 20/10/06
A few days ago I suddenly realized that some parts of the PBL code is still patchable with my broken remap.So I insented a patch to SBL1  after PBL has authenticated it, and surprise! It worked. This allowed me enough to boot to fastboot.
(To be continued)


## Update 20/03/26 #2
Today is a great day.After a lot of attempts, now I'm finally able to do:  
1.Booting from EDL mode to custom LK.  
2.PBL tethered patching.  
This [commit](https://github.com/fxsheep/lk4edl/commit/63881042c9a2ed53b47c704b13baf9a3a9587007) includes a sample patch that modifies the sbl1 partition GUID the PBL looks for.After patching it and booting the patched PBL, it immediately enters 9008 mode, as the corresponding sbl1 partition is not found.This is a significant step to my final goal, as I can do whatever I want to the PBL clone (tethered).It's time to RE the PBL and find out secureboot verify stuffs to patch.   
Just one more weirdness is that the PBL patch to set DACR will stuck the device.This patch exists even in alephsecurity's most successful example, nokia 6's demo.Currently I just left that out.It's not required anyway, as I'm not implementing a runtime debugger like alephsecurity.

## Update 20/03/26  
I borrowed MMU page remap codes from firehorse framework, remapping individual pages now works.Time to get PBL patched.My first goal is to make my patched PBL boot from sbl1bak instead of sbl1 partition.This is quite easy to patch as I don't need to mess with reverse-engineering much.Qualcomm identify firmware partitions all by GUID in their known implementions, I think it applies to PBL as well.As expected, I found this in PBL:

> ROM:0010D314 sbl1_partition_id DCD 0xDEA0BA2C        ; SBL1 partition GUID  
> ROM:0010D318                 DCW 0xCBDD  
> ROM:0010D31A                 DCW 0x4805  
> ROM:0010D31C byte_10D31C     DCB 0xB4                ; DATA XREF: find_sbl1_partition+AC↑r  
> ROM:0010D31D                 DCB 0xF9  
> ROM:0010D31E                 DCB 0xF4  
> ROM:0010D31F                 DCB 0x28 ; (  
> ROM:0010D320 unk_10D320      DCB 0x25 ; %            ; DATA XREF: find_sbl1_partition+B4↑r  
> ROM:0010D321                 DCB 0x1C  
> ROM:0010D322                 DCB 0x3E    
> ROM:0010D323                 DCB 0x98  

sbl1's GUID is located at 0x10D314(the names are of course my additions).Set another good GUID for sbl1bak and patching this out, then it should work.(I need to apply common patches like distouching MMU and pagetables according to alephsecurity's research, of course.)

## Update 20/03/25 #3
Okay...after finding out why programmer reboots (in our previous case), I did a workaround in my LK.Then it doesn't reboot anymore, but then stuck as well :( What's worse, it seems to broke PMIC configs so there's something wrong there and my phone can't poweroff anymore! I've had similar issues when porting LK, but I haven't found the root cause and solution.The only way to exit this state is completely by chance, or drain the battery to let it reach cut off voltage, kek.
However, another idea came up to my mind, why not use Alpehsecurity's way, re-run the bootrom ? At first I didn't choose it because I tried to branch to 0x100000 but result in a stuck (the MMC issue).But since I already had a working LK, I can probably uninitialize the MMC properly to a ready state for bootrom.So I had a try.First, I tried to jump to 0x100000 directly from LK. The phone stuck, which is expected as bootrom failed to init MMC.Then I called target_uninit() and platform_uninit() before branching to 0x100000.Then it worked! Phone warm-booted successfully up to stock LK, although it seems stuck at there.Nevermind, as the bootrom is the same, now I can just reuse alpehsecurity's work, but in my LK, of course.

## Update 20/03/25 #2
This time I loaded tz, rpm, and other firmware ELFs with a valid root key hash (but with very wrong image types) in PBL. It either dies or throws errors, no reboot occured.

>Successfully open flash programmer to write: tz.mbn  
>Image load failed with status: 20  

>PblHack: Error - 4  
>Did not receive Sahara hello packet from device  

>Successfully open flash programmer to write: emmc_appsboot.mbn  
>Expecting SAHARA_END_TRANSFER but found: 0  

That's the actual behavior of wrong image type errors in PBL. So the reboot is likely caused by a wrong PBL param, yay! The SBL1 executing routines are actually well documented in the wild.Figuring out what's actually wrong isn't too hard.

## Update 20/03/25
Still haven't figured out booting SBL1 in LK... I don't seem to handle the loading properly.Tried to load the programmer in this way,  stucks as well.   
Since loading on my own doesn't work, and basing on the fact that SBL1 and the programmer are basically the same thing just with some ifdefs at compile time to disable some functions, is it possible to re-use the programmer which still stays in memory? This seems to be a good idea. So I tried to patch the noreturn funtion that enters the programmer's main loop with a return (BX LR), then executing results in a reboot. Expected? The SBL1 as well as the programmer requires a parameter passed from PBL.I always pass the original parameter pointer to the programmer.When I cleared it and execute the programmer (not returning), phone reboots as well. Why not try to boot SBL1 directly with PBL's EDL mode? So I tried , result: reboot. Things are now becoming clearer: It either proves that the reboot is caused by wrong PBL params, or PBL in EDL mode doesn't accept SBL1 image and performed a reboot.The latter is very unlikely, as PBL usually causes an infinite loop instead of triggering a reboot when a fault in secureboot verification.(Btw, the image type of a qualcomm firmware ELF is stored in the signature, and secboot verifies a lot , including the image type.  )

## Update 20/03/23
Another mistake is found in my sbl1 booting routine.When LK jumps to SBL1 entry, it's not actually a jump, but a call, as it's written in C.What's wors it uses BLX, which not only jumps but also switches the bitness. The function calling SBL1 is in arm, thus BLX will switch to thumb mode.However SBL1 entry also has arm code, so this will definitely not work. Now I'm using "LDR PC, =ADDR" as a workaround.

## Update 20/03/22 #2
~~Now I tried to actually boot the loaded SBL1 with the original parameters from PBL.At first I didn't uninit the MMC in LK, and it just spinned forever.Then I uninitialized the MMC right before jumping, and this time a got a reboot.(It's really a reboot, as I verified  the result, with actual SBL1 wiped. )~~  
How silly I am... I forgot to remove the reboot code.Actually uninit the MMC got the same result.

## Update 20/03/22
Added an ELF loader found in originial little kernel project, SBL1 can be loaded properly now.  

## Update 20/03/21 #3
This time I attempted to jump from LK back to EDL, using the same way as the assembly.Also keep the MMU untouched in LK in order to prevent messing up the original tables.It actually worked, but due to the fact that the 9008 serial port role is set by the PBL and EDL programmer doesn't take care of it, we can just see fastboot gadget remains, as if it has crashed.When I re-plugged in the phone, however, 900E shows up.That was expected in EDL mode since EDL programmer doesn't (can't?) initialize the USB.  But later I attepmted to run fastboot commands after booting EDL from LK:


H:\下载\firehorse-master\host>fastboot.exe oem boot-edl  
                                                   (bootloader) Booting to EDL from LK...  
^C  
H:\下载\firehorse-master\host>fastboot.exe oem boot-edl  
                                                   FAILED (Device sent unknown status code: <?xml version="1.0" encoding="UTF-8" ?><data><log value="XML (0 )  
fastboot: error: Command failed  
  
H:\下载\firehorse-master\host>fastboot.exe oem boot-edl  
                                                   FAILED (Write to device failed (Unknown error))  
fastboot: error: Command failed  


An XML reply after all!So this proved that EDL is actually running again.Now at this point we've got a verified-working convenient environment (LK) for us to do more things, e.g. loading ELF binaries, isntead of diving in the dark.

## Update 20/03/21 #2
I tried to re-execute EDL programmer from itself, using the existing pbl2sbl parameter:
>	MOV R0, #0xFFFFFFFF 
>
>	MCR p15,0,R0,c3,c0,0	//Set DACR to 0xFFFFFFFF 
>
>	LDR R0, =0x08003100	//Load existing pbl2sbl as the parameter
>
>	LDR PC, =0x08008B30	//Jump to current EDL's entrypoint 

It "worked", no crashing, not even re-enuming the USB interface.But I'm pretty sure it got executed, as DACR value set by the assembly 
was overwritten. When I tried to read partition from the mmc using firehose then, however, it was stuck and spinning forever(probably).
This means I have the same problem as ugglite, one of alephsecurity's example in their research that "failed to initialize the MMC".
This might be either because the pbl2sbl data isn't correct(should be different each time), or because MMC initialized by the former instance of the EDL programmer isn't uninitialized.

## Update 20/03/21
After days of investigation, I found that loading RPM fw separately seems to be a dead end, since SBL1 is only responsible of loading and authenticating RPM fw , but actual clock setup and reset to the RPM processor is done by QSEE (trustzone), whose source even OEMs can't get hold of.Therefore it isn't likely possible to implement it by ourselves since this is fully undocumented, and there's probably an interdependency between TZ/RPM.
So the next way could be booting SBL1 from our LK.This isn't easy either as we don't have the PBL source(obviously) and don't know the exact requirements SBL1 demands, especially the SBL1 parameters passed from PBL.But another fun fact is that EDL programmer still exists in memory and therefore we can try executing it again (EDL-LK-EDL) for testing :P ,as EDL won't crash even if mmc failed to initialize.
