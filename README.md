# krateo-codegen-agents

Cross-cutting Krateo **codegen** specialist agents (kagent), federated on the `krateo-autopilot`
orchestrator via the `extraAgents` hook (reachable only through it). Unlike the per-component agents,
these are not tied to a Krateo component — they operate on arbitrary code.

| Chart | Agent |
|-------|-------|
| `charts/krateo-code-analysis-agent` | traces Krateo components to their GitHub source, searches code, checks commits for regressions |
| `charts/krateo-ansible-to-operator-agent` | converts Ansible playbooks/roles into Kubernetes operators |
| `charts/krateo-tf-provider-to-operator-agent` | translates Terraform providers into operators (or recommends ACK/Config Connector/ASO/Crossplane) |
| `charts/krateo-tf-to-helm-agent` | translates Terraform into Helm charts |

Each is a `kagent` Agent chart per the [/kagent standard](https://github.com/braghettos/krateo-autopilot/blob/main/AGENTS-VERSIONING.md);
its `Chart.yaml` `sources` points at this repo (its codebase), read via github MCP tools. All publish
to `oci://ghcr.io/braghettos/krateo/<agent>` on a semver tag.
