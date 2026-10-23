# PHP2Shell: 72 Hours of Internet-Wide Panic

> **Category:** Security · Vulnerability Management · Incident Response  
> **Level:** Foundational  

---

In late October 2026, two critical bugs in the PHP engine itself — not in any application, framework or plugin running on top of it — put an unknown but very large slice of the web on alert. The security community nicknamed the pair "PHP2Shell": **CVE-2026-109618** and **CVE-2026-109619**, both scored 9.2 on CVSS v4.0, both unauthenticated, both remote code execution.

This isn't a writeup of a specific incident I worked. It's a look at the public chaos of those few days — the leaked disclosure, the scramble of advisories, the mitigation that only half-worked, and what the honeypot data actually showed once the dust settled a little.

## Why This One Was Different

Most RCEs people deal with live in application code: a deserialization flaw in a library, a path traversal in an upload handler, a bad regex in an auth check. You can usually trace the bug to a decision someone made in a specific codebase.

PHP2Shell lives in `session.upload_progress` — a feature of the PHP engine itself, present since PHP 5.4. That means an application with genuinely clean code, properly validated input and no known vulnerable dependencies could still be exploitable, because the bug sits below all of that, in the runtime interpreting the request. Reporting at the time specifically called out e-commerce platforms like Magento, Shopware and WooCommerce as exposed *regardless of their own code quality* — because the flaw triggers before application logic ever runs, on some configurations.

## Two Bugs, Two Different Memory Problems

**CVE-2026-109618** is an out-of-memory corruption issue. When an OOM condition interrupts a vulnerable operation mid-flight, PHP can leave session data pointing at memory that has already been freed — and in some conditions, that can leak memory addresses, which is exactly what you need to defeat ASLR as a stepping stone toward full RCE.

**CVE-2026-109619** is a different animal: a stale global state bug combining a fatal-error-backtrace use-after-free with OPcache type confusion. A later request handled by the same worker process can end up reusing memory that should have been freed, with a fatal-error-backtrace path that specifically applies to PHP 8.5.

Different root causes, different code paths — which turned out to matter a lot once people started patching.

## Disclosed Early, Patched Fast

The details reportedly leaked on October 20, ahead of the coordinated disclosure date — which forced PHP to move the official release up from October 22. Fixed versions shipped October 21: **8.5.12, 8.4.27, 8.3.36, 8.2.35**. Backports were reported for 8.1, 8.0 and 7.4, but anything older — including the long tail of PHP 5.x installs still quietly running somewhere — was left with no official fix at all.

That compressed timeline is exactly what turns a critical bug into a scramble: less time to coordinate messaging, less time to test patches at scale, and a disclosure that outran the people who were supposed to control its pace.

## The Mitigation That Only Fixed Half the Problem

This is the part that made the following days genuinely chaotic for anyone trying to respond fast. The most widely circulated interim mitigation was setting `session.upload_progress.enabled=0` in `php.ini` and restarting all workers.

That stopgap addresses CVE-2026-109619. It does not fully address CVE-2026-109618 — by some accounts, the advisory for 109618 doesn't even mention sessions as the trigger path. Two bugs sharing a nickname and a disclosure date, needing two different stopgaps, got flattened in a lot of first-pass guidance into "turn off upload_progress and you're covered." Teams that did exactly that and called it done were still carrying exposure from the other half of PHP2Shell.

This is a pattern worth internalizing on its own: when a vulnerability gets a catchy combined name, it's tempting to treat it as one fix. Read past the headline into the actual advisories before declaring victory.

## The Industry Scramble

Within roughly 72 hours of disclosure, advisories and technical breakdowns went out from watchTowr, Imperva, Qualys, Arctic Wolf, Akamai, SOCRadar, Unit 42, JFrog, Bitdefender, Ellio and Upwind, among others — plus e-commerce-focused shops like sansec zeroing in on Magento/Shopware exposure specifically. That's not an unusual response to a critical CVE, but the volume and speed of it is a reasonable proxy for how seriously the industry took the "engine bug, not app bug" angle.

Public proof-of-concept repositories also appeared quickly on GitHub, which is its own double-edged part of this cycle: useful for defenders validating exposure, and a head start for anyone less well-intentioned.

## What the Honeypots Actually Saw

PHP's own security team reportedly said, at release time, that no functional exploit was known to exist yet. That didn't hold for long. watchTowr's honeypot network reported real in-the-wild scanning activity, and one analysis clocked 33 distinct source addresses sending 743 requests carrying the session upload progress field by 23:00 UTC on the day activity picked up. Some of that traffic was tentatively linked by researchers to Potassium, a Mirai-derived botnet — though without confirmed evidence that payloads were actually downloaded or executed from those sessions.

The gap between "no known functional exploit" and "internet-wide opportunistic scanning within hours" is a useful reminder of what a CVSS 9.2 pre-auth RCE actually means in practice: you don't need a public working exploit for mass scanning to start. Automated probing for the vulnerable surface is cheap, and it starts the moment a CVE ID and a rough description exist.

## The Actual Takeaways

- **Engine-level bugs break the "our code is fine" assumption.** Clean application code is not a defense against a vulnerability in the runtime underneath it.
- **A combined nickname can hide two different fixes.** PHP2Shell was two CVEs with two root causes and two separate stopgaps. Treat each advisory on its own terms, even when marketing (including security marketing) wants a single tidy name.
- **Patch, don't just mitigate.** The interim config flags bought time; they were never a substitute for getting to a fixed PHP version, and for 5.x/7.x/8.0/8.1 installs, "getting to a fixed version" wasn't even an option without a platform upgrade.
- **Scanning starts before exploitation is proven.** Honeypot data during this window is a good case study for why "no known exploit yet" is not a reason to deprioritize patching a 9.2.
- **A fifteen-year-old feature was the attack surface.** `session.upload_progress` shipped in PHP 5.4. Old, boring, rarely-thought-about features are exactly where this kind of bug survives undisturbed for over a decade.

---

*Sources: [Sansec](https://sansec.io/research/php2shell), [Bitdefender](https://www.bitdefender.com/en-us/blog/businessinsights/technical-advisory-php2shell-pre-auth-rce), [SOCRadar](https://socradar.io/blog/php2shell-cve-2026-109618-cve-2026-109619/), [Ellio](https://ellio.tech/en/blog/php2shell-what-we-observed-and-what-to-do/), [Upwind](https://www.upwind.io/feed/initial-assessment-php2shell-two-critical-unpatched-php-rces-cve-2026-109618-cve-2026-109619), [watchTowr](https://watchtowr.com/intelligence/php-remote-code-execution-php2shell/), [Imperva](https://www.imperva.com/blog/php2shell-vulnerability-updates/), [Qualys](https://blog.qualys.com/vulnerabilities-threat-research/2026/10/20/php-announces-critical-security-release-for-two-vulnerabilities), [Arctic Wolf](https://arcticwolf.com/resources/blog/php2shell-critical-php-vulnerabilities-enable-unauthenticated-remote-code-execution/), [Akamai](https://www.akamai.com/blog/security-research/understanding-php2shell-everything-you-need-to-know), [Unit 42](https://unit42.paloaltonetworks.com/cve-2026-109618-cve-2026-109619/), [JFrog](https://research.jfrog.com/post/php2shell-all-you-need-to-know-about-cve-2026-109618-and-cve-2026-109619/). Verify CVE details against the official NVD/PHP advisories before acting on them operationally.*
