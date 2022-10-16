---
title: "Install fonts in ubuntu."
tags:
- linux
- fonts
created: 2022-10-16
updated: 2022-10-16
---

# How do I install fonts?

Many fonts are packaged for Ubuntu and available via the "Fonts" category of the Ubuntu Software Center. If you prefer `apt-get`, search for packages starting with _otf-_ or _ttf-_.

Font files that are placed in the hidden `.fonts` directory of your home folder will automatically be available (but `/etc/fonts/fonts.conf` indicates it will be removed soon.). You can also place them in the `~/.local/share/fonts` directory on newer versions of Ubuntu per the comments below.

You can also double-click on the font file (or select _Open with Font Viewer_ in the right-click menu). Then click the **Install Font** button.

If you need the fonts to be available system-wide, you'll need to copy them to `/usr/local/share/fonts` and reboot (or manually rebuild the font cache with `fc-cache -f -v`).

You can confirm they are installed correctly by running `fc-list | grep "<name-of-font>"`

***
_fc-cache_ comes with the package _fontconfig_ on latest Ubuntu server versions, so don't forget your `sudo apt-get install fontconfig` if your system cannot find _fc-cache_ And yes, _fc-cache_ also scans `/usr/local/share/fonts`.
***

You may need to restart some programs, like OpenOffice Writer, before they actually show the new fonts (usually such programs are caching the font list when they start up).

_Edit: Changed advice to manually install into `/usr/local/share/fonts` instead of `/usr/share/fonts` to reflect comments and best practice._