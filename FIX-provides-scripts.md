# Fix: `provides.scripts requires a mapping` on bundle install

**Symptom**: `specify bundle install sketch-base` fails with
`provides.scripts requires a mapping`.

**Cause**: `provides.scripts` is not a valid field in `extension.yml`. The CLI
expects `scripts` to be a **mapping** keyed by script type (`sh`, `ps`, `py`), and
only inside a **command's markdown frontmatter**, not as a `provides` entry.

**Fix**: delete the `provides.scripts` block. Nothing replaces it. Script files
ship because they are in the extension directory; no manifest lists them.

Confirming evidence: the working `sketch` extension ships seven shell scripts
under `scripts/bash/` and declares none of them in its manifest.

Do **not** try to fix this by turning the list into a list of mappings
(`- file: <path>`). That is still not a mapping and fails the same way.

---

## Files changed

Five files. Only the first is required to fix the install; the rest align with the
convention that already works and prevent the error recurring.

| # | File | Lines | Required? |
|---|------|-------|-----------|
| 1 | `sketch-base/extensions/skt-compose/extension.yml` | 15 | **Yes, this is the fix** |
| 2 | `sketch-base/extensions/skt-compose/commands/compose.md` | 31 | Recommended |
| 3 | `sketch-langchain-deep-agent/extensions/skt-deepagent/commands/deep-agent-context.md` | 24 | Recommended |
| 4 | `sketch-base/.gitlab-ci.yml` | 39 | Recommended, prevents recurrence |
| 5 | `sketch-langchain-deep-agent/.gitlab-ci.yml` | 15 | Recommended, prevents recurrence |

No other file changes. No version bumps. No files added or deleted.

---

## 1. `sketch-base/extensions/skt-compose/extension.yml`

**This is the fix.** Delete the whole `scripts:` block under `provides:`.

```diff
 provides:
   commands:
     - name: "speckit.skt-compose.compose"
       file: "commands/compose.md"
       description: "[Compose] Merge every installed constitution layer and fill plan template slots, from the compose.yml declarations in this project. Normally invoked by a work-type extension's before_plan hook rather than by hand"
-  scripts:
-    - "scripts/lib/sketch_compose.py"
-    - "scripts/bash/sketch-compose.sh"
-    - "scripts/powershell/sketch-compose.ps1"
+
+# No provides.scripts key. The CLI expects `scripts` to be a MAPPING keyed by
+# script type (sh, ps, py), not a list of paths, and it is a per-command field
+# rather than a provides entry. Script files ship because they are in the
+# extension directory, not because a manifest lists them. The sketch extension
+# ships seven scripts and declares none of them.
+#
+# Commands invoke their scripts by path from the install root, as the sketch
+# commands do:
+#     SK=".specify/extensions/skt-compose"
+#     "$SK/scripts/bash/sketch-compose.sh" --json
 
 # No hooks. A work-type extension registers its own before_plan hook and calls
 # this command. Registering one here would fire in every project, including those
 # with no declarations to compose, and would make the hook impossible to order
 # relative to a work type's other hooks.
 hooks: {}
```

After this edit `provides:` has exactly one key, `commands`. The comment block is
optional but worth keeping so nobody adds the field back.

**Do not delete the script files themselves.** All three still ship:

```
extensions/skt-compose/scripts/lib/sketch_compose.py
extensions/skt-compose/scripts/bash/sketch-compose.sh
extensions/skt-compose/scripts/powershell/sketch-compose.ps1
```

---

## 2. `sketch-base/extensions/skt-compose/commands/compose.md`

Two hunks. Neither is needed to clear the install error; both bring this command in
line with the `sketch` commands, which are known to register correctly.

**Hunk A, frontmatter.** The original had `description` only. The working
`sketch` commands carry the full set.

```diff
 ---
-description: "Merge every installed constitution layer and fill plan template slots, from the compose.yml declarations in this project"
+name: "speckit.skt-compose.compose"
+description: "Merge every installed constitution layer and fill plan template slots, from the compose.yml declarations in this project."
+argument-hint: "[--extension <id>] [--check]"
+compatibility: "Requires a project initialised with specify init, and python3 on PATH."
+metadata:
+  author: "sketch"
+  source: "commands/compose.md"
+user-invocable: true
+disable-model-invocation: false
 ---
```

Note there is **no `scripts:` key here either.** The development guide documents a
frontmatter `scripts:` mapping with `{SCRIPT}` substitution, but that is for
reaching *core* spec-kit scripts. The `sketch` commands do not use it, and this one
follows them.

