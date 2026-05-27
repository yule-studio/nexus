---
title: "Infra / DevOps / CLI 도구 — Examples"
kind: knowledge
status: current
created_at: 2026-05-27T12:25:00+09:00
tags: [resources, examples, infra, devops, cli]
home_hub: resources
---

# Infra / DevOps / CLI 도구 — Examples

**[[examples|↑ Examples hub]]**

## 1. Kubernetes 생태계

- ⭐ **kubernetes/kubernetes** — Go production 정통 — [github.com/kubernetes/kubernetes](https://github.com/kubernetes/kubernetes) 🏗️ 📖
- ⭐ **kubernetes/sample-controller** — Operator 개발 학습 — [github.com/kubernetes/sample-controller](https://github.com/kubernetes/sample-controller) 🎓
- **operator-sdk examples** — [github.com/operator-framework/operator-sdk](https://github.com/operator-framework/operator-sdk)
- **kubebuilder examples** — [github.com/kubernetes-sigs/kubebuilder](https://github.com/kubernetes-sigs/kubebuilder)
- ⭐ **karpenter** — Go Cluster autoscaler — [github.com/aws/karpenter-provider-aws](https://github.com/aws/karpenter-provider-aws) 🏗️
- **argo-cd** — Go GitOps controller — [github.com/argoproj/argo-cd](https://github.com/argoproj/argo-cd) 🏗️
- **flux2** — GitOps — [github.com/fluxcd/flux2](https://github.com/fluxcd/flux2)
- **istio** — Service Mesh — [github.com/istio/istio](https://github.com/istio/istio) 🏗️
- **linkerd2** — Service Mesh (Rust + Go) — [github.com/linkerd/linkerd2](https://github.com/linkerd/linkerd2)

## 2. Container / Runtime

- ⭐ **containerd** — container runtime — [github.com/containerd/containerd](https://github.com/containerd/containerd) 🏗️
- **runc** — OCI runtime — [github.com/opencontainers/runc](https://github.com/opencontainers/runc)
- **firecracker** — microVM (Rust) — [github.com/firecracker-microvm/firecracker](https://github.com/firecracker-microvm/firecracker) 🏗️
- **docker (Moby)** — [github.com/moby/moby](https://github.com/moby/moby) 🏗️
- **podman** — daemonless container — [github.com/containers/podman](https://github.com/containers/podman)
- **kaniko** — in-container image build — [github.com/GoogleContainerTools/kaniko](https://github.com/GoogleContainerTools/kaniko)

## 3. Observability

- ⭐ **prometheus** — Go metric — [github.com/prometheus/prometheus](https://github.com/prometheus/prometheus) 🏗️ 📖
- **grafana** — TS + Go — [github.com/grafana/grafana](https://github.com/grafana/grafana) 🏗️
- **loki** — log aggregation — [github.com/grafana/loki](https://github.com/grafana/loki)
- **tempo** — distributed tracing — [github.com/grafana/tempo](https://github.com/grafana/tempo)
- **mimir** — Prometheus 호환 metric — [github.com/grafana/mimir](https://github.com/grafana/mimir)
- **jaeger** — distributed tracing — [github.com/jaegertracing/jaeger](https://github.com/jaegertracing/jaeger)
- **opentelemetry-collector** — OTel — [github.com/open-telemetry/opentelemetry-collector](https://github.com/open-telemetry/opentelemetry-collector) 🏗️
- **fluent-bit** — log forwarder (C) — [github.com/fluent/fluent-bit](https://github.com/fluent/fluent-bit) 🏗️
- **vector** — log/metric (Rust) — [github.com/vectordotdev/vector](https://github.com/vectordotdev/vector) 🆕

## 4. IaC / Config

- ⭐ **terraform** — HCL + Go — [github.com/hashicorp/terraform](https://github.com/hashicorp/terraform) 🏗️
- **opentofu** — Terraform fork — [github.com/opentofu/opentofu](https://github.com/opentofu/opentofu) 🆕
- **pulumi** — TS/Python/Go — [github.com/pulumi/pulumi](https://github.com/pulumi/pulumi)
- **crossplane** — K8s native IaC — [github.com/crossplane/crossplane](https://github.com/crossplane/crossplane)
- **terragrunt** — Terraform wrapper — [github.com/gruntwork-io/terragrunt](https://github.com/gruntwork-io/terragrunt)
- **ansible** — Python — [github.com/ansible/ansible](https://github.com/ansible/ansible) 🏗️

### Terraform 모듈 / examples
- **terraform-aws-modules** — verified AWS modules — [github.com/terraform-aws-modules](https://github.com/terraform-aws-modules)

## 5. CI/CD / Build

- **jenkins** — Java legacy + 새 시도 — [github.com/jenkinsci/jenkins](https://github.com/jenkinsci/jenkins) 🏗️
- **woodpecker** — Drone fork — [github.com/woodpecker-ci/woodpecker](https://github.com/woodpecker-ci/woodpecker)
- **tekton** — K8s native CI — [github.com/tektoncd/pipeline](https://github.com/tektoncd/pipeline)
- ⭐ **act** — GitHub Actions local — [github.com/nektos/act](https://github.com/nektos/act)
- **bazel** — Google build — [github.com/bazelbuild/bazel](https://github.com/bazelbuild/bazel) 🏗️
- **nx** — TS monorepo — [github.com/nrwl/nx](https://github.com/nrwl/nx)
- **turbo** — TS monorepo — [github.com/vercel/turborepo](https://github.com/vercel/turborepo)
- **moonrepo** — modern monorepo — [github.com/moonrepo/moon](https://github.com/moonrepo/moon)

## 6. Service Mesh / Proxy / Gateway

- ⭐ **envoy** — C++ proxy 정통 — [github.com/envoyproxy/envoy](https://github.com/envoyproxy/envoy) 🏗️
- **nginx** — C — [github.com/nginx/nginx](https://github.com/nginx/nginx) 🏗️
- **HAProxy** — C — [github.com/haproxy/haproxy](https://github.com/haproxy/haproxy) 🏗️
- **traefik** — Go reverse proxy — [github.com/traefik/traefik](https://github.com/traefik/traefik)
- **caddy** — Go web server / proxy — [github.com/caddyserver/caddy](https://github.com/caddyserver/caddy) 🆕
- **Kong** — API Gateway — [github.com/Kong/kong](https://github.com/Kong/kong)
- **APISIX** — API Gateway — [github.com/apache/apisix](https://github.com/apache/apisix)

## 7. CLI 도구

### Rust CLI 정통
- ⭐ **ripgrep (rg)** — [github.com/BurntSushi/ripgrep](https://github.com/BurntSushi/ripgrep) 📖
- ⭐ **fd** — [github.com/sharkdp/fd](https://github.com/sharkdp/fd)
- **bat** — [github.com/sharkdp/bat](https://github.com/sharkdp/bat)
- **eza** — [github.com/eza-community/eza](https://github.com/eza-community/eza)
- **zoxide** — [github.com/ajeetdsouza/zoxide](https://github.com/ajeetdsouza/zoxide)
- **starship** — prompt — [github.com/starship/starship](https://github.com/starship/starship)
- **delta** — git diff — [github.com/dandavison/delta](https://github.com/dandavison/delta)
- **lazygit** — TUI git — [github.com/jesseduffield/lazygit](https://github.com/jesseduffield/lazygit)

### Go CLI
- **fzf** — fuzzy finder — [github.com/junegunn/fzf](https://github.com/junegunn/fzf) 🏗️
- **jq** — JSON CLI (C) — [github.com/jqlang/jq](https://github.com/jqlang/jq) 📖
- **yq** — YAML CLI — [github.com/mikefarah/yq](https://github.com/mikefarah/yq)
- **gh** — GitHub CLI — [github.com/cli/cli](https://github.com/cli/cli) 🏗️

## 8. Editor / IDE

- ⭐ **neovim** — C/Lua — [github.com/neovim/neovim](https://github.com/neovim/neovim) 🏗️
- **VS Code (open source)** — TS — [github.com/microsoft/vscode](https://github.com/microsoft/vscode) 🏗️
- **helix** — Rust modal editor — [github.com/helix-editor/helix](https://github.com/helix-editor/helix) 🆕
- **zed** — Rust editor 🆕 — [github.com/zed-industries/zed](https://github.com/zed-industries/zed) 🏗️
- **LazyVim** — Neovim config — [github.com/LazyVim/LazyVim](https://github.com/LazyVim/LazyVim) 🎓
- **AstroNvim** — Neovim config — [github.com/AstroNvim/AstroNvim](https://github.com/AstroNvim/AstroNvim)

## 9. Database 도구

- **dbeaver** — DB GUI (Java) — [github.com/dbeaver/dbeaver](https://github.com/dbeaver/dbeaver)
- **adminer** — single-file DB UI — [github.com/vrana/adminer](https://github.com/vrana/adminer)
- **flyway** — migration — [github.com/flyway/flyway](https://github.com/flyway/flyway)
- **liquibase** — migration — [github.com/liquibase/liquibase](https://github.com/liquibase/liquibase)
- **schemaspy** — DB diagram — [github.com/schemaspy/schemaspy](https://github.com/schemaspy/schemaspy)
- **atlas** 🆕 — modern schema migration — [github.com/ariga/atlas](https://github.com/ariga/atlas)

## 10. Secret / Auth

- **vault** (HashiCorp) — Go — [github.com/hashicorp/vault](https://github.com/hashicorp/vault) 🏗️
- **keycloak** — Java — [github.com/keycloak/keycloak](https://github.com/keycloak/keycloak) 🏗️
- **ory/kratos** — Go identity — [github.com/ory/kratos](https://github.com/ory/kratos)
- **dex** — OIDC — [github.com/dexidp/dex](https://github.com/dexidp/dex)
- **infisical** — open-source secret mgmt — [github.com/Infisical/infisical](https://github.com/Infisical/infisical) 🆕

## 11. 큰 OSS Infra / Self-host

- ⭐ **gitea** — self-host Git (Go) — [github.com/go-gitea/gitea](https://github.com/go-gitea/gitea) 🏗️
- **forgejo** — Gitea fork — [codeberg.org/forgejo/forgejo](https://codeberg.org/forgejo/forgejo) 🆕
- **n8n** — workflow automation (TS) — [github.com/n8n-io/n8n](https://github.com/n8n-io/n8n) 🆕 🏗️
- **NocoDB** — Airtable alternative (TS) — [github.com/nocodb/nocodb](https://github.com/nocodb/nocodb)
- **Supabase** — Postgres backend — [github.com/supabase/supabase](https://github.com/supabase/supabase) 🆕 🏗️
- **PocketBase** — single-file BaaS (Go) — [github.com/pocketbase/pocketbase](https://github.com/pocketbase/pocketbase) 🆕

## 12. 관련

- [[examples]] — Examples hub
- [[backend-systems]] — Backend
- [[../books/devops/devops]] — DevOps 책
- [[../official-docs/cloud-infra]] — Cloud / Infra 공식
