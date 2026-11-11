# Linux Wasn't Ready for This

> **Category:** Linux · Open Source · AI  
> **Level:** Foundational

---

In 2024, the Linux kernel's private security list received two or three bug reports a week. By March 2026, according to Willy Tarreau, the pace was already between five and ten a day. The interesting part is not the volume itself, it is the hit rate: most of those reports are valid. And that is exactly where the problem lies.

AI tools have collapsed the cost of finding a bug. The cost of fixing it, testing it, reviewing it and taking responsibility for the fix has not changed, and it falls on a small group of maintainers who look after the software behind most of the internet's servers, Android and nearly every supercomputer. That mismatch is what this post is about.

## From Noise to a Serious Tool

Until recently, an AI-generated report meant **AI slop**: convincing text about bugs that did not exist, which did not survive the first test. Easy to spot and even easier to discard.

According to the maintainers themselves, that changed abruptly early this year, over a span of weeks, with no clear cause. The models got better and AI-assisted reports started pointing at real problems.

**Greg Kroah-Hartman** gave a concrete example: a simple request to one of these tools returned **60 issues, each with a proposed fix**. Roughly a third were wrong. The other two thirds held up, even if they needed human adjustment. That is a meaningful error rate (the best models still fail somewhere between 25% and 33% of the time, without showing any doubt), but for people who have always done this work by hand, the gain in scale is enormous.

**Daniel Stenberg**, maintainer of curl, describes the same transition: the useless reports have practically disappeared, replaced by something harder to manage, high volume with high quality. In other words, the difficulty was never AI being wrong. It started when AI began to be right.

## Duplicates at Scale

The intuitive conclusion would be: more bugs found, more secure kernel. In practice, what reaches the list is repetition. The same defect in the networking subsystem shows up three, four times, described in different words, because several people ran similar tools (often the same one) against the same code.

**Willy Tarreau**, who helps triage the list, described exactly that scenario: something new every day and a growing volume of duplicates. In May, **Linus Torvalds** used a release-candidate announcement to note that the security list had become practically unmanageable, flooded with repeated reports sharing the same root cause.

The CVE count reflects it. Linux 7.0 went past 1,000 CVEs, 7.2 past 1,500, and the per-release count is now approaching 2,000, and automated AI-driven bug hunting is cited as one of the factors.

There is also an imbalance of incentives. Finding vulnerabilities became a showcase: AI companies promote scanners that sweep open source projects, and every finding becomes marketing material. The fix, on the other hand, is a cost, and the one who bears it is not who found the bug but who has to validate, fix and maintain it.

## Torvalds Is Not Against AI

A rejection from the project would have been the natural reaction. That is not what happened.

About two months after that complaint, the community split over an automated patch review bot. Some developers wanted the right to refuse machine reviews. One went as far as arguing that there is no ethical justification for using generative AI in free software.

Torvalds' answer was categorical. As top-level maintainer, he made it clear that Linux is not an anti-AI project, and that anyone who disagrees has the exit open source itself provides: fork it or leave. He acknowledged AI can be a painful tool for maintainers, partly because it keeps finding embarrassing bugs, but noted nobody is required to use it. And he added, in a line that got a lot of attention, that AI has flaws, but natural intelligence is not always all that great either.

There is no contradiction with the May complaint. The criticism was never about the tool, it was about how it was being used. In that same message he asked anyone who wanted to contribute to read the documentation, send the fix along with the report and add real value on top of what the AI produced. The friction is short-term. In the long run, a bug found and fixed is a gain.

Torvalds uses these tools himself. Weeks later, while debugging an Intel Xe GPU driver problem (24 debug patches and 18 boots later, the fix was turning a `round_up()` into a `round_down()`), he reported that AI did a good part of the heavy lifting, despite claiming more than once that the problem was impossible.

The friction, then, is not in the technology. It is in the bottlenecks and the human strain that such an effective tool creates.

## The Embargo Lost Its Purpose

Responsible disclosure has always followed a familiar script: whoever finds a serious flaw tells the owner privately, waits for the fix and only then makes it public. The kernel's private list exists because of that model.

The model rests on one premise, that the flaw is known to few. With AI, that premise does not hold: if one person found the bug with a tool, it is reasonable to assume dozens of others found the same one.

The kernel made it official. The documentation now says that bugs found with the help of AI should be treated as public, with one important caveat: exploitation details must not be disclosed. The report states that the problem exists and the maintainer asks for more information privately if needed. Since more than 90% of what arrives is not critical, low and medium severity issues are now fixed directly in Git, without highlighting the security angle.

It is a big cultural shift. The secrecy that organized vulnerability disclosure for decades is being abandoned, not by choice, but because it stopped being possible.

## Cutting to Survive

With more reports than review capacity, the kernel networking team chose to shrink the surface. The tree includes drivers for 1990s network cards, amateur radio, ATM and ISDN support: technologies with very few users, but which AI tools analyze as rigorously as the rest of the code, finding bugs all the same.

**Jakub Kicinski**, one of the networking maintainers, was straightforward: there is no capacity to review every AI-generated fix, and ignoring security fixes is not an alternative. The proposal was to remove the code: a pull request dropping old drivers and entire subsystems such as ISDN, AX.25 (amateur radio), CAIF and ATM, about **138,000 lines** in total. The rationale for dropping amateur radio support explicitly mentioned the lack of volunteers to handle the volume of AI-generated reports.

