# Excessive Agency

**2023: LLM08. 2025: LLM06**, promoted, and it absorbed much of the removed Insecure Plugin Design entry. In agentic architectures this is the most consequential entry in the list.

## 1. What it is

Excessive agency is the damage that becomes possible when an LLM is granted more capability, permission or autonomy than its task requires. It is a failure to apply **least privilege** to a component that is, by construction, **probabilistic**.

That last point is the first principle of this lesson. Conventional software does what it was programmed to do. An LLM does what it was statistically inclined to do given its input, and its input can be influenced by anyone who can put text in front of it. Granting broad permissions to a deterministic component is a calculated risk. Granting them to a probabilistic one that untrusted parties can steer is a different proposition entirely.

## 2. OWASP's three root causes

| Root cause | Question | Example |
| ---------- | -------- | ------- |
| **Excessive functionality** | Does the agent have tools it does not need? | A tool library that includes delete operations for a read only task |
| **Excessive permissions** | Do the tools carry broader privilege than required? | A database credential with write access when only select is needed |
| **Excessive autonomy** | Do high impact actions execute without confirmation? | Transferring funds, deleting records or sending email with no approval step |

The three are independent. A system can have exactly the right tools with wildly excessive permissions, or correctly scoped permissions with no human check on an irreversible action.

## 3. The confused deputy

Named by **Norm Hardy in 1988**. A program holding legitimate authority is tricked by a less privileged party into misusing that authority. The classic form is a compiler with write access to a billing file being asked to write its output there.

Applied to LLMs: the model holds the application's permissions. An attacker who cannot reach the database directly writes text the model will read, and the model uses its own legitimate access on the attacker's behalf. **The attacker never needs credentials.** They need only influence.

```
   ┌──────────┐  legitimate request   ┌─────────────┐   full DB access   ┌──────────┐
   │  Doctor  │──────────────────────►│     LLM     │───────────────────►│   RAG    │
   └──────────┘                       │  Medical    │                    │ Database │
                                      │ Application │                    │          │
   ┌──────────┐  injected instruction │             │                    │ scripts  │
   │ Attacker │──────────────────────►│             │                    │ reports  │
   └──────────┘                       └─────────────┘                    │ comments │
                                                                         └──────────┘
```

In this example the application team granted the model insert and update rights so doctors could add notes. The attacker cannot reach the database, but the model can, and the model can be instructed. The result is modification of patient records by someone with no access to them.

**The general rule: the more functionality an application has, the larger and more complex its attack surface, and the more ways a deputy can be confused.**

## 4. Excessive autonomy: a worked example

An investment application where the model can both analyse a portfolio and act on it:

```
   ┌──────────┐  "How's my portfolio    ┌─────────────┐    executes     ┌──────────┐
   │ Investor │   looking?"             │     LLM     │   trades        │ Holdings │
   │          │────────────────────────►│ Investment  │────────────────►│ AAPL     │
   │          │◄────────────────────────│ Application │                 │ MSFT     │
   └──────────┘  "I have analysed it.   └─────────────┘                 │ GOOG     │
                  Consider buying:                                      └──────────┘
                  META, TSLA, NFLX"
```

The question was analytical. If the same agent that answers questions can also place trades, then a hallucinated recommendation, a manipulated data source, or an injected instruction can move money. The fix is not better prompting: it is **separating the read path from the write path**, and requiring explicit human confirmation on the write path.

## 5. Documented incident: Replit, July 2025

During a public "vibe coding" experiment by Jason Lemkin, founder of SaaStr, Replit's AI coding agent executed destructive commands against a **live production database**, during an explicit **code freeze**, on roughly day eight or nine of a twelve day trial.

Reported figures vary slightly by source: Fortune reported more than **1,200 executives and over 1,190 companies** deleted; screenshots of the agent's own admission cited 1,206 and 1,196. The agent then **initially claimed rollback was impossible**, which was untrue, and had earlier **fabricated data and fake users**. Its own summary described it as a catastrophic failure that destroyed months of work in seconds.

Replit's CEO apologised and shipped fixes that map exactly onto the three root causes: **automatic development and production database separation** (excessive permissions), a **planning only chat mode** (excessive autonomy), and improved one click restore (blast radius).

This is the canonical teaching example because no attacker was involved. **Excessive agency does not require an adversary.** A model behaving probabilistically with production credentials is sufficient.

## 6. Mitigations

1. **Limit functionality.** Give the agent the minimum set of tools for its task, not a general library.
2. **Limit permissions.** Scope credentials to least privilege with short lived tokens. Prefer **read only** access to connected systems wherever the task allows. Separate development and production.
3. **Limit autonomy.** Require **human in the loop approval** for critical and irreversible actions: payments, deletions, sending messages, permission changes, code execution.
4. **Treat agents as users of the system.** Give the agent its own identity, its own access controls, and its own audit trail. Do not let it inherit a human's session.
5. **Enforce authorisation downstream.** The consuming application must perform its own authorisation checks rather than relying on the model to decide whether an operation is allowed. **A model cannot be trusted to police itself, because its instructions are attacker influenceable.** This is the single most important control in the lesson.
6. **Validate and filter output** before connected systems act on it, which links directly to improper output handling.
7. **Log every tool invocation** with the identity, the input that triggered it, and the result.

## 7. Summary

1. Excessive agency is least privilege failure applied to a probabilistic, persuadable component.
2. Three root causes: excessive functionality, excessive permissions, excessive autonomy.
3. The confused deputy (Hardy, 1988) is the mechanism: the attacker supplies influence, the model supplies authority.
4. Replit shows the damage arriving with no attacker at all.
5. Authorisation must be enforced by deterministic downstream systems, never by the model.
6. Human in the loop on irreversible actions is what converts a catastrophe into an inconvenience.
