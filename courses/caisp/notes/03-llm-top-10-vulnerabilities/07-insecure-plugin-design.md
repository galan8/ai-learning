# Insecure Plugin Design

**2023: LLM07. In the 2025 list this entry was removed**, its concerns distributed across **Excessive Agency (LLM06)** and **Supply Chain (LLM03)**. The threat did not go away; OWASP concluded it was better handled as two problems, one about what the tool is permitted to do and one about where the tool came from.

## 1. What it is

Plugins (now more commonly called **tools**, **actions** or **MCP servers**) extend an LLM beyond text generation by connecting it to external resources: reading web pages, searching jobs, querying code repositories, booking travel, sending email.

A plugin is **an API endpoint whose caller is a language model that can be persuaded by anyone who can put text in front of it.** That single sentence contains the whole vulnerability class.

## 2. How a plugin gets attacked

**Any functionality a plugin provides can be abused.** The security question is not only what the plugin is intended to do, but what it *can* do when invoked by an untrusted instruction.

```
   Attacker plants instructions in content
              │
              ▼
   Model reads that content (web page, document, email, issue)
              │
              ▼
   Model decides to call a plugin  ◄── the confused deputy moment
              │
              ▼
   Plugin executes with ITS OWN permissions, not the attacker's
              │
              ▼
   Action taken / data returned / data exfiltrated
```

The model is a **confused deputy**: it holds legitimate authority (the plugin's permissions) and is tricked by a less privileged party (whoever wrote the content) into misusing it.

### Attack examples

1. **Browser plugin.** A plugin with browsing access can read history and reachable resources, and can fetch attacker chosen URLs, which doubles as an exfiltration channel.
2. **SQL plugin.** A plugin connected to a database that blindly trusts model generated SQL can drop tables and destroy data.
3. **Code plugin.** A plugin with repository access can be induced to change repository settings or expose private source.
4. **Email plugin.** A plugin that reads mail can be instructed to search for password reset messages and forward their contents to an attacker controlled destination. This is the highest severity case because password reset emails are the key to every other account.

## 3. Documented research: Johann Rehberger

Rehberger (embracethered.com) produced the foundational work on this class in 2023.

**Cross Plugin Request Forgery (CPRF), May 2023.** The first documented end to end indirect prompt injection into confused deputy exploit in the ChatGPT plugin ecosystem: a malicious website's injected instructions caused the model to invoke plugins and exfiltrate personal data. The name is deliberate, echoing CSRF, where a browser is tricked into using the user's authority.

**Data exfiltration via markdown image rendering.** ChatGPT automatically rendered markdown images, so a response containing an image reference to an attacker controlled URL with data in the query string silently transmitted that data when displayed. Microsoft and Anthropic addressed this class of bug in their products.

**Chat with Code plugin.** A real exploit in which prompt injection from a visited page modified GitHub permissions and **turned private repositories public**.

**Email exfiltration proof of concept.** Letting the assistant visit a website resulted in email contents being stolen, with no human in the loop at any point.

## 4. What changed: plugins became MCP

The plumbing has moved: ChatGPT plugins were deprecated in favour of GPTs, then Actions, and the industry has converged on the **Model Context Protocol (MCP)** as the standard mechanism for connecting models to tools. **The attack class is unchanged.** Untrusted tool integration plus a confused deputy plus indirect injection produces the same outcome regardless of the protocol.

MCP's own security failures, all from 2025, are what a current threat model needs:

1. **Tool poisoning** (named by Invariant Labs, 1 April 2025). Hidden instructions embedded in a **tool's description**, which the model reads but the user usually cannot see, execute when the agent selects the tool. The proof of concept used a poisoned arithmetic tool to exfiltrate the victim's MCP configuration file and SSH keys. A demonstration against a WhatsApp integration exfiltrated an entire message history with nothing visibly wrong in the tool output.
2. **Rug pull.** A server silently redefines a tool's behaviour *after* the user approved it. Most clients do not detect the change.
3. **Toxic agent flows.** Invariant Labs showed a malicious **public GitHub issue** hijacking an assistant into leaking **private repository** data. No tool was compromised: trusted tools plus untrusted content were sufficient.
4. **Cross tenant exposure.** Asana's MCP server, launched 1 May 2025, had an access control flaw found on 4 June 2025; Asana disabled it for remediation and estimated roughly **1,000 customers** affected, exposing tasks, project metadata, comments and files.
5. **Remote code execution.** **CVE-2025-6514** in the `mcp-remote` package, **CVSS 9.6**, disclosed by JFrog on 9 July 2025, affecting versions 0.0.5 to 0.1.15 and fixed in 0.1.16. The package had been downloaded over 437,000 times. It was the first real world full RCE from simply connecting to an untrusted remote MCP server.

**The lethal trifecta** (Simon Willison, 16 June 2025) is the most useful design heuristic to come out of this: an agent that combines **private data access, exposure to untrusted content, and an outbound communication channel** will lose data to prompt injection. Remove any one of the three and the attack fails.

## 5. Mitigations

**From the lesson:**

1. **Consider a plugin's functionality carefully before building it.** Narrow scope is the primary control.
2. **Evaluate access rights during design**, not after deployment.
3. **Sanitise source code and other input** before the plugin consumes it.
4. **Replace automatic plugin actions with human in the loop** design for anything consequential.

**Additions for the MCP era:**

5. **Treat every MCP server and plugin as untrusted**, including its tool descriptions, which are model readable instructions.
6. **Pin and verify tool definitions**, and detect changes, which addresses rug pulls.
7. **Apply the lethal trifecta test** to every agent configuration: private data, untrusted content, outbound channel. Break one.
8. **Scope credentials per tool** with least privilege and short lived tokens. A read only token cannot drop a table.
9. **Enforce authorisation downstream**, in the resource, not in the model or the plugin.
10. **Use an MCP gateway** to centralise policy, logging and allowlisting rather than letting clients connect anywhere.

## 6. Summary

1. A plugin is an API whose caller is a persuadable model, which makes the model a confused deputy.
2. Any capability a plugin has is a capability an injection inherits.
3. Rehberger's 2023 work established the class: CPRF, image rendering exfiltration, private repositories exposed, email stolen.
4. Plugins became Actions became MCP; the protocol changed and the attack class did not.
5. MCP added tool poisoning, rug pulls and cross tenant leaks, with real CVEs including an RCE at CVSS 9.6.
6. The lethal trifecta is the design test: private data, untrusted content, outbound channel. Remove one.
