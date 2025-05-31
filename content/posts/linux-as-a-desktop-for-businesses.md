---
title: "Linux as a desktop for businesses"
date: 2025-04-03T12:19:32+09:30
tags:
- nixos
---
I've often contemplated why Linux isn't used more as a desktop operating system in business environments. Compared to Windows it's:

- less expensive
  - no licensing costs
  - lower hardware requirements
- more secure
  - fewer linux desktops means less malware developed for the platform
  - open-source meaning security vulnerabilites being fixed aren't at the mercy of slow corporations and instead are fixed by the community
  - no spyware/bloatware pre-installed by default
- more stable
  - linux has a proven track record of being stable and has less breaking changes between updates.
  - rollbacks are also possible and more reliable if an update is botched
  
I could go on but the above highlights the main selling points of Linux.

Like all things though there's always the bad that comes along with the good. Linux is known to be hard, especially if you're not tech-oriented or comfortable with tinkering (and breaking) computer systems. It also doesn't benefit from having the desktop market dominance that Windows has which means application developers often don't ship with Linux support. There are other negatives that come with Linux but these are probably the biggest ones. If these 2x pain points can be overcome Linux could serve as a viable desktop replacment for businesses. 

The first point how Linux is difficult can be solved by good Linux engineers. A good Linux engineer can build a desktop system that is intuitive and easy to use as well as upgrade and fix the system when things go wrong. In practice this isn't too difficult but to do so at scale in a production environment can be challenging (especially if you're a one man band). This problem however can be solved with a system like NixOS. NixOS is a system that is defined entirely in code and once configured the system has no side effects due to the functional language that it's programmed in (Nix).

To the non-technical readers I might lose you here but essentially NixOS is a system that takes great deliberate effort to always produce the same system state given a code file. It does this in a way that makes the resulting system configuration pure with no side effects. This is in stark contrast to typical systems like Windows where changes are made to the system incrementally in an uncontrolled manner leading to systems which eventually become an unrecognizable bloated mess.

The second point where Linux is lacking is application availability. Graphically intensive applications likke Photoshop for example don't ship for Linux. This is a reality that won't change anytime soon so if a workflow requires any graphically-heavy tasks you're out of luck. Microsoft Office also doesn't run on Linux either but they're are open source alternatives like Libre Office which make a viable replacement. Getting end-users to adapt to this change though can be challenging.

If however you're workflow only requires a web browser and an open source Office suite then Linux can be a very promising solution for you.
