# Dependency Pinning

## 1. What it is

**Dependency pinning is specifying the exact version of every dependency rather than allowing a flexible version range.**

The effect: the same versions are installed every time the project is built, on every machine, in every environment.

```
   NOT PINNED                        PINNED
   ──────────                        ──────
   flask>=2.0            ← range     flask==2.0.2
   requests~=2.25        ← compatible requests==2.25.1
   "express": "^4.17.1"  ← caret     "express": "4.17.1"
                                     "lodash": "4.17.21"
```

The unpinned forms all mean "this version or something newer that ought to be compatible", and the decision about what you actually get is made at install time by the package manager, not by you at review time.

## 2. What it buys

1. **Consistency.** Identical builds across developer machines, CI and production. The class of bug that only appears in one environment largely disappears.
2. **Reproducibility.** A build from six months ago can be rebuilt exactly. Without this, incident investigation is guesswork.
3. **Security.** This is the part worth dwelling on, below.

## 3. The security argument

Three distinct security benefits, and they are often conflated:

**a) It removes the silent upgrade.** A range allows a new version to arrive without any human deciding to accept it. If that new version is malicious, whether through a compromised maintainer account, a hijacked package or dependency confusion, it installs itself. Pinning converts every dependency change into a deliberate, reviewable act.

**b) It defeats the version number trick in dependency confusion.** That attack relies on publishing a higher version number that resolution prefers. A pinned exact version has nothing to prefer.

**c) It makes what you scanned the thing you shipped.** Scanning results describe specific versions. If resolution can pick different versions at deploy time, your scan describes something you did not deploy.

## 4. The same principle in every ecosystem

The syntax varies; the principle is identical.

**Python** (`requirements.txt`):
```
flask==2.0.2
requests==2.25.1
```

**Node** (`package.json`):
```json
{
  "dependencies": {
    "express": "4.17.1",
    "lodash": "4.17.21"
  }
}
```

**PHP** (`composer.json`):
```json
{
  "require": {
    "monolog/monolog": "2.2.0",
    "guzzlehttp/guzzle": "7.3.0"
  }
}
```

**Java** (`pom.xml`):
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-core</artifactId>
        <version>5.3.8</version>
    </dependency>
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
        <version>2.12.3</version>
    </dependency>
</dependencies>
```

**And for AI artifacts**, the same principle applies to models. Every lab in this course pinned a `revision_id`, which is a commit hash pin on a model repository:

```python
model = AutoModelForCausalLM.from_pretrained(
    "microsoft/Phi-3-mini-4k-instruct",
    revision="0a67737cc96d2554230f90338b163bc6380a2a85",
)
```

Without it, `from_pretrained` pulls whatever `main` points to today, which may not be what you evaluated.

## 5. Pinning direct dependencies is not enough

Since six of seven vulnerabilities arrive transitively, pinning only what you named leaves most of the tree floating. **Lockfiles** solve this by recording the resolved version of every package in the tree, direct and transitive: `package-lock.json`, `poetry.lock`, `Pipfile.lock`, `yarn.lock`, `Cargo.lock`.

Commit the lockfile, and install from it in CI (`npm ci` rather than `npm install`).

## 6. The trade off, stated honestly

Pinning has a real cost: **you stop receiving security fixes automatically.** A pinned vulnerable version stays vulnerable until someone updates it.

Pinning is therefore only safe when paired with **active dependency management**: automated update tooling (Dependabot, Renovate) proposing version bumps as reviewable pull requests, and continuous vulnerability scanning telling you when a pinned version has become dangerous.

**Pinning without updating is not security, it is just a frozen risk profile.** The pairing is the control: pin so that changes are deliberate, then deliberately change.

## 7. Summary

1. Pinning specifies exact versions instead of ranges.
2. It buys consistency, reproducibility and security.
3. Security value: no silent upgrades, defeats the dependency confusion version trick, and makes scanned versions equal shipped versions.
4. Same principle across Python, Node, PHP, Java, and model revision hashes.
5. Pin transitively using lockfiles, since most exposure is transitive.
6. Pinning without an update process freezes your vulnerabilities in place; pair it with automated update tooling and scanning.
