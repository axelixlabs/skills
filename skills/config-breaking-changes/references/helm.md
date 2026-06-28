# Helm chart values — where to look and what changes mean

Use this when the diff touches the Axelix Helm chart. Helm has no built-in "deprecation"
annotation for values, so the same four rules (in `SKILL.md`) are enforced through
conventions: `values.yaml` comments, `values.schema.json`, template `default`/`coalesce`
fallbacks, and explicit warnings. This file maps the rules onto those mechanics.

## Where values are defined and consumed

- **`values.yaml`** — the source of truth for every value, its **default**, and (by comment)
  its documentation and deprecation status. A value's place in the YAML tree is its key path
  (e.g. `discovery.auto.enabled`).
- **`values.schema.json`** — JSON Schema for values: declares **types**, enums, required keys,
  and constraints. Treat a `type`/`enum`/`required` change here as a contract change.
- **`templates/`** — where `.Values.<path>` is consumed. Fallback/precedence logic lives here.
- **`Chart.yaml`** — `version` (chart version) and `appVersion`. Use these to decide whether
  the change crosses a **major** boundary, which Rules 2 and 4 require.
- **`README.md` / values table / `NOTES.txt`** — operator-facing docs and upgrade notes.

A value is "removed" when its key disappears from `values.yaml` **and** stops being referenced
in templates. Renaming a key path is a rename; dropping a key is a removal.

## Rule 1 (rename) in Helm

The introducing PR keeps both keys, with the **new key taking precedence and falling back to the
old**. Encode precedence in the template, not just in docs:

```yaml
# new wins, falls back to the deprecated key
instanceNamePrefix: {{ .Values.instanceNamePrefix | default .Values.instanceName | quote }}
# or, equivalently:
{{ coalesce .Values.instanceNamePrefix .Values.instanceName }}
```

Verify the order: `default`/`coalesce` must yield the **new** value when set and fall back to the
**old** one — not the reverse. An operator who only set the old key must still get a working render.

## "Deprecation" markers (Rules 1 & 2)

Because Helm has no native mechanism, a value is properly deprecated only when the deprecation is
**visible to operators**. Look for, and require, a combination of:

- A clear comment in `values.yaml` on the key, e.g.
  `# DEPRECATED: use instanceNamePrefix instead. Removed in chart v3.0.0.`
- The **alternative is stated** — the replacement key, the new bundle of keys, or an explanation
  that the value is obsolete (Rule 2's "communicate the alternative").
- Ideally an active warning when the deprecated key is set, so silent users find out:
  ```yaml
  {{- if .Values.instanceName }}
  {{- fail "instanceName is deprecated; use instanceNamePrefix (see README)" }}  # hard stop
  {{- end }}
  ```
  or a softer notice appended in `NOTES.txt`. A hard `fail` is itself breaking, so reserve it for
  the *removal* step, not the grace-period step.
- Marking it in `values.schema.json` (e.g. a `description`/`deprecated` annotation) when a schema
  is maintained.

For **Rule 2 removal**, confirm via `git log`/`git blame` on `values.yaml` that the deprecation
comment landed in an **earlier released chart version**, and that this PR bumps the chart's
**major** `version` in `Chart.yaml`. A key deprecated and deleted in the same PR is a violation.

## Rule 3 (type/format) in Helm

A type/format change is a violation when a value that operators already set could now render
differently or fail:

- `values.schema.json` changes a key's `type`, tightens a constraint, or removes/renames an
  `enum` value.
- A template starts coercing or parsing a value differently (e.g. was used as a raw string, now
  `int`/`toYaml`/`fromJson`-parsed; a scalar becomes a list/map).
- A previously free-form value is now matched against a fixed set.

The compliant path mirrors Spring Boot: introduce a **new key** that accepts the new type/format
(e.g. `heartbeatIntervalText`) and run the Rule 1 migration with precedence + fallback.

## Rule 4 (default) in Helm

The default is the value literally shipped in `values.yaml` (and any `| default <x>` in templates
that supplies an effective default when the key is unset). Changing it changes behavior for every
operator who never overrode it — allowed only at a major chart bump, always a breaking change to be
labelled. Adding a brand-new key with a default does **not** count (nobody depended on it yet).
