# Smart Sampa: Structural Cybersecurity Failures and Contractual Fragility

> **Category:** Security · Incident Analysis · Government Infrastructure
> **Level:** Foundational

---

The crisis at **Smart Sampa**, São Paulo's ambitious facial-recognition video surveillance program budgeted at R$588 million, is far from an isolated case of individual misconduct or "leaked passwords." While the city administration attributes the incident to the unlawful conduct of a single employee, a technical review of the infrastructure and the program's backstory reveals an alarming picture of multi-layered vulnerabilities, bare-minimum credential hygiene, and opacity across the supplier chain.

The case exposed, for roughly 11 months, credentials and real-time feeds from more than **58,000 cameras** spread across the capital, raising serious doubts about the public sector's ability to protect citizens' privacy and data.

---

## Technical Diagnosis: Architectural Weaknesses

As revealed by the investigation published on the **Penegui channel** in the video *"Smart Sampa: fui investigar e achei MAIS do que SENHAS VAZADAS"* ("I went to investigate and found MORE than LEAKED PASSWORDS"), the system's problem is not limited to a public repository with 181 logins. The investigation detailed primary, systemic failures that strike directly at the platform's security architecture:

**Primitive Credential Hygiene**
Active accounts were found where the username and password were exactly identical to the public agent's employee registration number. This kind of weakness violates the most basic principles of information security in critical infrastructure.

**Prior Infostealer Leaks (2024)**
Penegui's investigation identified at least 15 credential and session-token records tied to the system, leaked by infostealer malware as far back as 2024. This shows that the compromise of the access base predates the public repository exposure reported by the press by a wide margin.

**VPN Exposure and Attack Surface**
Platform subdomains and VPN login pages were publicly visible on the internet. In networks where IP-address restriction is used as the only barrier, compromising a single VPN credential lets an attacker simulate authorized internal access.

**Session and Authorization Failures (IDOR)**
Changing an account's password did not automatically invalidate session tokens that were already active. In addition, the lack of rigorous permission checks across interfaces and endpoints, a risk associated with IDOR (Insecure Direct Object Reference) flaws, raises the suspicion that an account authorized in one region could access cameras and data from other areas without proper environment segregation.

## Context: Alerts Ignored Since 2022

Smart Sampa's structural weaknesses did not appear with the 2026 leak. Idec, together with the Laboratório de Políticas Públicas e Internet (LAPIN) and other organizations, filed a complaint with the São Paulo Public Prosecutor's Office (MP-SP) back in 2022, shortly after the program's bidding notice was published, flagging the risk of algorithmic discrimination (racial bias in facial recognition) and the absence of social oversight of the technology.

The city itself confirms a central point of that criticism: it has no access to the source code, the AI models, or the training data used by the system, and it does not run independent audits of those components. In other words, the city outsourced facial recognition over its population without ever being able to audit how that recognition is performed.

A technical note published in February 2026 by LAPIN, the Instituto de Referência Negra Peregum, and Rede Liberdade, based on requests under Brazil's Freedom of Information law, had already documented the human cost of these failures even before the credential leak: at least 23 people were wrongly brought in to police stations due to facial-recognition inconsistencies, and 82 were arrested and later released. Municipal records gathered by the same investigation counted 268 such detention-then-release incidents between November 2024 and May 2026. A study by the Centro de Estudos de Segurança e Cidadania (CESeC), for its part, found no significant reduction in crime rates attributable to the program.

After the leak, Idec itself summarized the problem in a line that captures this piece's central thesis, coming from an independent research organization and not merely from an individual investigation: "It is not enough to claim the platform is secure." The organization is asking the city to investigate the system's structural failures, not just the conduct of the employee who was removed.

## Requirements Analysis: Where Software Engineering and Security Failed

From a software engineering and cybersecurity architecture standpoint, a critical R$588 million project demands a rigorous elicitation and specification of **Functional and Non-Functional Requirements**. The Smart Sampa case demonstrates systematic negligence at that planning stage:

### Neglected Non-Functional Security Requirements

