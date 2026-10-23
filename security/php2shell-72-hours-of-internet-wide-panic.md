# PHP2Shell: 72 Hours of Internet-Wide Panic

> **Category:** Security · Vulnerability Management · Incident Response  
> **Level:** Foundational  

---

Two critical bugs in the PHP engine itself went public this week. **CVE-2026-109618** and **CVE-2026-109619** — nicknamed "PHP2Shell" — are both unauthenticated remote code execution, both scored 9.2 on CVSS v4.0, and neither one lives in application code. They live in the engine every PHP app runs on top of.

The details leaked on October 20, ahead of the coordinated disclosure date. PHP moved the official release up two days and shipped patches on October 21: 8.5.12, 8.4.27, 8.3.36, 8.2.35. Anything older than 8.2 has no official fix.

## Why Clean Code Didn't Help

Most RCEs trace back to a decision made in a specific codebase: a deserialization flaw, a path traversal, a bad regex in an auth check. PHP2Shell doesn't. It lives in `session.upload_progress`, a feature of the engine itself, present since PHP 5.4. An application with properly validated input and no vulnerable dependencies could still be exploitable, because the bug sits below all of that — in the runtime interpreting the request, before application logic even runs on some configurations. Reporting this week called out Magento, Shopware and WooCommerce stores as exposed regardless of their own code quality.

## Two Bugs, Two Different Memory Problems

**CVE-2026-109618** is an out-of-memory corruption bug. When an OOM condition interrupts a vulnerable operation mid-flight, PHP can leave session data pointing at memory that's already been freed — which can leak addresses, enough to defeat ASLR as a step toward full RCE.

**CVE-2026-109619** is a different bug: stale global state, combining a fatal-error-backtrace use-after-free with OPcache type confusion. A later request on the same worker can reuse memory that should have been freed. The fatal-error-backtrace path applies specifically to PHP 8.5.

## The Mitigation That Covered Half the Problem

The most widely circulated interim fix was setting `session.upload_progress.enabled=0` and restarting all workers. That addresses 109619. It does not fully address 109618 — the advisory for 109618 doesn't even mention sessions as the trigger path.

Two bugs, one nickname, one disclosure date, two different stopgaps — and a lot of first-pass guidance flattened that into "turn off upload_progress and you're covered." Teams that stopped there were still carrying half the exposure.

## What the Honeypots Saw

PHP's own security team said, at release, that no functional exploit was known yet. That didn't hold. watchTowr's honeypot network picked up scanning within hours — 33 source addresses, 743 requests carrying the session upload progress field by 23:00 UTC the same day. Some of that traffic was tentatively linked to Potassium, a Mirai-derived botnet, though without confirmed payload execution.

A CVSS 9.2 pre-auth RCE doesn't need a public working exploit for mass scanning to start. The moment a CVE ID and a rough description exist, automated probing is already cheap enough to run at internet scale.

## Old, Boring, Unpatched

`session.upload_progress` has existed since 2012. Nobody thinks about it, nobody disables it by default, and almost nobody audits a feature that's been quietly sitting in the engine for fourteen years. That's exactly where this kind of bug survives — not in new code getting daily scrutiny, but in old code nobody's looked at since it shipped.

---

*Related: [watchTowr — PHP2Shell RCE Exploited](https://watchtowr.com/intelligence/php-remote-code-execution-php2shell/)*
