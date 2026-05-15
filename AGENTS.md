# AGENTS.md

This file provides guidance to coding agents (e.g. Claude Code, claude.ai/code) when working with code in this repository.

## Repository purpose

Go module `open-cluster-management.io/managed-serviceaccount` — an OCM (Open Cluster Management) addon built on `addon-framework`. The hub defines a `ManagedServiceAccount` resource (a request for an OCM-controlled ServiceAccount in a managed cluster); the agent on each managed cluster ensures the corresponding `ServiceAccount` exists, issues a long-lived token via the `TokenRequest` API, and ships the token back to the hub as a `Secret`.

Three use cases (from `README.md`):
1. Ensure a ServiceAccount exists on a managed cluster without holding a kubeconfig to it.
2. Get a token from the hub for that ServiceAccount so the hub can call the managed cluster's API.
3. Homogenize client identity when the hub talks to many managed clusters.

Two binaries: `cmd/manager/` (hub-side OCM addon manager) and `cmd/agent/` (spoke-side controller). Both produced from the same `Dockerfile`.

Fork is mirrored to `kluster-management/managed-serviceaccount`; **upstream is `open-cluster-management-io/managed-serviceaccount`** and this repo tracks it.

## Architecture

- `cmd/manager/` — hub-side entry point. Registers the addon with OCM via `addon-framework` and reconciles `ManagedServiceAccount` against per-cluster `ManagedClusterAddOn` state.
- `cmd/agent/` — spoke-side entry point. Watches `ManagedServiceAccount` on the hub (via `ManifestWork` projection), creates/refreshes the local `ServiceAccount`, mints a token, and writes it back as a `Secret`.
- `apis/authentication/`:
  - `v1alpha1/` — `ManagedServiceAccount` (legacy).
  - `v1beta1/` — current API version. Each version has `*_types.go` (hand-written) and generated `zz_generated.*.go`.
- `pkg/addon/`:
  - `manager/` — addon manager glue (templated agent manifests, RBAC, healthcheck).
  - `agent/` — agent-side runtime helpers.
  - `commoncontroller/` — controllers shared between manager and agent.
- `pkg/controllers/event/` — event reconciler.
- `pkg/features/features.go` — feature gates.
- `pkg/common/constants.go` — finalizer / label / annotation strings (user contract).
- `pkg/util/namespace.go` — addon-installation namespace helpers.
- `pkg/generated/` — generated typed clientset/listers/informers. Do not hand-edit.
- `charts/managed-serviceaccount/` — Helm chart for the install.
- `config/` — kustomize bases (`config/crd`, `config/default`, `config/rbac`, …).
- `deploy/` — bundled deployment manifests.
- `e2e/`:
  - `e2e.go`, `e2e_test.go`, `framework/` — harness.
  - `install/`, `token/`, `ephemeral_identity/` — suite buckets.
- `Dockerfile` — single multi-stage build that emits both manager and agent layers.
- `Makefile` — Kubebuilder-style harness with a local Go toolchain (no AppsCode Docker wrapper). Installs `controller-gen` / `kustomize` / `client-gen` into `bin/` on first use.
- `PROJECT` — Kubebuilder project metadata.

## Common commands

This repo uses a **local Go toolchain**, not the AppsCode Docker harness.

- `make build` (alias `make all`) — `generate fmt vet build-manager build-agent`.
- `make build-manager` / `make build-agent` — build a single binary.
- `make generate` — controller-gen DeepCopy generation.
- `make manifests` — controller-gen CRDs/RBAC/webhooks.
- `make client-gen` — regenerate `pkg/generated/`.
- `make fmt`, `make vet` — standard.
- `make test` — `manifests generate fmt vet`, then Go tests.
- `make test-integration` — controller-runtime envtest integration tests.
- `make run` — run a controller against `~/.kube/config` locally.
- `make install` / `make uninstall` — `kustomize` apply/remove CRDs.
- `make deploy` / `make undeploy` — kustomize-deploy the full stack.
- `make images` — build the all-in-one image from `Dockerfile`.
- `make controller-gen` / `make kustomize` — install the tools into `bin/`.
- `make help` — list all targets.

Run a single Go test:

```
go test ./pkg/addon/manager/... -run TestName -v
```

End-to-end suite:

```
go test ./e2e/... -v
```

## Conventions

- Module path is `open-cluster-management.io/managed-serviceaccount` (**upstream**). Imports must use that.
- **Upstream-tracking** fork (mirrored as `kluster-management/managed-serviceaccount`). Prefer rebasing onto upstream over diverging; isolate AppsCode-only patches.
- License: Apache-2.0 (`LICENSE`).
- Sign off commits (`git commit -s`); contributions follow the DCO (`DCO`, `CONTRIBUTING.md`).
- CRD API group is `authentication.open-cluster-management.io`; versions `v1alpha1` and `v1beta1` coexist (`v1beta1` is current, `v1alpha1` retained for compat).
- Do not hand-edit `zz_generated.*.go` or anything under `pkg/generated/` — change `apis/authentication/<v>/*_types.go` and re-run `make generate manifests client-gen`.
- Two binaries from one Dockerfile (manager + agent). Don't conflate their RBAC or runtime surfaces — manager runs on the hub, agent runs on each managed cluster.
- The `ManagedServiceAccount`-to-token-`Secret` round-trip is the project's headline feature; preserve that flow when changing any of `pkg/addon/agent/`, `pkg/addon/manager/`, or `pkg/controllers/event/`.