Months later, while assembling the networking changes for Linux 7.3, Kicinski estimated that between a third and half of what landed in net-next was AI-driven. In the stats for the previous cycle he had already noted that maintainers were spending around half their time trying to disprove AI reviews.

The outcome is counterintuitive: a technology that should make the kernel more secure is leading to parts of it being removed. The old code worked and bothered nobody. It became a problem once it started being analyzed relentlessly.

## Banning Doesn't Solve It

Some projects tried the ban route. **QEMU**, widely used for virtualization, prohibited AI-assisted contributions. A year later, one of its main developers suggested relaxing the rule, arguing that as the tools evolved, a total ban had become hard to defend. The proposal is still under discussion and the ban remains in force.

Torvalds explicitly rejects that approach. He said he will very loudly ignore people who try to stop others from using these tools, and that the answer to AI's downsides is not to bury your head in the sand and sing "la la la, I can't hear you".

There is also a telling counterpoint: the networking team itself, the most affected by the volume, already runs AI review bots on every posted patch and, according to the 7.3 networking pull request, has access to several frontier models, credited to Meta. The same technology that creates the overload helps absorb it. Torvalds' position sums it up well: the goal is for these tools to work for maintainers, not only against them.

## What AI Cannot Sign

One question remains: if AI finds bugs and writes fixes, could it maintain the kernel on its own?

The BPF maintainers, responsible for one of the kernel's most sensitive subsystems, posted a notice titled "summer vacation": several of them would be away at the same time, and more bugs would be marked as won't fix. **Alexei Starovoitov** criticized patches where someone just forwards an AI-found bug and an AI-written fix with little checking. AI is already good at finding problems, but not yet reliable at fixing them unsupervised. In his assessment, applying a hundred fixes for a hundred bugs that would never happen in production is no reason to celebrate.

On the security list, Tarreau described something similar: at the current volume, the work is limited to triage, separating what is urgent from what can wait.

This is where the real limit shows up. Every change accepted into the kernel carries a `Signed-off-by` line identifying who answers for it, and the project's documentation is explicit: an AI agent cannot sign. Only a person can attest, legally included, to where the code came from.

That means however many fixes the tools produce, each one depends on someone who reads it, understands it and accepts responsibility for it. And that group does not grow at the same pace. Companies hire kernel developers to build features, rarely for maintenance, which ends up happening largely in people's free time. It is a barely sustainable model, and it does not attract new maintainers. People take vacations, burn out and remove old code to keep up with the rest.

AI is not breaking Linux. It is making it obvious that a central piece of the internet's infrastructure depends on the time and attention of a group of people too small for the responsibility it carries. The scarce resource in the kernel today is not code. It is human attention.

---

*Related: [Tom's Hardware: Torvalds says AI bug reports made the security list almost entirely unmanageable](https://www.tomshardware.com/software/linux/linus-torvalds-says-ai-bug-reports-have-made-the-linux-security-mailing-list-almost-entirely-unmanageable) · [The Register: Linux kernel czar says AI bug reports aren't slop anymore](https://www.theregister.com/software/2026/03/26/linux-kernel-czar-says-ai-bug-reports-arent-slop-anymore/5226256) · [The Register: AI slop got better, so now maintainers have more work](https://www.theregister.com/software/2026/04/06/ai-slop-got-better-so-now-maintainers-have-more-work/5223172) · [Tom's Hardware: Linux kernel nears 2,000 CVEs per release](https://www.tomshardware.com/software/linux/linux-kernel-nears-2-000-cves-per-release-as-ai-bug-hunters-scour-40-million-lines-of-code-maintainers-say-they-are-completely-overwhelmed) · [The Register: Linus Torvalds tells AI haters to fork off](https://www.theregister.com/ai-and-ml/2026/07/15/linus-torvalds-tells-ai-haters-to-fork-off/5271894) · [ADTmag: Torvalds tells kernel critics AI isn't going anywhere](https://adtmag.com/articles/2026/07/28/torvalds-tells-linux-kernel-critics-ai-is-not-going-anywhere.aspx) · [PC Gamer: Torvalds' debug session from hell](https://www.pcgamer.com/hardware/linus-torvalds-details-his-debug-session-from-hell-pushing-an-ai-to-fix-an-error-it-repeatedly-said-was-impossible-and-unsolvable-he-then-made-it-write-the-report/) · [kernel.org: Reporting security bugs](https://docs.kernel.org/process/security-bugs.html) · [kernel.org: AI Coding Assistants](https://docs.kernel.org/process/coding-assistants.html) · [Phoronix: Pull request to remove old network drivers and ISDN due to AI/LLM noise](https://www.phoronix.com/news/Linux-7.1-PR-Remove-Old-Net) · [LWN: Kernel code removals driven by LLM-created security reports](https://lwn.net/Articles/1068928/) · [LWN: netdev development stats for 7.1](https://lwn.net/Articles/1067741/) · [BPF list: Summer vacations and "Won't Fix" bugs](https://ratatoskr.run/bpf/2026/07/17206127) · [The Register: QEMU mulls relaxing AI contribution ban](https://www.theregister.com/ai-and-ml/2026/05/29/qemu-mulls-relaxing-ai-contribution-ban/5248638)*
