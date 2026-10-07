# Why Threat Model

## Through a SWOT lens

**Strengths.** Easy to get started. Requires nothing except the design of the application, no code, no environment, no licences.

**Weaknesses.** Requires genuine understanding of the architecture, the security domain and the design. The presence of subject matter experts is mandatory, not optional, and that is a scheduling constraint as much as a skills one.

**Opportunities.** Keeping threat models in sync with the current design, expressed **as code** and versioned alongside it, turns a document into a living artifact and fits agile delivery.

**Threats (to the practice itself).** Threat modeling does not fit neatly into CI/CD. Models become obsolete quickly when design changes are not reflected back into them.

## Benefits

1. Finds design flaws early in the SDLC, when they are cheapest to fix
2. Acts as a conversation starter with stakeholders; it is inherently collaborative
3. Serves as a risk management technique for prioritisation and budget optimisation
4. Documents and manages risk early, rather than discovering it in production
5. Gives stakeholders visibility for future planning
6. Builds relationships between security and engineering teams

The second and sixth points are underrated. A threat model session is often the first time architects, developers, QA and security are in one room reasoning about the same design. That shared understanding frequently outlasts the document.

## Challenges

1. **Mostly manual.** SAST, DAST and SCA can be automated; threat modeling largely cannot, because design is always evolving and multiple stakeholders (architects, developers, QA, security) are involved.
2. **Yet another thing to learn.** DFD notation is a new skill for most engineers.
3. **Inconsistent between teams.** The same system threat modeled by two teams produces two different models.
4. **Bloat.** Attempting comprehensiveness produces models too large to act on.
5. **Keeping models in sync with the current design is the single biggest challenge.**
6. **Too heavy for agile and DevOps.** Traditional threat modeling assumes a design phase that short scrum cycles do not have.
7. **Non security engineers are uncomfortable making security decisions.** Developers make security decisions constantly and look to security for advice; security must understand the constraints developers face and collaborate rather than dictate.
8. **Time consuming**, especially in agile setups with many stakeholders.
9. **No upfront design**, so design changes are constant.
10. **No automated tooling that fits CI/CD.**
11. **Security cannot be embedded in every scrum team.** A centre of excellence is more realistic, but is not hands on.

## The honest conclusion

Most of these challenges share one root cause: **traditional threat modeling was designed for a waterfall world and is being applied to an agile one.** The practical responses are to keep models small and scoped to a single use case rather than an entire system, to store them as code beside the design so they version with it, and to accept that a lightweight model kept current beats a comprehensive model that is out of date.

The value proposition survives the challenges: it is the only method that finds design flaws before code exists, and design flaws are the most expensive class of defect to fix later.
