---
title: "My Path to node.js and Oracle Glory - Volume 2"
date: 2015-03-14T11:25:00-04:00
description: Linux setup on Chromebook
menu:
  sidebar:
    name: Volume 2
    identifier: volume-2
    parent: node-oracle-glory
    weight: 1100
---

OK, so now having established my database server in Volume 1 of this series, I now need to move to the next step - setting up my chromebook with Linux in order to facilitate node.js development.  I chose a chromebook as my dev machine of choice as I wanted something that natively runs on Linux, but didn't want to pay as much as a Mac and didn't really like the Linux based machine options out there...  Google is doing a great job with these and after having my chromebook for a little over a week - I have to say I love it...

The first step is to put your chromebook into developer mode - this is a fairly frightening experience, but since everything on your chromebook is "in the cloud" and can be restored, fear not and hack on!

I bought a Toshiba Chromebook 2 and found instructions to put the machine into developer mode at this [URL][url-1] and this [URL][url-2] as well...

- To put it your chromebook into developer mode, you simply hit the esc+refresh+power keys at the same time after you've logged in.  When you do this, your machine will reboot and show an alert screen - at this point, hit Ctrl+D to go into Developer mode.  **THIS WILL WIPE YOUR MACHINE!**  You'll need to re-login and setup the machine again, but once done you won't need to do this again...  I bought this machine to develop, so this was basically the first thing I did with it - the data wipe didn't concern me...  Since most things you do are auto backed-up to Google drive, this probably won't concern most people.  The process for the laptop to transition to Developer Mode takes 15 minutes or so (one time only) - so go grab a beer...

**Next up - Crouton!**  ...What the hell does a crouton have to do with this?  Well, crouton stands for **C**h**R**omium **O**s **U**niversal chroo**T** envir**ON**ment - and all the information you really need about it is at the GitHub location [here][url-3].  This is now the point where I'm over my head, but I believe crouton is a set of scripts written by a prolific Google developer named David Schneider that allows you to create what's known as a chroot, which allows Linux distributions to run under a segregated file system as a guest OS -  more info about chroots [here][url-4]

Basically, in order to run Ubuntu (the Linux distribution I've chosen) alongside my ChromeOS, I need to run it within a chroot.  I need to use the crouton script bundle to generate this chroot.

You should go to [https://goo.gl/fd3zc][goo-gl1] - it will download into your Downloads folder - leave it there.

In order to properly generate a chroot - you have to have some opinions.  You need to choose a Linux distribution and a desktop environment...

I need to choose a Linux distribution?  I have no idea where to start even - there are over 600 distributions of Linux and 300 are actively being developed!!  Where do you even begin in this world?  My understanding of a distribution of Linux is that it is a packaging of software (libraries, desktop environment, utilities, GUI, etc.).  I have a lot to learn on this, but for now I'm going to be lame and choose a commercially backed option - Ubuntu.  I am choosing the most recent "Long Term Support" (LTS) version of this distro, code named *Trusty Tahr*.

Within Ubuntu, there are different flavors based on the desktop environment you prefer.  After some basic research, I want a basic desktop - I actually installed several different versions and thought the Unity desktop was awful...  So, I'm going with Xfce - a basic, yet comfortable desktop environment!  This flavor of Ubuntu is known as Xubuntu...

I have my opinions settled - I know what I want - next is to go to the Chrome web store and install a few items

1. [Secure Shell][s-shell]
2. [Crosh Window][crosh-url] - opens a separate window from your browser
3. [crouton extension][crouton] - Allows for copy/paste to/from guest OS window (in my case Ubuntu)

You now have all your dependencies - you can either do Ctrl+alt+t to open a "Chrome Operating Shell" or crosh for short in a full screen window or you can use the Crosh Window app you just installed.  

- At the command prompt, type `shell`
- Next type sudo sh ~/Downloads/crouton -t xiwi,xfce -r trusty
  - This is basically telling crouton that the target (-t) is the xfce desktop environment using the xiwi version (which allows for the copy/paste) and the distribution release (-r) should be Trusty Tahr (trusty)
  - This takes a while to install - go get another beer...  At the very end of the install, you'll be asked "Please specify a username for the primary user:" - plug in your preferred username...  you'll then be asked for a password as well...

Once done, you'll be taken back to the shell prompt where you just type **sudo startxfce4** and bam!  Ubuntu should boot!  It will boot in a full screen Chrome OS window - in order to minimize and size the window, just click at the top where it says "crouton integration" triggered full screen **Exit full screen** and then resize as necessary.

Each time you log out of Ubuntu, you want to ensure you logout properly - for the distro I chose, I just go to the upper right hand corner where my username is, choose it and select Log Out...  

Now that I've done this once, the next time is a breeze - open a new Crosh Window, type sudo startxfce4 and that's it - I'm up and running in seconds.

I learned a lot about this crouton extension jazz by reading this - the dude Francois Beaufort seems to be one of the chromebook leads and has a lot of great posts...  You probably want to follow him on Google + if you have a chromebook and are a developer.

OK - it worked - I have Linux up and running within a Chrome OS Window!  Sweet!

## That's it for today - next up Volume 3, Setting up node.js on Ubuntu in my Chromebook

[url-1]:   http://liliputing.com/2014/11/toshiba-chromebook-2-review-bay-trail-and-full-hd.html
[url-2]:   http://www.linux.com/learn/tutorials/795730-how-to-easily-install-ubuntu-on-chromebook-with-crouton
[url-3]:   https://github.com/dnschneid/crouton
[url-4]:   https://github.com/dnschneid/crouton#whats-a-chroot
[goo-gl1]: https://goo.gl/fd3zc
[s-shell]: https://chrome.google.com/webstore/detail/secure-shell/pnhechapfaindjhompbnflcldabbghjo?utm_source=chrome-app-launcher
[crosh-url]: https://chrome.google.com/webstore/detail/crosh-window/nhbmpbdladcchdhkemlojfjdknjadhmh?utm_source=chrome-app-launcher
[crouton]: https://chrome.google.com/webstore/detail/crouton-integration/gcpneefbbnfalgjniomfjknbcgkbijom
