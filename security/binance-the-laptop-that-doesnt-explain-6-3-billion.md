# Binance: The Laptop That Doesn't Explain $6.3 Billion

> **Category:** Security · Incident Response · Digital Assets
> **Level:** Foundational

---

Between 20:04 and 23:56 UTC on October 29, 2026, Binance's own withdrawal system signed and released **$6.3 billion** from **17 hot wallets**, spread across **9 blockchains**, to an account that belonged to no customer. The largest cryptocurrency theft ever recorded — bigger than Bybit (~$1.5 billion, February 2025), bigger than Coincheck, bigger than anything before it. In under four hours, a meaningful slice of one of the world's largest exchanges simply walked out the door.

Binance's first explanation didn't last long. And that's exactly where this case gets interesting for anyone who works in security: this isn't the story of a leaked private key, it's the story of an explanation that kept shrinking as the investigation went on.

## The Explanation That Didn't Hold

In the first hours, Binance's version was straightforward: a **compromised DevOps engineer's laptop** had given an outside actor access to an internal service container, allowing unauthorized withdrawal transactions to be submitted. A clean story, with a specific culprit (a device), a specific cause (phishing or malware on a laptop), and an obvious lesson (lock down the endpoints of anyone with privileged access).

That version didn't hold for long. Co-CEO **Richard Teng** publicly walked it back, saying the attack involved a "more complex" process than the misuse of a single laptop's permissions. Binance confirmed it had identified **"additional points of breach"** beyond the original laptop — without detailing what they were. As of this writing, the company hasn't disclosed how the laptop was compromised in the first place, nor confirmed whether the affected wallets used MPC, multisig, or another signing scheme. A full technical post-mortem and an updated proof of reserves were promised, but haven't shipped yet.

This isn't unusual — it's the pattern. The first explanation of any major incident is almost always the simplest one to tell and the least complete one to be true, because it ships before the forensic investigation is done.

## What Actually Happened

