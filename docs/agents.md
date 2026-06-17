# The codegen + knowledge agents

This repo packages five cross-cutting specialist agents (four codegen/IaC + one docs/grounding). Each is a
`kagent.dev/v1alpha2` `Agent` (chart `templates/agent.yaml`), with its behaviour spec in
`charts/<name>/files/prompts-{eng,ita}.yaml` and its model/tools wired in the chart. All four:

- reference a shared **ModelConfig by name** — `gemini-pro` (the heavy/reasoning tier, per
  AGENTS-VERSIONING §8 C3; codegen is synthesis-heavy), `modelConfig.create: false`;
- ground via the **github MCP server** (`RemoteMCPServer` `github-mcp-server`) — they read the real
  source before asserting, and never guess;
- are federated on `krateo-autopilot` via `extraAgents` and reachable only through the orchestrator
  (no standalone entry point).

Everything below is grounded in each chart's `templates/agent.yaml` and `files/prompts-eng.yaml`.

---

## krateo-docs-agent

Keeps Krateo Autopilot's grounding source current from the official docs.

- **Inputs:** a request to refresh/check the autopilot's grounding (optionally a specific topic).
- **What it does:** reads the official Krateo docs from the `krateoplatformops` org and the current
  `braghettos/krateo-autopilot` `docs/llms.txt`, **reconciles upstream against the braghettos fork**
  (upstream wins only where it doesn't contradict the fork — e.g. core-provider is always-on bootstrap,
  `composableoperations` is an engine-present marker, base install = portal+operations+starter), and
  proposes the update as a **pull request** to `braghettos/krateo-autopilot` (new branch + PR, never a
  direct commit — the PR is the human coherence gate).
- **Outputs:** a PR updating `docs/llms.txt`, summarizing the change and citing its sources.
- **Tools:** `github-mcp-server` → read (`get_file_contents`, `search_code`, `search_repositories`,
  `list_commits`) **and** propose (`create_branch`, `create_or_update_file`, `create_pull_request`).
  Requires the github MCP server running with a PAT scoped to read public repos + open PRs on
  `braghettos/krateo-autopilot`.

## krateo-code-analysis-agent

Traces a Krateo workload back to its GitHub source and finds where a bug lives.

- **Inputs:** a pod/component name or an observed error/symptom (optionally cluster telemetry).
- **What it does:** resolves the component to a concrete `owner/repo` (1. the managing Helm release →
  its chart `Chart.yaml` `sources`; 2. the image convention `ghcr.io/<org>/<name>` →
  `github.com/<org>/<name>`; 3. repository search by name), then reads the repo tree, searches the
  code for the error (stripping timestamps/UUIDs first, trying both the literal string and key
  function/variable names), reads the implicated files, and lists recent commits to spot a
  regression. Primary orgs: `krateoplatformops`, `krateoplatformops-blueprints`,
  `krateoplatformops-test`, `braghettos`.
- **Outputs:** the originating repo + file/lines, the likely root cause, and the regressing commit
  when applicable — with the repo/path it relied on cited.
- **Tools (read-only):** `github-mcp-server` → `get_file_contents`, `search_code`,
  `search_repositories`, `list_commits`. (`kagent-tool-server` when wired, to resolve a pod to its
  Helm release/chart metadata.) This agent does **not** write back.

## krateo-ansible-to-operator-agent

Converts Ansible playbooks/roles into Kubernetes operators via the **Operator SDK Ansible Operator**
path (never the Go or Helm operator path).

- **Inputs:** a repo/path to an Ansible playbook or role.
- **What it does:** reads the role (tasks, handlers, vars, templates, dependencies); designs the CRD
  (`spec` fields from the playbook's variables with types/required/defaults/enums, a `status` with
  conditions, a meaningful `group`/`kind`/`version`); generates the Operator SDK Ansible Operator
  project (`watches.yaml`, the adapted role, `config/crd/bases/`, `config/rbac/role.yaml`, a
  Dockerfile on the Ansible base image, a Makefile); adapts tasks for the reconcile context (drops
  `become`/`hosts`, scopes with `ansible_operator_meta`, prefers `kubernetes.core.k8s`, ensures
  idempotency). Field names camelCase, Kind PascalCase. Preserves the original logic — never
  simplifies or skips tasks; records Galaxy deps in `requirements.yml`.
