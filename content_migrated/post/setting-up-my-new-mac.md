+++
author = "Jakob Højgaard"
categories = ["Boot Camp", "OS X", "Windows 8.1"]
date = 0001-01-01T00:00:00Z
description = ""
draft = false
slug = "setting-up-my-new-mac"
tags = ["Boot Camp", "OS X", "Windows 8.1"]
title = "Installing Windows 8.1 on my mac"

+++

I recently got a new machine. And since i do a fair bit of iOS development (through Xamarin) I chose to get a mac. However, since I still do most of my work on Windows I would like to be able to boot it directly into Windows. No problem I thought, mac has Bootcamp. I'll just install the newest version of Windows using the Bootcamp assistant. Well everything went well until the Windows installation had to update the boot record. I popped a message box stating that "Could not update the boot configuration". No way to continue!

After quite a bit og googling and reading I found that I found a working solution [here](https://discussions.apple.com/thread/5474320?start=585&tstart=0) (on page 40!). It's posted by a guy called **KNNSpeed**. For convenience i'll republish the solution here:

> 1) Using an .iso of Windows 8.1 I bought from the Microsoft Online Store, I copied the .iso to my rMBP and pointed Bootcamp to it to make the USB installer. The image created is slightly over 4GB, so it's important to use an 8GB flashdrive. There have been reports of issues caused by using 4GB flashdrives, so 8GB or larger is the way to go.

> 2) After letting the Boot Camp Assistant do its thing (I personally use a 50/50 split, but more importantly I checked all 3 checkboxes) and entering my password to install the helper tool (if you don't use a password you probably won't see the helper tool prompt), I rebooted and went right into the EFI Boot option in the boot manager (the hold-down-ALT-at-bootup menu).

> 3) In Windows 8.1's setup, I got an error about installing over the BOOTCAMP partition. No problem, if we're installing Windows8.1 in native EFI mode, we can't use a single partition anyway since Windows needs to make an extra MSR partition. It can't do that when BOOTCAMP fills up all the empty space! Solution: delete that BOOTCAMP partition right there in Windows setup, and click "New." You should be able to make an NTFS partition that will fit right into the empty space you just made, and you'll know you're on the right track because there'll be a second partition labelled MSR that gets created, too.

> 4) Continue to install Windows as normal in the NTFS partition you just made (the non-MSR one). This is why I used the USB method; DVDs are so slow when doing Windows setup. Anyways, you should get the classic "Could not update the boot configuration" error at the end. DON'T PANIC, the next step fixes that.

> 5) Click OK or Continue or whatever it says (it's been a while), and wait for your MacBook Pro to reboot, and go to the ALT menu. Now turn off your computer (or boot into OS X and shut it down, whatever--just turn it off without doing anything), turn it back on and do a PRAM/NVRAM reset (Hold Command+Option+P+R until you hear the chime play a second time), and go back to the ALT menu. Try not to miss the menu and boot into OS X, since I don't know if that'll mess anything up (it might, and you'll have to NVRAM reset again). Go right back into EFI Boot and proceed with Windows setup as normal.

> 6) Windows should install with no errors, and when you reboot you now need to go right into the "Windows" drive in the ALT boot menu, as you're done with booting from the flashdrive. Also, it's OK if you go into OS X by accident now--just don't remove the flashdrive yet. If Windows reboots during this part of the installation, just use the ALT boot menu to go back into the Windows drive (remember: this isn't the "Windows" flashdrive icon--it's the Windows hard drive icon).

> 7) Once Windows is done installing, it'll ask you to install the Bootcamp drivers. Go ahead and do that and reboot into Windows. Now go into "This PC" or "Computer" or whatever Microsoft renamed "My Computer" to and run "Setup.exe" from the flashdrive's WindowsSupport folder to install the Bootcamp drivers again. Apparently the setup that runs earlier misses a few things for some reason.
