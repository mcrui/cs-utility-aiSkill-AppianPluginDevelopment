# cs-utility-aiSkill-AppianPluginDevelopment

Source repository for the **`creating-appian-plugins`** AI skill — a reference for building
Appian plug-ins from scratch, including their build environments. Works with both
**Claude Code** (as a skill) and **Kiro** (as a steering file).

The skill covers smart services, expression functions, connected systems and integrations, custom
data types, servlets, and SAIL component plug-ins.

## Layout

```
.kiro/
  steering/
    creating-appian-plugins.md    Kiro steering file (fileMatch on *.gradle / appian-plugin.xml)
creating-appian-plugins/
  SKILL.md                        entry point: decision tree, traps, checklists
  references/
    build.gradle                  working build, verified end to end
    module-reference.md           every appian-plugin.xml module element
    java-patterns.md              working code per module type
    component-plugins.md          the ZIP artifact (a different system)
```

## Installation

### Claude Code (skill)

`~/.claude/skills/creating-appian-plugins` is a **directory junction** pointing at
`creating-appian-plugins/` in this repo. Edit here and the installed skill updates immediately —
there is no copy step to forget.

Verify the link:

```powershell
Get-Item ~\.claude\skills\creating-appian-plugins | Select-Object LinkType, Target
```

Recreate it if it is ever lost:

```powershell
Remove-Item ~\.claude\skills\creating-appian-plugins -Recurse -Force
New-Item -ItemType Junction `
  -Path   ~\.claude\skills\creating-appian-plugins `
  -Target ~\Projects\cs-utility-aiSkill-AppianPluginDevelopment\creating-appian-plugins
```

To install as a plain copy instead (e.g. on another machine):

```powershell
Copy-Item creating-appian-plugins ~\.claude\skills\ -Recurse
```

### Kiro (steering file)

Kiro uses **steering files** in `.kiro/steering/` to provide the same contextual guidance that
Claude Code gets from skills. This repo ships a ready-made steering file at
`.kiro/steering/creating-appian-plugins.md`.

**Option A — Work directly in this repo.** The steering file is already in place. Open this
project in Kiro and the guidance is active immediately.

**Option B — Add to another project.** Copy the steering directory into your target workspace:

```powershell
Copy-Item .kiro -Destination <your-project-root>\ -Recurse
```

Or create a symlink so edits here propagate automatically:

```powershell
# From your target project root
New-Item -ItemType Junction `
  -Path   .kiro\steering `
  -Target ~\Projects\cs-utility-aiSkill-AppianPluginDevelopment\.kiro\steering
```

**Option C — Reference files inline.** Kiro steering files support `#[[file:<path>]]` includes.
If you prefer a minimal steering file that points at the full references rather than embedding
them, create `.kiro/steering/creating-appian-plugins.md` in your project with:

```markdown
---
inclusion: manual
---
# Creating Appian Plug-ins
#[[file:creating-appian-plugins/SKILL.md]]
#[[file:creating-appian-plugins/references/module-reference.md]]
#[[file:creating-appian-plugins/references/java-patterns.md]]
#[[file:creating-appian-plugins/references/component-plugins.md]]
#[[file:creating-appian-plugins/references/build.gradle]]
```

The `inclusion: manual` front matter means the steering is available via the `#` context key in
chat rather than being always loaded — useful if you don't need Appian plug-in guidance on every
prompt.

#### Steering inclusion modes

| Mode | Front matter | Behaviour |
|---|---|---|
| Always (default) | *(none)* | Loaded into every Kiro prompt automatically |
| File match | `inclusion: fileMatch` + `fileMatchPattern: '*.gradle'` | Loaded when a matching file is read into context |
| Manual | `inclusion: manual` | Available via `#` in chat — the user opts in per prompt |

The shipped `.kiro/steering/creating-appian-plugins.md` uses `manual` inclusion so it's available
via `#` in chat but doesn't load on every prompt — pull it in when you're working on plug-in code.

## How this skill was built, and how to extend it

It was produced with the TDD-for-documentation cycle from `superpowers:writing-skills`. That matters
for future edits: **the content is grounded in observed failures, not in what seemed worth saying.**

- **RED** — three agents were given plug-in build tasks with no skill and no access to a reference
  implementation. Their mistakes became the skill's contents. The worst: one searched Maven Central
  by groupId, concluded the SDK was unobtainable, and wrote a *stub* of `AppianServlet` — code that
  compiles and can never deploy.
- **GREEN** — two agents ran the same tasks with the skill. Both produced working builds and the
  stubbing failure did not recur. They were asked to report what the skill got wrong.
- **REFACTOR** — every defect they found was verified independently, then fixed. Several were real:
  the `Simple*` connected-system classes are in `connected-systems-client`, not `-core`; the Gradle
  toolchain needs the foojay resolver or it only works on machines that already have a JDK 17.

### If you change the skill

1. Make the change in this repo.
2. Re-verify `references/build.gradle` actually builds — see below. A build file in a skill that does
   not run is worse than none.
3. Ideally, hand the changed skill to a fresh agent with a realistic task and ask it to report what
   was wrong, missing or ambiguous. That is what surfaced every defect fixed so far.

### Verifying the reference build

```bash
mkdir -p /tmp/t/src/main/{java/com/x,resources} && cd /tmp/t
cp <repo>/creating-appian-plugins/references/build.gradle .
# settings.gradle MUST include the foojay resolver plugin — see SKILL.md
gradle wrapper --gradle-version 9.6.1
./gradlew clean build --warning-mode all
```

Expect three green guard tasks, and **no** "installed via auto-provisioning without toolchain
repositories" warning. Also negative-test the checks — a verifier that cannot fail proves nothing:
point a `class=` in the descriptor at a class that does not exist and confirm the build fails.

## Provenance and confidence

- Smart services, expression functions, the packaging contract and the Gradle build are grounded in a
  plug-in running in production on Appian 26.3, and were re-verified by building.
- Connected-system class locations, package names and method signatures were recovered by unzipping
  and decompiling the published Maven artifacts.
- Descriptor module elements come from current Appian documentation.
- **`component-plugins.md` was assembled from documentation only** and has never been validated
  against a deployed component. It is flagged as such in the file.
- Nothing here has been validated against a running Appian server. Descriptor acceptance and
  deployment behaviour remain unproven; the build-time checks are a substitute for deployment, not
  proof of it.

## Known open questions

- Function i18n bundle filename: docs say name it after the category class, working plug-ins name it
  after the function key. The skill currently says ship both. Settle this on a real server and the
  guidance can be simplified.
- Whether `majorVersion` on `@TemplateId` behaves the same for connected system templates as for
  integration templates.
- Whether component-plugin `event` parameters take a single value or a Value/SaveInto pair — Appian's
  own pages disagree.
