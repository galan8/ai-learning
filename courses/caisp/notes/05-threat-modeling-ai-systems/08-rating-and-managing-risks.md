# Rating and Managing Risks

## Why rate at all

Risk management is the brakes on a vehicle. Brakes are not there to stop you driving; they let you drive faster, safely, because you can control your speed. Risk management works the same way: it does not block delivery, it lets an organisation move quickly while staying in control.

Threat modeling produces threats. Not all threats are equally risky, and resources are finite. Rating turns a list into an order of work.

## Risk in two questions

1. **Could something happen?** (likelihood)
2. **What if it does?** (impact)

Worked through the classic example, where the threat is theft of your house:

- **Likelihood:** what is the crime rate in the neighbourhood, how exposed does the house look, are there visible deterrents?
- **Impact:** what is inside worth stealing, and who is home?

Answering both gives you the robbery risk. Neither alone does.

## Risk management steps

1. **Risk identification.** Threat modeling identifies threats and the risks they pose.
2. **Risk analysis.** Calculate likelihood and impact.
3. **Risk prioritisation.** Order by risk value.
4. **Risk ownership.** Assign an owner to each risk. A risk without a named owner is not managed.
5. **Risk mitigation.** Bring higher risks down to an acceptable level. **Not every risk needs mitigating.**
6. **Risk monitoring.** Reassess as new threats appear and check that controls still hold.

## Risk treatment options

| Treatment | Meaning | LLM example |
| --------- | ------- | ----------- |
| **Avoid** | Remove the risk by not doing the thing | Do not connect the model to the production database at all |
| **Mitigate** | Reduce likelihood or impact with controls | Add output encoding, rate limits, human approval |
| **Accept** | Acknowledge and live with it, with a documented decision and owner | Accept residual hallucination risk on a low stakes internal summariser |
| **Transfer** | Move the risk to another party | Cyber insurance, or contractual liability with a model vendor |

Accepting a risk is a legitimate decision, not a failure, provided it is explicit, owned and time bound. Undocumented acceptance is just an unmanaged risk.

## OWASP Risk Rating Methodology

Risk = **Likelihood × Impact**, where each side is derived by scoring factors from 0 to 9 and **averaging them**, then banding the result.

### Likelihood factors

**Threat agent factors**

| Factor | Question | Scores |
| ------ | -------- | ------ |
| Skill level | How technically skilled is this group? | No technical skills (1), some technical skills (3), advanced computer user (5), network and programming skills (6), security penetration skills (9) |
| Motive | How motivated are they to find and exploit it? | Low or no reward (1), possible reward (4), high reward (9) |
| Opportunity | What resources and access are required? | Full access or expensive resources required (0), special access or resources required (4), some access or resources required (7), no access or resources required (9) |
| Size | How large is this group? | Developers (2), system administrators (2), intranet users (4), partners (5), authenticated users (6), anonymous internet users (9) |

**Vulnerability factors**

| Factor | Question | Scores |
| ------ | -------- | ------ |
| Ease of discovery | How easy is it to find? | Practically impossible (1), difficult (3), easy (7), automated tools available (9) |
| Ease of exploit | How easy is it to exploit? | Theoretical (1), difficult (3), easy (5), automated tools available (9) |
| Awareness | How well known is it? | Unknown (1), hidden (4), obvious (6), public knowledge (9) |
| Intrusion detection | How likely is the exploit to be detected? | Active detection in application (1), logged and reviewed (3), logged without review (8), not logged (9) |

### Impact factors

**Technical impact**

| Factor | Scores |
| ------ | ------ |
| Loss of confidentiality | Minimal non sensitive data disclosed (2), minimal critical data disclosed (6), extensive non sensitive data disclosed (6), extensive critical data disclosed (7), all data disclosed (9) |
| Loss of integrity | Minimal slightly corrupt data (1), minimal seriously corrupt data (3), extensive slightly corrupt data (5), extensive seriously corrupt data (7), all data totally corrupt (9) |
| Loss of availability | Minimal secondary services interrupted (1), minimal primary services interrupted (5), extensive secondary services interrupted (5), extensive primary services interrupted (7), all services completely lost (9) |
| Loss of accountability | Fully traceable (1), possibly traceable (7), completely anonymous (9) |