* **Authentication and Access Control:** the system never established Multi-Factor Authentication (MFA/2FA) as a mandatory requirement for any user level, allowing direct access with just a username and password.
* **Session Lifecycle Management:** there was no specification for short token expiration or for immediate invalidation of active sessions the moment a password is reset.
* **Zero Trust and Defense in Depth:** the architecture was designed on the assumption that the login perimeter was enough. In critical infrastructure systems, requirements must assume that every internal request is authenticated and authorized granularly.
* **Encryption and Data Protection:** there were no clear requirements for end-to-end encryption of data in transit and at rest, nor for logical and physical segregation of the databases.

### Governance and Vendor Qualification Requirements

* **Contractor Qualification Criteria:** the bidding notice and contracting requirements did not demand a proven track record from the vendor in developing complex, high-criticality software or cyber defense systems.
* **Source-Code Audits and Penetration Tests:** independent penetration testing and transparent source-code auditing were never established as delivery requirements (quality gates) before launch.

### LGPD Compliance and Privacy Requirements

* **Traceability and Audit Logs:** the system failed to guarantee immediate, automatic traceability of suspicious queries, allowing unauthorized access to persist for months without automatic alerts to the Security Operations Center (SOC).
* **Data Protection Impact Assessment (DPIA):** the absence of a detailed prior DPIA prevented proper risk mapping when monitoring highly sensitive environments, such as daycare centers and public hospitals.

---

## Deconstructing the "Honeypot" Narrative and Contractual Opacity

In response to the leaks, the city administration argued the system was secure and suggested the exposure might have been a deliberate "honeypot" to identify illegal access.

That justification was technically challenged in the Penegui channel's video: a legitimate honeypot is a simulated, isolated structure with no connection to real production data. In Smart Sampa's case, the exposed credentials allowed real viewing of cameras in sensitive public locations and browsing of security agents' data. Children, patients, and officers on duty cannot be used as virtual "bait."

The investigation also shed light on the supply chain:

* **Sentinel X:** the company responsible for the program's technology was incorporated just 29 days after the original contract was signed, and had no track record in developing complex software or artificial intelligence.
* **Ties to CLD:** the examination of the shareholders found ties to executives of the former CLD (Consladórcio), a company with a history in road infrastructure and speed-camera (radar) contracts, as well as involvement in legal disputes with an asset freeze exceeding R$1 billion tied to public-lighting contracts.

Smart Sampa's original contract was signed in August 2023 with the Consórcio Smart City SP, then made up of CLD Construtora, Laços Detentores e Eletrônica, Flama Serviços, Camerite Sistemas, and PK9 Tecnologia, at a monthly cost of R$9.8 million. Already in the first phase, the consortium fell behind on delivery: of the 5,000 cameras planned, only 2,963 had their images made available on schedule. Sentinel X, the company now at the center of the facial-recognition controversy, did not take part in the original bid: it joined the consortium through a contract amendment signed in December 2025, replacing Camerite, effective January 2026. In other words, the AI vendor now underpinning the program was swapped in through an administrative amendment less than a year before the leak came to light, with no new competitive process.

---

## The City's Response and What Was Left Out

After confirming the leak reported by UOL on October 20, 2026, the city announced a set of technical measures: taking down the exposed repository, adopting exclusive per-user passwords, two-factor authentication, and SMS-based codes for creating new passwords. In parallel, it stated that Smart Sampa is subject to periodic oversight by the Tribunal de Contas do Município (TCM), with audits and security testing provided for in the contract. As of this writing, there is no public TCM report specifically covering this episode, which leaves that claim resting solely on the official version.

There is also a factual discrepancy worth noting: the city denies that the leaked material contained access passwords, while the original UOL report points to 181 exposed login-and-password combinations. That kind of contradiction between the institutional narrative and the data gathered by the press is, on its own, a symptom of the lack of transparency discussed throughout this piece.

As of October 27, 2026, the state of the external investigations was as follows: the São Paulo Public Prosecutor's Office (MPSP) had opened only a preliminary inquiry ("Notícia de Fato"), not a formal civil investigation, to decide whether there is cause to proceed; the Brazilian data protection authority (ANPD) stated only that it is "monitoring the case within the scope of its legal authority," without detailing any concrete measures; and the independent technical examination ordered by the 3rd Public Treasury Court, in a ruling signed on October 5 (before the leak was even made public), was still awaiting formal confirmation from NEV-USP that it would take on the assignment, along with the naming of the responsible experts.

