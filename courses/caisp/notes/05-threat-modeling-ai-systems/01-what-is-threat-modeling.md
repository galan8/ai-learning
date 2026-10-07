# What is Threat Modeling

## Definition

Threat modeling is a technique to analyse an application or system **from a design perspective** to find security vulnerabilities, without running or even building the code.

The consensus definition from the Threat Modeling Manifesto (2020): threat modeling is analysing representations of a system to highlight concerns about security and privacy characteristics.

Because it works on the design rather than the artifact, it can be done at any point, but it is most valuable in the **design phase**, since architectural changes become progressively more expensive in time and effort the later they are made.

## The first principle

Every other technique in application security examines something that already exists. SAST reads code. DAST probes a running system. SCA inspects dependencies. All of them find **implementation** defects, and all of them require the thing to exist first.

Threat modeling is the only technique that finds **design** defects, and it is the only one available before anything is built. A design flaw cannot be found by scanning code, because the code correctly implements a flawed design. That is the gap threat modeling fills, and it is why it cannot be replaced by tooling.

## Framing that helps

**"A fancy name for something we all do instinctively."** Anyone who has wondered who might break into their house, through which window, and whether the lock is good enough, has threat modeled. The discipline is the same reasoning applied systematically to a system.

**"Evil brainstorming."** Tanya Janca's framing: take off the developer hat, put on the attacker hat, and ask how you would undo the system's greatness. Her own phrasing, from SE Radio: she knows this annoys Adam Shostack, but she likes to think of threat modeling as evil brainstorming. The value is that it gives non security engineers permission to think adversarially without needing a security background.

## The four questions

The framework introduced by Adam Shostack and adopted as the core of the Threat Modeling Manifesto:

1. **What are we working on?** The system, the software, and the assets valuable to the organisation. What are we building, creating or storing, and why is it an asset?
2. **What can go wrong?** Who might want to attack, harm or steal those assets, and how might they abuse the design?
3. **What are we going to do about it?** What safeguards protect the assets and prevent harm to the organisation and its customers?
4. **Did we do a good enough job?** Was the threat model adequate, and what could be improved?

The fourth question is the one most often skipped, and it is what turns threat modeling from an event into a practice.

## The working loop

```
   Pick a use case
         │
         ▼
   Visualise it  (draw a data flow diagram: how data moves,
         │        which components and third parties are involved)
         ▼
   Identify threats  (for each asset and each trust boundary,
         │            what could go wrong?)
         ▼
   Rate and prioritise  (how likely, how damaging?)
         │
         ▼
   Decide and implement controls  (mitigate, avoid, transfer, accept)
         │
         ▼
   Test the mitigations
         │
         └──────────► repeat continuously as the design changes
```

Every threat eventually poses a risk. Identifying threats without rating and treating the resulting risks leaves the work unfinished.

## Where it sits alongside risk analysis

Threat modeling does not stand alone. In a mature organisation it forms a cycle with risk management and delivery:

```
   Risk Analysis ──informs──► Engineering Story ──triggers──► Threat Modelling
        ▲                                                            │
        └──────────────────── mitigates ◄────────────────────────────┘
```

1. **Risk analysis**: risk management, infosec and technical managers identify risks and hazards.
2. **Engineering story**: technical managers and product owners turn those into work to mitigate them.
3. **Threat modeling**: technical managers and engineers run a session focused on identifying and mitigating threats in line with the risks identified.

The output of threat modeling feeds back into risk analysis, and the loop repeats.