- **Outputs:** the operator project files, an example CR, and the
  `operator-sdk init --plugins=ansible` / `make docker-build` / `make deploy` steps.
- **Tools:** `github-mcp-server` → reads `get_file_contents`, `search_code`, `search_repositories`;
  **writes** `create_or_update_file`, `push_files` — writes mutate a repo and require explicit user
  confirmation first.

## krateo-tf-provider-to-operator-agent

Brings Terraform-provider functionality into Kubernetes — **router first, generator second**:
recommend an existing operator before generating a new one.

- **Inputs:** the Terraform provider + the specific resources to bring into Kubernetes.
- **What it does:** enforces a strict priority order — (1) vendor-native operator
  (AWS→ACK, GCP→Config Connector, Azure→ASO), (2) Crossplane/Upjet only if the native operator lacks
  the specific resource, (3) generate a new operator only if neither covers it. Verifies coverage
  (the operator's CRDs/source + the cluster's installed API resources) before recommending. When
  routing native it shows the TF-resource→CRD-kind mapping, an equivalent CR, and the install
  command; when generating it emits one CRD per TF resource (`spec` from schema arguments, `status`
  from computed attributes, `spec.providerConfigRef`), a Go Operator SDK controller (Ready/Synced/
  Error conditions, finalizer, `status.atProvider`), RBAC, and a ProviderConfig CRD referencing a
  Secret — always covering credentials and always with a TF-resource→CRD→API-group table.
- **Outputs:** a recommendation (with mapping + install command) or a generated operator project, in
  both cases with the coverage mapping table.
- **Tools:** `github-mcp-server` → reads `get_file_contents`, `search_code`, `search_repositories`;
  **writes** `create_or_update_file`, `push_files` (confirm before any write).
  (`kagent-tool-server` when wired, to list available API resources for coverage checks.)

## krateo-tf-to-helm-agent

Converts Terraform modules into Helm charts that deploy **native Kubernetes resources** (no CRDs, no
external operators).

- **Inputs:** a repo/path to a Terraform module.
- **What it does:** reads the module (`variables.tf` → `values.yaml`; `main.tf`/resource files →
  templates; `outputs.tf` → `NOTES.txt`; `locals.tf` → `_helpers.tpl`); classifies resources —
  TF Kubernetes-provider resources (`kubernetes_deployment`/`_service`/`_config_map`/… and `_v1`
  variants, `kubernetes_manifest`) map directly to templates, while non-Kubernetes resources (e.g.
  `aws_rds_instance`) are documented in `NOTES.txt` as external dependencies (never silently
  dropped) with their connection string/endpoint surfaced as a `values.yaml` parameter. Generates
  the chart following Helm best practices (helper-based labels, `{{ include }}` reuse, `{{- toYaml }}`
  for nested objects, `.Release.Namespace`, never hardcoded secrets), translating `var.x`→
  `{{ .Values.x }}`, `count`/`for_each`→`range`, `dynamic`→`if`/`range`, and handling edge cases
  (`helm_release`→chart dependency, `null_resource`→Helm hook Job, etc.). Preserves every field
  faithfully.
- **Outputs:** the Helm chart (`Chart.yaml`, `values.yaml`, `templates/`, `_helpers.tpl`,
  `NOTES.txt`, a basic Helm test), the install command, an example values file, and a list of
  anything that could not be converted.
- **Tools:** `github-mcp-server` → reads `get_file_contents`, `search_code`, `search_repositories`;
  **writes** `create_or_update_file`, `push_files` (confirm before any write).

---

See [conventions.md](conventions.md) for the Krateo output conventions every generated chart/operator
must satisfy, and [llms.txt](llms.txt) for how to fetch these docs at the deployed version's tag.