One detail sums up well the gap between the city's cautious rhetoric and its actions: days after the leak, on October 27, 2026, the city itself announced new agreements with the Public Prosecutor's Office to expand Smart Sampa's use against organized crime. The system's use is expanding at the very moment its security and governance failures are under investigation, before the independent examination has even begun.

## What Should Be Done: Independent Examination and Accountability

Given the findings and the actions brought by civil society organizations (such as LAPIN, Conectas, Idec, and the Public Defender's Office):

* **Independent Technical Examination Ordered by the Courts:** the 3rd Public Treasury Court of São Paulo rejected the city's attempts to waive the technical evidence and ordered an independent examination of Smart Sampa. The Núcleo de Estudos da Violência at USP (NEV-USP) was named to assess information security, encryption, data governance, and the accuracy of the algorithms.
* **ANPD and MPSP Investigations:** Brazil's national data protection authority (ANPD) is monitoring the incident and looking into the city's failure to provide formal notification, while the São Paulo Public Prosecutor's Office (MPSP) is weighing whether to open a civil investigation.

The Smart Sampa case reinforces a fundamental lesson in cybersecurity: the effectiveness of technology in public safety does not depend on the number of cameras installed, but on the rigor of the software architecture, the requirements analysis, and the continuous protection of citizens' data.

---

## Sources consulted for additional context

* [Penegui: Smart Sampa, fui investigar e achei MAIS do que SENHAS VAZADAS](https://www.youtube.com/watch?v=o1SA9DsKfZU)
* [Idec: Vazamento no Smart Sampa reforça alerta feito desde 2022](https://idec.org.br/imprensa/vazamento-dados-smart-sampa-cameras)
* [Agência Brasil: pesquisa aponta problemas e prisões indevidas a partir do Smart Sampa](https://agenciabrasil.ebc.com.br/geral/noticia/2026-02/pesquisa-aponta-problemas-e-prisoes-indevidas-partir-do-smart-sampa)
* [Geocracia: São Paulo reforça controle de acesso ao Smart Sampa após vazamento](https://geocracia.com/sao-paulo-reforca-controle-de-acesso-ao-smart-sampa-apos-vazamento/)
* [CNN Brasil: Smart Sampa deve passar por perícia independente após vazamento de dados](https://www.cnnbrasil.com.br/nacional/sudeste/sp/smart-sampa-deve-passar-por-pericia-independente-apos-vazamento-de-dados/)
* [CNN Brasil: MPSP inicia análise dos vazamentos de dados do Smart Sampa](https://www.cnnbrasil.com.br/blogs/jairo-nascimento/nacional/sudeste/sp/mpsp-inicia-analise-dos-vazamentos-de-dados-do-smart-sampa/)
* [CNN Brasil: ANPD acompanha invasão e exposição de dados do sistema Smart Sampa](https://www.cnnbrasil.com.br/nacional/sudeste/sp/anpd-acompanha-invasao-e-exposicao-de-dados-do-sistema-smart-sampa/)
* [Diário ESP: Prefeitura de SP firma convênios com MP para usar Smart Sampa contra crime organizado](https://diarioesp.com.br/sao-paulo/2026/10/27/prefeitura-de-sp-firma-convenios-com-mp-para-usar-smart-sampa-contra-crime-organizado.html)
* [Mobile Time: Idec vê risco de racismo estrutural com projeto de câmeras e reconhecimento facial em São Paulo](https://www.mobiletime.com.br/noticias/22/05/2023/idec-ve-risco-de-racismo-estrutural-com-projeto-de-cameras-e-reconhecimento-facial-em-sao-paulo/)
* [Prefeitura de São Paulo: prefeito assina contrato para o início do Smart Sampa](https://prefeitura.sp.gov.br/w/noticia/prefeito-assina-contrato-para-o-inicio-do-smart-sampa-maior-programa-de-videomonitoramento-da-cidade-com-ate-40-mil-cameras-2)
* [Jornal 24 Horas: Smart Sampa, entenda o que vazou e quem são as empresas que operam o sistema](https://jornal24horas.com.br/?p=69303)
