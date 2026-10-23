# Relapse: The Browser Bug That Opened the PS5's Kernel

> **Category:** Security · Kernel Exploitation · Console Hacking  
> **Level:** Practitioner  

---

On September 29, a jailbreak called Relapse went public on GitHub, built by a developer going by ntfargo. It chains a memory corruption bug in the PS5's browser engine with a use-after-free race in the kernel, and it works on every firmware from 7.00 to 13.60 — PS5 and PS5 Pro both.

Sony patched the entry point in firmware 14.00, released September 16, two weeks before Relapse shipped publicly. The exploit had clearly existed in private hands before that.

## The Chain

Getting from "visit a web page" to "kernel code execution" on a locked-down console takes two separate bugs, because the browser and the kernel are different trust boundaries.

Stage one lives in JavaScriptCore, the JS engine behind the PS5's browser. A memory corruption bug there gets you arbitrary read/write inside the browser process — enough to build further primitives, not enough to touch the kernel.

Stage two is where the actual escalation happens: a use-after-free race condition in `aiomultiwait`, the kernel's asynchronous I/O wait syscall. Win the race, and a freed kernel object gets reused in a way the attacker controls. That's kernel read/write, full stop — the ceiling for what a modern console's security model is built to prevent.

## A FreeBSD Problem, Not a PlayStation Problem

The PS5 runs on a FreeBSD 11 base. Every major public PS5 kernel exploit since 2022 has come from a FreeBSD bug, not a Sony-written one.

TheFloW's 2022 disclosure used a use-after-free in FreeBSD's IPv6 socket handling — a bug already fixed upstream on the PS4, but still live in PS5 firmware when he found it. CVE-2024-43102, a UMTX_SHM_DESTROY use-after-free, got weaponized into a separate PS5 kernel exploit the same way. `aiomultiwait` is the third entry in that pattern: a bug in the operating system Sony built on top of, not in anything Sony wrote for the console itself.

Shipping a console on a general-purpose OS kernel means inheriting that kernel's entire bug history — including bugs nobody was thinking about PlayStation when they patched them.

## Sony's Countermove

Firmware 14.10, released October 1, doesn't try to force every console onto the new version before it'll boot. Instead, it ties PlayStation Network access to a minimum firmware version. A console still running Relapse keeps working — single-player, offline, no PSN — but the moment it tries to authenticate online, the version check blocks it.

That's enforcement at the service layer instead of the boot layer, and it's a deliberate trade. Sony doesn't control what firmware is physically running on a console someone's deliberately downgraded and exploited. It does control who gets to talk to PSN. Moving the checkpoint to the thing Sony actually owns is a more defensible line than trying to win a fight over code already running on hardware outside their control.

## What's Left

One developer, Gezine, claims to have partially defeated 14.10's protections — posted footage reportedly showing the hidden debug menu — while also saying a fully updated console will need a hypervisor-level exploit to go further. The PS5 runs a hypervisor between the kernel and the hardware specifically to make kernel access alone insufficient. If that claim holds, the next jailbreak isn't a browser-to-kernel chain anymore — it's a third stage on top of the first two.

Separately, a leak of PS5 BootROM keys at the end of 2025 didn't enable Relapse directly, but it's the kind of compromise that outlives any single firmware patch. BootROM is the first thing that runs, the root of the chain of trust everything else depends on. Patch the kernel bug and you've closed Relapse. You can't patch a leaked key.

---

*Related: [doctorglitch — PS5 Relapse Jailbreak Explained](https://doctorglitch.com/ps5-relapse-jailbreak-explained/)*
