---
layout: post
title: "Remapping Caps Lock to Esc on Arch Linux"
description: "Remap Caps Lock to be an additional Esc key, and having both Shift keys act as Caps Lock instead"
date: 2021-05-19
tags: [programming, vim, neovim, linux, arch]
comments: false
share: true
---

I've recently switched to a very minimalist Arch Linux setup using BSPWM (from Windows) and it's been quite the learning experience (I might write another blog post about this when I get the time to reflect on things). So far, I'be been enjoying it a lot!

For the past few months I've been using vim (even when I was on Windows, through WSL), however I never remapped my Caps Lock key to act as an Esc as so many people advised me to do.
The other day I decided to give it a shot, here I'm just going to document how I did the remapping (for my own future reference).

I did the remapping using `setxkbmap`. For the available options with "caps", I looked here:

```bash
grep "caps" /usr/share/X11/xkb/rules/xorg.lst
```
Which gave me the following options:

```
[omitted for brevity]
...
caps:swapescape      Swap Esc and Caps Lock
caps:escape          Make Caps Lock an additional Esc
caps:escape_shifted_capslock Make Caps Lock an additional Esc, but Shift + Caps Lock is the regular Caps Lock
caps:backspace       Make Caps Lock an additional Backspace
caps:super           Make Caps Lock an additional Super
caps:hyper           Make Caps Lock an additional Hyper
caps:menu            Make Caps Lock an additional Menu key
caps:numlock         Make Caps Lock an additional Num Lock
caps:ctrl_modifier   Make Caps Lock an additional Ctrl
caps:none            Caps Lock is disabled
compose:caps         Caps Lock
compose:caps-altgr   3rd level of Caps Lock
shift:breaks_caps    Shift cancels Caps Lock
shift:both_capslock  Both Shift together enable Caps Lock
shift:both_capslock_cancel Both Shift together enable Caps Lock; one Shift key disables it
...
[omitted for brevity]

```
The first option I was interested in was `caps:escape`, because I would like my Caps Lock key to act as an **additonal Esc** rather than switch them around. 
The second option I was looking at was `shift:both_capslock`, because I'm one of those people that still uses Caps Lock and this option give me the ability to toggle it by pressing down on both shift keys, perfect!

I use `startx` with `xinit` instead of any login managers, so all I had to do was modify my `.xinitrc` to include the remapping just before I execute my window manager (BSPWM, in my case):

```bash
# Make Caps Lock an additional Esc and both Shift Keys toggle Caps Lock
setxkbmap -option caps:escape,shift:both_capslock &

# Start the window manager (BSPWM)
exec bspwm
```
Now, I can press both shift keys to toggle Caps Lock and Caps Lock acts as an additional Esc (much better for vim ergonomics).

You can check my [dotfiles](https://github.com/n1ghtmare/dotfiles), if you're interested in more details on my setup.