**Business impact**

| Factor | Scores |
| ------ | ------ |
| Financial damage | Less than the cost to fix (1), minor effect on annual profit (3), significant effect on annual profit (7), bankruptcy (9) |
| Reputation damage | Minimal damage (1), loss of major accounts (4), loss of goodwill (5), brand damage (9) |
| Non compliance | Minor violation (2), clear violation (5), high profile violation (7) |
| Privacy violation | One individual (3), hundreds of people (5), thousands of people (7), millions of people (9) |

### Bands and the severity matrix

Each average maps to a band: **0 to under 3 = LOW, 3 to under 6 = MEDIUM, 6 to 9 = HIGH.**

The course notes say "0 to less than 4 is low"; the published methodology uses **3** as the low to medium boundary.

Likelihood and impact are then combined in a matrix rather than multiplied numerically:

```
                      IMPACT
                 LOW    MEDIUM   HIGH
              ┌───────┬────────┬──────────┐
        HIGH  │ MED   │ HIGH   │ CRITICAL │
              ├───────┼────────┼──────────┤
 LIKELI MED   │ LOW   │ MEDIUM │ HIGH     │
 HOOD         ├───────┼────────┼──────────┤
        LOW   │ NOTE  │ LOW    │ MEDIUM   │
              └───────┴────────┴──────────┘
```

Note: OWASP now prefixes this page with a recommendation to consider more mature methodologies (NIST SP 800-30, the Government of Canada Harmonized TRA, or Mozilla's Rapid Risk Assessment) for formal work. Use OWASP's method for its transparency and speed, not because it is the most rigorous available.

## Worked example: an LLM customer support assistant

**Threat:** indirect prompt injection through a support ticket attachment causes the assistant to disclose another customer's data from the RAG store.

**Likelihood, threat agent factors**

| Factor | Rationale | Score |
| ------ | --------- | ----- |
| Skill level | Some technical skill to craft an injection payload | 3 |
| Motive | Customer data has resale value | 9 |
| Opportunity | Anyone can open a support ticket, no special access needed | 9 |
| Size | Anonymous internet users can submit tickets | 9 |

**Likelihood, vulnerability factors**

| Factor | Rationale | Score |
| ------ | --------- | ----- |
| Ease of discovery | Injection is trivially testable against a public endpoint | 7 |
| Ease of exploit | Public payload collections exist | 9 |
| Awareness | Prompt injection is public knowledge and is OWASP LLM01 | 9 |
| Intrusion detection | Prompts are logged but nobody reviews them | 8 |

**Overall likelihood** = (3+9+9+9+7+9+9+8) ÷ 8 = **7.875 → HIGH**

**Impact, technical**

| Factor | Rationale | Score |
| ------ | --------- | ----- |
| Loss of confidentiality | Extensive critical data (other customers' records) | 7 |
| Loss of integrity | Nothing is modified | 1 |
| Loss of availability | Service stays up | 1 |
| Loss of accountability | The action runs under the attacker's own ticket, so partly traceable | 7 |

**Impact, business**

| Factor | Rationale | Score |
| ------ | --------- | ----- |
| Financial damage | Significant effect on annual profit through remediation and penalties | 7 |
| Reputation damage | Loss of goodwill | 5 |
| Non compliance | Clear violation of data protection obligations | 5 |
| Privacy violation | Thousands of people potentially affected | 7 |

**Overall technical impact** = (7+1+1+7) ÷ 4 = **4.0 → MEDIUM**
**Overall business impact** = (7+5+5+7) ÷ 4 = **6.0 → HIGH**

**Result:** likelihood HIGH, business impact HIGH → **CRITICAL**. Technical impact alone would have rated it MEDIUM, and the business impact is what makes it critical. That is the argument for scoring both.

**Treatment:** mitigate. Enforce per user authorisation at retrieval time so the model can only retrieve what the requesting customer is entitled to (this addresses impact directly and does not depend on stopping the injection). Add output filtering, review prompt logs, and treat ticket attachments as untrusted content.

**Note on the scoring:** the numbers encode judgement, not measurement. Their value is that the judgement is explicit and can be challenged. Two engineers disagreeing about whether intrusion detection scores 3 or 8 are having a far more productive argument than two engineers disagreeing about whether a risk is "high".
