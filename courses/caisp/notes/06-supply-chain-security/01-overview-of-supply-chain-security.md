# An Overview of Supply Chain Security

## 1. Why this matters

Modern software is assembled, not written. Synopsys' Open Source Security and Risk Analysis reports have consistently found that **roughly three quarters of the code in audited codebases comes from open source dependencies** rather than from the organisation shipping it.

That means the majority of your attack surface is code you did not write, did not review, and often do not know you have.

**The economic logic for the attacker:** compromising one widely used component reaches every downstream consumer at once. Attacking a target directly reaches one target. Attacking their supply chain reaches thousands.

## 2. The numbers

From Sonatype's State of the Software Supply Chain research:

1. **742% average annual increase** in software supply chain attacks over a three year period (measured since 2019). Note: the figure is 742%, not 743%.
2. **Billions of vulnerable dependency downloads per month.** Sonatype reported 1.2 billion known vulnerable, avoidable downloads monthly in one framing and 3.4 billion monthly downloads of vulnerable software where a fixed version already existed in another. Both figures come from the same report family; quote whichever with its framing.
3. **Six out of every seven project vulnerabilities come from transitive dependencies**, not the ones you chose directly. This is the number that matters most: you are exposed mainly through dependencies you never selected and probably cannot name.
4. **96% of vulnerable open source Java downloads were avoidable**, because a fixed version already existed and was not used.

That last point reframes the problem. Most supply chain exposure is not zero day. It is **known vulnerabilities, already fixed upstream, still being downloaded**.

## 3. What a software supply chain is

The **software supply chain** is the process of developing, producing and distributing software, including every component and step involved. It begins with development and the components pulled in there: libraries, modules, frameworks and their dependencies.

The broader definition (NIST): a linked set of resources and processes between and among multiple levels of an enterprise, each of which is an acquirer, beginning with the sourcing of products and services and extending through the product and service lifecycle.

**The key insight in that definition: every party is simultaneously a consumer and a supplier.** You inherit from upstream and pass it downstream, usually without adding any verification in the middle.

## 4. Recent supply chain attacks, by lifecycle stage

NIST frames supply chain risk across six lifecycle stages. Real incidents at each:

| Stage | Incident |
| ----- | -------- |
| **Design** | **Hijacked cellular devices (2016).** A foreign company designed software used by a US phone manufacturer; the phones made encrypted records of texts, call histories and contacts and transmitted them to a foreign server every 72 hours |
| **Development and production** | **SolarWinds (2020).** An IT management company was infiltrated; the threat actor persisted for months, compromised the build servers and used the legitimate update process to reach customer networks |
| **Distribution** | **End user device malware (2012).** Researchers investigating counterfeit software found malware preinstalled on 20% of devices tested, installed after shipping from factory to distributor, transporter or reseller |
| **Acquisition and deployment** | **Kaspersky antivirus (2017).** An overseas antivirus vendor was assessed as being used by a foreign intelligence service; US government customers were directed to remove it |
| **Maintenance** | **Backdoors in routine updates (2020).** Thousands of public and private networks were infiltrated when a threat actor used a routine update to deliver a backdoor |
| **Disposal** | **Sensitive data spillage (2019).** A researcher bought used computers, drives and phones and found only two of 85 devices properly wiped, recovering hundreds of instances of PII including social security and passport numbers |

**Disposal is the stage everyone forgets**, and it requires no technical sophistication at all.

## 5. Other landmark cases

**Log4Shell (December 2021).** Log4j is the most widely used Java logging library, embedded in an enormous number of systems, usually as a transitive dependency nobody had catalogued. CISA later reported that Iranian government sponsored actors exploited Log4Shell to compromise a US federal agency network via an unpatched VMware Horizon server. It is the canonical example of why an SBOM matters: most affected organisations could not answer "are we running Log4j?" for days or weeks.

**Codecov (2021).** Attackers modified Codecov's Bash Uploader script, which ran inside customers' CI pipelines and exfiltrated environment variables, meaning credentials and tokens, from every build that used it. Undetected for roughly two months.

**CCleaner (2017).** After Avast acquired Piriform, attackers who had compromised the build environment distributed a malware carrying version of CCleaner through the official update channel to millions of users. Signed, legitimate, and malicious.

**Typosquatted PyPI packages.** Imposter HTTP libraries and similar packages have repeatedly been published to PyPI under names one character away from legitimate ones. Simple, cheap and persistent.

## 6. The pattern across all of them

```
   Attacker compromises          Legitimate distribution
   something UPSTREAM      ──►   mechanism carries it     ──►  Thousands of
   (build server, script,        (update, package          victims, each
   package name, vendor)          manager, installer)      trusting the source
```

**The distribution channel is the weapon.** SolarWinds, CCleaner and Codecov all used the victim's own trusted update or build process. This is why "download from the official source" is not a defence: in each of those cases the official source was the delivery vehicle.

## 7. Secure by default

Because attackers now target trusted supply chains rather than perimeters, defence has to shift from "is this vendor reputable?" to **"can I verify this specific artifact?"** That shift is what the rest of the chapter is about: vetting, scanning, pinning, SBOMs, provenance, attestation and signing.

## 8. Summary

1. Roughly three quarters of code in a modern codebase is open source dependency code.
2. Supply chain attacks grew at a 742% average annual rate over three years to 2022.
3. Six of seven vulnerabilities arrive through transitive dependencies you never chose.
4. Most exposure is to known, already fixed vulnerabilities, not zero days.
5. Risk spans the whole lifecycle from design to disposal.
6. The recurring pattern is compromise upstream, delivered through a trusted channel.
7. Reputation is not verification, and verification is what the rest of this chapter builds.