Technical reconstructions from TRM Labs and Elliptic describe the core mechanism: a single transaction bundled **50 pre-signed EIP-7702 authorizations** (Ethereum's standard letting a regular account — an EOA — temporarily behave like a smart contract), redirecting a Binance hot wallet to a **"sweeper" contract** — a contract built to do exactly one thing: sweep a wallet's balance the moment it's pointed at it.

The critical detail: the withdrawal transactions carried **valid signatures from Binance's own hot-wallet infrastructure**. No private key leaked — Binance was categorical about that. What happened is that the attacker, having reached the service container through the compromised laptop, got the **legitimate signing system to approve transactions that should never have been approved**. The validation path that was supposed to block those withdrawals simply didn't.

That's an important distinction, and it's the kind of detail that separates "the key leaked" from "the process that decides what gets signed was fooled" — two completely different problems, with completely different defenses.

## MPC Protects the Key, Not the Decision

This incident exposed a point custody analysts (Fireblocks published an analysis on October 30) had mostly been making in the abstract until now: **MPC (Multi-Party Computation) protects key material, but it doesn't protect the orchestration layer that decides when and what gets signed**.

Splitting a private key across multiple parties, so no single party can reconstruct it alone, is a real defense against outright key theft. But if the system that instructs the MPC nodes what to sign is compromised — and that's exactly the system that says "sign this withdrawal transaction" — the key never has to be stolen. It does exactly what it was designed to do: sign what it was told to sign. The problem isn't the cryptography, it's the decision layer wrapped around it.

Fireblocks pointed to the real defenses against this kind of scenario: granular policy controls, transaction simulation before signing (predicting the on-chain outcome before approving), and delegation restrictions — exactly the controls that, apparently, didn't catch 50 EIP-7702 authorizations going out at once to an unknown contract.

## The Parallel With Bybit

This pattern — the cryptographic protection working exactly as designed, and the attack happening at the layer that decides or displays what's being signed — isn't new. It's structurally almost identical to what happened to **Bybit in February 2025** (~$1.5 billion, mostly in ETH).

At Bybit, the vector was different: malicious JavaScript injected into the Safe{Wallet} UI through a compromised developer machine (some reports point to phished AWS credentials as the entry point). The multisig was 2-of-3. All three signers saw, on screen, a legitimate transaction — correct destination address, correct URL, everything checked out. But the payload that actually reached the hardware wallets for signing was different: a delegatecall swapping the Safe contract's masterCopy for a backdoored version. The signers signed exactly what they saw on screen. The problem is the screen was lying.

Bybit and Binance are, at bottom, the same lesson told two different ways: **multisig and MPC protect against someone stealing or forging a signature, not against the system around them being fooled into requesting or displaying the wrong one**. In both cases, the signing mechanism worked perfectly — it signed exactly what it was instructed to sign. The failure was entirely in whoever, or whatever, decided what would be instructed.

## Who's Behind It

Attribution here followed a confidence hierarchy worth noting. First came the blockchain intelligence firms: **TRM Labs** (with what they call high confidence, citing on-chain and off-chain intelligence), **Chainalysis**, and independent investigator **ZachXBT**, all pointing to **TraderTraitor** — the Lazarus Group unit linked to North Korea. TRM cited specific features matching the group's earlier attacks: the same small Tornado Cash "staging" transactions well before the main heist, and fund co-mingling linked to other attacks already attributed to them.

On **November 1**, the **FBI officially confirmed** the attribution to TraderTraitor, expecting the stolen assets to be laundered through DeFi and instant swap services, and noting ongoing conversion into Ethereum. The FBI also directly tied this attack to the group behind Bybit — same actor, similar tradecraft, nearly two years apart.

Worth noting the caveat: not all public attribution is aligned. The US president publicly stated "we don't know if it's North Korea," even after the FBI's confirmation — a reminder that geopolitical attribution of cyberattacks is rarely unanimous, even when technical and intelligence agencies converge.

## The Race to Contain the Damage

Containment moved fast by industry standards. **Tether** froze more than $1.5 billion, **Circle** more than $800 million — between the two, and with other partners, roughly **$3 billion** of the withdrawn funds were already frozen by the morning after the attack (October 30). Binance announced it had **replaced roughly 90% of the stolen amount with company funds**, publicly guaranteeing users wouldn't bear the loss. Withdrawals resumed in phases starting **November 1**.

Market impact was immediate but contained: bitcoin dipped to a six-week low of around **$76,368** before partially recovering.

## What's Still Unknown

Weighing what's confirmed against what's still open:

**Confirmed:** the approximate amount (~$6.3 billion, with some variation between sources), the time window (20:04–23:56 UTC, Oct 29), the number of wallets and blockchains affected, the on-chain mechanism (50 EIP-7702 authorizations redirecting to a sweeper), that the signatures were valid and no private key leaked, the FBI's attribution to TraderTraitor, and the containment response (freezes, 90% replacement, resumed withdrawals).

**Not yet publicly confirmed:** how the engineer's laptop was originally compromised, what exactly the "additional points of breach" Binance mentioned were, whether the affected wallets used MPC, multisig, or another scheme, and the final exact amount stolen (estimates still range between $5.9 and $6.7 billion depending on the source).

The full technical post-mortem Binance promised is the document that will actually settle whether this was an isolated incident involving one specific engineer or a systemic gap in the exchange's privileged access model. Until then, any reconstruction — including this one — rests on third-party intelligence (TRM, Elliptic, Chainalysis) and partial statements from Binance itself, not a complete, official forensic report.

---

*Related: [BleepingComputer — Binance links $6.3 billion crypto theft to compromised DevOps laptop](https://www.bleepingcomputer.com/news/security/binance-links-63-billion-crypto-theft-to-compromised-devops-laptop/amp/) · [BleepingComputer — FBI links $6.3 billion Binance crypto heist to North Korean hackers](https://www.bleepingcomputer.com/news/security/fbi-links-63-billion-binance-crypto-heist-to-north-korean-hackers/) · [Forbes — Binance Lost $6.3 Billion In 20 Minutes. Who Holds The Keys?](https://forbes.com/sites/digital-assets/2026/11/02/binance-lost-63-billion-in-20-minutes-who-holds-the-keys) · [CBC — Binance says vast majority of stolen $6.3B replaced](https://www.cbc.ca/news/world/binance-hack-north-korea-9.7404502) · [NCC Group — In-Depth Technical Analysis of the Bybit Hack](https://nccgroup.com/sg/research-blog/in-depth-technical-analysis-of-the-bybit-hack)*
