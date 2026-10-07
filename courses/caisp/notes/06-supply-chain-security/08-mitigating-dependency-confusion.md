# Mitigating Dependency Confusion

## 1. How the attack works

```
   INTERNAL (private registry)          PUBLIC (npm, PyPI)
   ───────────────────────────          ──────────────────
   mycompany-internal-coreutils    ◄──  mycompany-internal-coreutils
   mycompany-internal-eventstream  ◄──  mycompany-internal-eventstream
   mycompany-internal-logmanager   ◄──  mycompany-internal-logmanager
   mycompany-internal-beautifier   ◄──  mycompany-internal-beautifier
   mycompany-internal-bower        ◄──  mycompany-internal-bower
   mycompany-internal-regex        ◄──  mycompany-internal-regex
        (legitimate)                        (attacker registered,
                                             higher version number)
```

1. The attacker **discovers the names of an organisation's internal packages**, which are not published publicly. Sources include leaked `package.json` files, public repositories, job adverts, error messages, CI logs, and simple guessing.
2. They **register packages with those exact names on the public registry**, typically with a very high version number.
3. The victim's package manager, resolving a dependency, **prefers the public package** because default resolution checks public registries and takes the highest version.
4. The malicious package installs and executes.

**The root cause is a resolution ambiguity, not a vulnerability.** Nothing is exploited. The package manager does exactly what it was designed to do; the design simply assumed names were unique across a single namespace.

This is precisely what happened to PyTorch: `torchtriton` was an internally hosted dependency, an attacker registered the name on public PyPI, and pip preferred the public one.

## 2. Mitigations

**a) Register your internal names publicly.** Create placeholder packages on the public registry using your internal package names, so an attacker cannot. This is what PyTorch did after the torchtriton incident: they renamed the dependency to `pytorch-triton` and registered a placeholder on PyPI to claim the namespace.

Cheap, effective, and it scales poorly only in that you must remember to do it for every new internal package.

**b) Pin versions explicitly** for both direct and transitive dependencies, using lockfiles (`package-lock.json`, `poetry.lock`, `Pipfile.lock`). A pinned dependency does not get silently upgraded to an attacker's higher version number, which removes the mechanism the attack relies on.

**c) Use scoped or namespaced packages** (`@mycompany/utils` on npm), so internal names live in a namespace you control and cannot be squatted.

**d) Configure resolution explicitly.** Tell the package manager which registry serves which packages rather than allowing it to search both and pick. Most package managers support this and most teams have not configured it.

**e) Use an internal trusted package manager or proxy.** This is the strongest control: developers pull only from an internal repository containing vetted components, and the proxy decides what reaches them. It addresses dependency confusion, hallucinated packages and typosquatting with one mechanism.

## 3. Tooling

**`confused`** (by Visma Prodsec) is the reference tool for detecting exposure: it takes a dependency manifest and checks whether the named packages are unclaimed on public registries, showing you which of your internal names an attacker could still register.

## 4. Summary

1. Dependency confusion exploits name resolution, not a software vulnerability.
2. The attacker needs only the names of internal packages, which leak easily.
3. Register your internal names publicly as placeholders to deny them to attackers.
4. Pin versions and use lockfiles so a high public version cannot win.
5. Use scoped namespaces and explicitly configure which registry serves what.
6. An internal proxy with an allowlist is the strongest single control, and covers hallucinated packages too.
7. `confused` by Visma Prodsec identifies which of your internal names remain unclaimed.
