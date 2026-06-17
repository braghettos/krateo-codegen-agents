# Krateo output conventions

The codegen agents do not just emit generic Kubernetes YAML — when the output is meant to install as
a **Krateo composition**, it must follow the Krateo chart conventions below. This page is the output
contract: it is what `tf-to-helm`, `ansible-to-operator`, and `tf-provider-to-operator` must produce
(and what `code-analysis` reads when it traces a chart). It is grounded in the AGENTS-VERSIONING
standard and in what the agent prompts already instruct; for the live, version-correct detail read
an existing Krateo chart (e.g. `braghettos/krateo-snowplow-chart`) via the github tools.

## 1. A Krateo component = a chart + a CompositionDefinition

A Krateo-installable component is a Helm chart published to the single OCI registry
`oci://ghcr.io/braghettos/krateo/<name>` on a semver tag, plus a **`compositiondefinition.yaml`**
that registers the chart as a Krateo composition. The CompositionDefinition points at the OCI chart
and its version; `CompositionDefinition.spec.chart.version` is the cluster-observable deployed
version (and equals the chart's git tag, since `Chart.yaml` ships `version: CHART_VERSION`
substituted at release).

Generated charts should slot into this model: a self-contained chart that the installer can pin and
wire, not a one-off `kubectl apply` bundle.

## 2. `values.schema.json` is mandatory

The Krateo core-provider **requires** a `values.schema.json` next to `values.yaml`. Every generated
chart must ship one that fully types the values surface:

- type/`enum`/`required` for every value the templates read (and for nested objects);
- defaults that match `values.yaml`;
- no values referenced in templates/helpers that are missing from the schema, and no schema fields
  the chart never reads.

A chart without a conformant `values.schema.json` will not reconcile as a composition.

## 3. The §-style chart standard (shape)

Generated charts follow the standard Krateo chart shape:

```
<chart>/
  Chart.yaml              # version: CHART_VERSION, appVersion: CHART_VERSION (CI-stamped, tag-driven)
                          # sources: the braghettos repo(s) that own the code + chart
  values.yaml             # the tunable surface — block YAML only here, not scattered in templates
  values.schema.json      # mandatory (§2)
  templates/
    _helpers.tpl          # name/fullname/labels helpers; reuse via {{ include }}
    *.yaml               # native Kubernetes resource templates
  compositiondefinition.yaml   # → oci://ghcr.io/braghettos/krateo/<name>
  README.md
```

Conventions the templates must respect (these are the same best practices the `tf-to-helm` prompt
enforces):

- **Helper-based labels and names** via `_helpers.tpl`; reuse with `{{ include }}`.
- **`{{- toYaml }}`** for nested objects/maps; `{{ .Release.Namespace }}` instead of hardcoded
  namespaces.
- **Never hardcode secrets** — Secret data goes through values (`{{ .Values.x | b64enc }}`), and
  sensitive inputs are surfaced as `--set`/Secret-backed values.
- **Block YAML lives in `values.yaml`**, kept out of templates; CRDs (when a component owns one) go
  in a dedicated CRD sub-chart, never bundled into the app chart.
- **Faithful translation** — preserve every resource/field; document (don't silently drop) anything
  that has no Kubernetes equivalent (e.g. an `aws_rds_instance` becomes a documented external
  dependency with its endpoint exposed as a value).

## 4. Operator output (ansible / tf-provider paths)

When the right answer is a **controller/operator** rather than a chart, the conventions shift to the
operator project shape but the same grounding rules hold:

- **CRDs**: camelCase field names, PascalCase `Kind`, a meaningful `group`/`kind`/`version`, OpenAPI
  v3 validation on `spec`, and a `status` with conditions (Ready/Synced/Error). One CRD per logical
  resource.
- **Project layout**: the upstream-canonical layout for the chosen path — Operator SDK Ansible
  Operator (`watches.yaml`, `roles/`, `config/crd/bases/`, `config/rbac/role.yaml`, Dockerfile,
  Makefile) for the Ansible path; a Go Operator SDK controller (reconcile, finalizer,
  `status.atProvider`, RBAC, a `ProviderConfig` CRD referencing a Secret) for the TF-provider path.
- **Prefer existing operators**: the `tf-provider-to-operator` agent recommends a vendor-native
  operator (ACK / Config Connector / ASO) or Crossplane/Upjet before generating; a generated
  operator is the last resort, only for resources nothing existing covers.
- **Credentials and lifecycle**: always cover credentials (Secret-backed `ProviderConfig`); map TF
  `lifecycle`/`depends_on` to finalizer behaviour and owner/cross-resource references.

## 5. Versioning & provenance (what the generated artifact inherits)

- **`Chart.yaml`** uses `version: CHART_VERSION` and `appVersion: CHART_VERSION` (CI-stamped at the
  release tag) — no literal versions — so the tag is the single source of version truth and
  `docs/`-at-tag stays version-honest.
- **`sources`** lists the braghettos repo(s) that own the code and the chart, so a downstream agent
  (e.g. `code-analysis`) can ground in the real source via the github MCP.
- **Single registry**: publish to `oci://ghcr.io/braghettos/krateo/<name>` with the canonical
  `lint.yaml` + `release-oci.yaml` CI.

For the authoritative, version-correct shape, read a current Krateo chart via the github tools rather
than trusting this summary — see [llms.txt](llms.txt) for how to fetch at the deployed tag, and the
[AGENTS-VERSIONING standard](https://github.com/braghettos/krateo-autopilot/blob/main/AGENTS-VERSIONING.md)
(§7/§8) for the agent-chart contract these four agents themselves satisfy.