**Hunk B, invocation.** Resolve the install root first, matching
`sketch-specify.sh`'s `SK=` pattern, so the command fails with a clear message when
the extension is absent rather than with a path error.

```diff
 ## Outline
 
-1. **Run the composer** from the repository root:
+1. **Run the composer** from the repository root. Resolve the install root first,
+   then run the variant matching this project's script type:
 
-   - Bash: `.specify/extensions/skt-compose/scripts/bash/sketch-compose.sh --json`
-   - PowerShell: `.specify/extensions/skt-compose/scripts/powershell/sketch-compose.ps1 -json`
+   ```bash
+   SK=".specify/extensions/skt-compose"
+   [ -d "$SK" ] || { echo "skt-compose is not installed"; exit 1; }
+
+   "$SK/scripts/bash/sketch-compose.sh" --json
+   ```
+
+   ```powershell
+   $SK = ".specify/extensions/skt-compose"
+   if (-not (Test-Path $SK)) { Write-Error "skt-compose is not installed"; exit 1 }
+
+   & "$SK/scripts/powershell/sketch-compose.ps1" -Json
+   ```
 
-   Where only one variant is present, run that one.
+   Where only one variant is present, run that one. Both wrap the same Python
+   library, so their output is identical.
```

The last line replaces the original sentence rather than adding a second copy of
it. Check for a duplicated "Where only one variant is present" after applying.

---

## 3. `sketch-langchain-deep-agent/extensions/skt-deepagent/commands/deep-agent-context.md`

Same two hunks. This command calls `skt-compose`'s script, so the `SK=` guard also
gives a useful message when the composer is missing.

**Hunk A, frontmatter:**

```diff
 ---
-description: "Compose the deep agent constitution layer and plan sections, before the plan command reads either"
+name: "speckit.skt-deepagent.context"
+description: "Compose the deep agent constitution layer and plan sections, before the plan command reads either."
+compatibility: "Requires the skt-compose extension and python3 on PATH."
+metadata:
+  author: "sketch"
+  source: "commands/deep-agent-context.md"
+user-invocable: true
+disable-model-invocation: false
 ---
```

No `argument-hint`: this command takes no arguments of its own.

**Hunk B, invocation:**

```diff
 1. **Run the composer** from the repository root. Use the variant matching this
    project's script type:
 
-   - Bash: `.specify/extensions/skt-compose/scripts/bash/sketch-compose.sh --json`
-   - PowerShell: `.specify/extensions/skt-compose/scripts/powershell/sketch-compose.ps1 -json`
+   ```bash
+   SK=".specify/extensions/skt-compose"
+   [ -d "$SK" ] || { echo "skt-compose is not installed; this work type is inert without it"; exit 1; }
+
+   "$SK/scripts/bash/sketch-compose.sh" --json
+   ```
+
+   ```powershell
+   $SK = ".specify/extensions/skt-compose"
+   if (-not (Test-Path $SK)) { Write-Error "skt-compose is not installed; this work type is inert without it"; exit 1 }
+
+   & "$SK/scripts/powershell/sketch-compose.ps1" -Json
+   ```
 
    Where only one variant is present, run that one. Pass no `--extension` filter:
```

Note the path stays `.specify/extensions/skt-compose/`, not `skt-deepagent`. This
extension deliberately ships no scripts of its own.

---

## 4. `sketch-base/.gitlab-ci.yml`

**Why this matters.** The `manifests` job only inspected
`extensions/sketch/extension.yml`, which never had the bad field, so CI passed
while the real CLI rejected the install. This generalises it to walk every
extension and reject the invalid field.

In the `manifests` job, replace the single-extension block:

```diff
-      em = Path("extensions/sketch/extension.yml")
-      d = yaml.safe_load(em.read_text()) or {}
-      eid = (d.get("extension") or {}).get("id")
-      cmds = (d.get("provides") or {}).get("commands") or []
-      if not cmds:
-          errs.append("the extension provides no commands")
-      for c in cmds:
-          if not c["name"].startswith(f"speckit.{eid}."):
-              errs.append(f"command {c['name']!r} must start with 'speckit.{eid}.'")
-          if not (em.parent / c["file"]).is_file():
-              errs.append(f"command {c['name']!r} -> {c['file']} does not exist")
+      # Every extension in the repo, not just sketch: a second extension is where
+      # a manifest field the CLI rejects would otherwise slip through.
+      for em in sorted(Path("extensions").glob("*/extension.yml")):
+          d = yaml.safe_load(em.read_text()) or {}
+          eid = (d.get("extension") or {}).get("id")
+          if eid != em.parent.name:
+              errs.append(f"{em}: id is {eid!r} but the directory is {em.parent.name!r}")
+          provides = d.get("provides") or {}
+
+          # provides.scripts is NOT a valid manifest field. The CLI expects
+          # `scripts` to be a MAPPING keyed by script type (sh, ps, py), and only
+          # in a command's frontmatter. A list of paths under provides fails
+          # install with: "provides.scripts requires a mapping". Script files ship
+          # because they are in the extension directory; nothing lists them.
+          if "scripts" in provides:
+              errs.append(f"{em}: provides.scripts is not a valid field. Script files ship by being in the directory")
+          for key in provides:
+              if key not in ("commands", "templates", "memory"):
+                  errs.append(f"{em}: provides.{key} is not a recognised field")
+
+          cmds = provides.get("commands") or []
+          if not cmds:
+              errs.append(f"{em}: provides no commands")
+          for c in cmds:
+              if not c["name"].startswith(f"speckit.{eid}."):
+                  errs.append(f"{em}: command {c['name']!r} must start with 'speckit.{eid}.'")
+              if not (em.parent / c["file"]).is_file():
+                  errs.append(f"{em}: command {c['name']!r} -> {c['file']} does not exist")
```

Everything above and below this block, including the `bundle.yml` component
validation, is unchanged.

**The allowed-key list is a partial guess.** `commands` is confirmed by the working
`sketch` extension. `templates` is what both presets use in `provides` and has not
been confirmed against the CLI for an extension. `memory` is inferred from the
`sketch-constitution` preset. If a future extension legitimately provides something
else, widen the tuple rather than deleting the check.

---

## 5. `sketch-langchain-deep-agent/.gitlab-ci.yml`

Same guard, for the one extension this repo ships.

```diff
-      cmds = (d.get("provides") or {}).get("commands") or []
+      provides = d.get("provides") or {}
+
+      # provides.scripts is NOT a valid manifest field. The CLI expects `scripts`
+      # to be a MAPPING keyed by script type, and only in a command's frontmatter.
+      # A list under provides fails install with "provides.scripts requires a
+      # mapping". This extension ships no scripts at all, so the guard is here to
+      # stop one being added back.
+      if "scripts" in provides:
+          errs.append("provides.scripts is not a valid field, and this extension ships no scripts")
+      for key in provides:
+          if key not in ("commands", "templates", "memory"):
+              errs.append(f"provides.{key} is not a recognised field")
+
+      cmds = provides.get("commands") or []
       if not cmds:
           errs.append("the extension provides no commands")
```

---

## Verify

After applying, in order:

```bash
# 1. the manifest is now well formed
python3 -c "
import yaml
d = yaml.safe_load(open('extensions/skt-compose/extension.yml'))
keys = list(d['provides'].keys())
assert keys == ['commands'], keys
print('provides keys:', keys)
"

# 2. the install that was failing
specify bundle install sketch-base

# 3. nothing else regressed
python3 extensions/skt-compose/scripts/lib/selftest.py    # expect: passed 54, failed 0
bash extensions/sketch/scripts/bash/selftest.sh           # expect: 87 passed, 0 failed
```

Then confirm the composer still runs from its installed location:

```bash
SK=".specify/extensions/skt-compose"
"$SK/scripts/bash/sketch-compose.sh" --json
```

Expect `{"ok": true, ...}`. In a project with no `compose.yml` it reports
`nothing to compose`, which is correct rather than an error.

---

## If the install still fails

The next candidate is **`provides.templates` in the two preset manifests**:

```yaml
# presets/sketch-constitution/preset.yml
# presets/sketch-plan/preset.yml
provides:
  templates:
    - type: "template"
      name: "plan"
      file: "templates/plan-template.md"
```

That field and its `type` value were inferred, not confirmed against the CLI. If
the error moves to a preset and mentions `templates`, report the exact message
rather than guessing at a shape. Check a bundled preset in your spec-kit install
for the real schema.

Two things to avoid in that case:

- Do not delete `provides.templates` by analogy with `provides.scripts`. A preset
  has to declare what it provides, or nothing gets installed. A template file is
  not picked up by being in the directory the way a script is.
- Do not reshape it by trial and error against the install command. Find a working
  example first.
