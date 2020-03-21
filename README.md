# firehorse_land
This is a fun project utilizing "peek and poke" function in Redmi 3S's EDL programmer,
which leads to [arbitary code execution](https://alephsecurity.com/2018/01/22/qualcomm-edl-1/) in highest privilege level of its Application Processor.

The ultimate goal of this project is to (re)load the rest of the boot chain from the LK, bypassing Secure Boot,
and use backup partitons (*bak) on the EMMC to store tampered firmwares and boot them in EDL while keeping the stock firmware,
making it a Semi-Tethered jailbreak.

## Current status:
Supports booting from EDL to a custom LK(little kernel, aboot),[source](https://github.com/fxsheep/lk4edl).
Nothing much is working at this point, as RPM firmware isn't loaded(yet).
### Working:
USB
### Not working:
Vibrator, Display, etc.

## Usage(use at your own risk)
1. Download [firehorse research framework](https://github.com/alephsecurity/firehorse)
2. Download this repo and place all files under firehorse\host.(firehorse is the repo you downloaded in 1)
3. Setup firehorse framework properly ,you just need to install QPST, Python2.7 and configure constants.py.
4. Edit firehorse\host\exploit.cmd , modify port variable according to your EDL port number.
5. Put your Redmi 3S in normal mode, check if adb is working.
6. Run exploit.cmd , wait for the exploit to run.
7. Your device will enter EDL mode and the custom LK will be loaded and executed in less than a minute.
8. You'll see the 9008 serial port disappears and a Fastboot device shows up.
9. Use fastboot oem lk_log to see LK boot messages.

## Update 20/03/21
After days of investigation, I found that loading RPM fw separately seems to be a dead end, since SBL1 is only responsible of loading and authenticating RPM fw , but actual clock setup and reset to the RPM processor is done by QSEE (trustzone), whose source even OEMs can't get hold of.Therefore it isn't likely possible to implement it by ourselves since this is fully undocumented, and there's probably an interdependency between TZ/RPM.
So the next way could be booting SBL1 from our LK.This isn't easy either as we don't have the PBL source(obviously) and don't know the exact requirements SBL1 demands, especially the SBL1 parameters passed from PBL.But another fun fact is that EDL programmer still exists in memory and therefore we can try executing it again (EDL-LK-EDL) for testing :P ,as EDL won't crash even if mmc failed to initialize.

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
