# Windows dual boot


## Hardware clock

Most Linux distributions set the hardware clock to UTC.
Windows sets it to the configured timezone.
This leads to problems when dual booting:
After booting the other OS the time will be wrong until the system is connected to the internet and synchronized the time.

This can be fixed by telling Windows to use UTC.
The opposite would also be possible, but this is easier.

1. Press Win + r
2. Run `regedit`
3. Confirm UAC
4. Navigate to `HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\TimeZoneInformation\`
5. Create the new DWORD `RealTimeIsUniversal` and set the value to `1`

At the next boot the problem should be fixed.


## Windows file access from Linux

Windows cannot easily read Linux file systems.
But the other way around is possible.
At least when windows shuts down completely and leaves the file in a clean state.
Nowadays, if you press *Shut down* Windows will not actually shutdown, but does more a kind of suspend-to-disk, so it is faster on startup.

You can disable that behaviour:

1. Press Win + r
2. Run `powercfg.cpl`
3. In the left column click on *Choose what the power buttons do*
4. Click on *Change settings that are currently unavailable*
5. Remove the checkbox for *Turn on fast startup*

This will maybe cost you a few second when booting Windows, but now you can access the files from Linux.
