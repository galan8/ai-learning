# Automation of Vetting and Third-Party Code

## 1. Why automate

Managing dependencies manually does not work at modern scale. An average Java application carries around 148 dependencies, updating roughly ten times a year, which means tracking on the order of 1,500 dependency changes annually for a single application.

**Automation is not an efficiency improvement here; it is the only way the process runs at all.** A manual vetting framework that cannot keep pace produces stale assurance, which is worse than none because it is believed.

## 2. Where automation applies

**Dependency retrieval and management.** Build tools such as Maven, Gradle, npm and pip already automate retrieval. The security question is what they are permitted to retrieve and from where, which is why internal repositories and allowlists matter more than the tool choice.

**Static analysis (SAST).** SonarQube, Fortify, Checkmarx. Analyses source without running it.

**Software composition analysis and open source risk.** Black Duck, Snyk, Mend (formerly WhiteSource). These identify third party components, their known vulnerabilities and their licences, and are the automated form of step 6 in the vetting process.

**Dynamic analysis (DAST).** Tests the running application.

**Configuration management.** Ansible, Chef and Puppet enforce consistent configuration, which mitigates the risk of drift and misconfiguration.

**Cloud security posture.** AWS Security Hub, Microsoft Defender for Cloud and equivalents manage dependencies and configuration in cloud services and produce risk scores, including compliance mapping such as PCI DSS.

**Custom automation.** Scripts and bespoke tooling fill the gaps between products, and in practice most pipelines need some.

## 3. Where it belongs: CI/CD

The pipeline is the enforcement point. A scan that runs somewhere other than the pipeline is advisory; a scan that gates a build is a control.

```
   commit ──► build ──► SAST ──► SCA ──► container scan ──► DAST ──► deploy
                          │        │           │              │
                          └────────┴───────────┴──────────────┘
                                   fail the build on policy breach
```

**The design decision that matters is what fails the build.** Failing on every finding stops delivery and gets the gate disabled. Failing on nothing makes it theatre. The usual answer is severity plus exploitability plus reachability, which is why VEX matters.

## 4. Collaboration and workflow

Findings need to reach the people who fix them. Integration with Jira, Confluence and equivalents turns scan output into tracked work with an owner and a due date, and provides the compliance status reporting that the vetting process's documentation step requires.

**Developers and security teams both need training on the tooling**: how to use it, how to interpret findings, and how to respond. A tool nobody trusts gets ignored, and the fastest way to lose trust is unexplained false positives.

## 5. The limits of automation

Worth stating plainly, because the lesson does not:

1. **Automation finds known problems.** CVE scanners match against databases of disclosed vulnerabilities. A malicious package with no CVE, a poisoned model, or a backdoor passes cleanly.
2. **False positives erode the control.** A pipeline that cries wolf gets bypassed.
3. **Coverage is uneven.** Tooling for application dependencies is mature; tooling for models, datasets and AI artifacts is not.
4. **Automation cannot make risk decisions.** It can tell you a component has a critical CVE. Whether to accept, mitigate or replace is a judgement call with business context.

## 6. Summary

1. Manual dependency management does not scale to ~1,500 changes per application per year.
2. Automate across SAST, SCA, DAST, configuration management and cloud posture.
3. The CI/CD pipeline is where automation becomes enforcement rather than advice.
4. Deciding what fails the build is the critical design choice.
5. Route findings into tracked work with owners, and train both developers and security on the tooling.
6. Automation catches known issues only; poisoned models and novel malicious packages pass through.
