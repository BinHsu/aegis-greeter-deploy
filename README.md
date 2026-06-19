# aegis-greeter-deploy

GitOps manifests for the **greeter** workload — the light example in the
aegis 4-repo model. The platform (aegis-platform-aws) renders an ArgoCD
Application that points at this repo's overlays; app developers never touch
platform infrastructure.

## Where this repo fits — the 4-repo model

```text
aegis-platform-aws          shared platform: Terraform, ArgoCD, observability
aegis-core                  heavy workload: AI inference engine + gateway (OIDC, IRSA, S3)
aegis-core-deploy           GitOps manifests for aegis-core   ← sibling of this repo
aegis-greeter               light workload: stateless HTTP greeter (app source)
aegis-greeter-deploy  ←     GitOps manifests for aegis-greeter (THIS repo)
```

**greeter is the minimal onboarding example.** It is a plain Deployment —
no GPU, no identity federation, no model storage. Its purpose is to show
that adding a workload to the platform costs one JSON entry in
`aegis-platform-aws` (`workloads.auto.tfvars.json`) plus a deploy repo like
this one. The platform renders the ArgoCD Application; the workload team owns
only `k8s/`.

The image is built and published in the sibling
[`aegis-greeter`](https://github.com/BinHsu/aegis-greeter) repo. This repo is
config only — no application code.

## Enrollment — how the platform picks this repo up

The platform's ApplicationSet discovers workload deploy repos by two signals:

1. The repo carries the `aegis-workload` GitHub topic.
2. `argocd/application.yaml` is present (the workload's declared intent).

The ApplicationSet renders the effective ArgoCD Application, injecting two
values this repo cannot know: the cluster region
(`aegis.binhsu.org/region`) and the ECR repository URL
(`aegis.binhsu.org/ecr-repository`). The kustomize `replacements` rules in
each overlay consume those annotations — the platform never learns
greeter-internal resource names.

`argocd/application.yaml` in this repo is a self-registration marker, not
something you `kubectl apply` directly.

## Release model — build once, promote by digest

The image is built **once** in the aegis-greeter build CI and promoted across
environments by its content digest (`sha256:…`). Environments differ only in
config (replicas, limits); the artifact is identical.

```text
aegis-greeter CI
  │  build + push image ONCE → capture sha256 digest
  ▼
overlays/staging   ← CI bumps the digest here
  │                  kustomize edit set image aegis-greeter@sha256:<digest>
  ▼  ArgoCD syncs → aegis-staging cluster
verify
  │
  ▼
promotion PR       ← copies the SAME digest into overlays/prod (no rebuild)
  ▼  ArgoCD syncs → aegis-prod cluster
```

**Why digest, not tag.** A git-sha tag is mutable by convention — a rebuild
can diverge between environments. The digest is the image's content hash, so
staging and prod provably run the identical artifact.

**Registry injected at sync, not committed.** The ECR repository URL embeds
an AWS account ID; committing it would violate the anonymization policy. The
platform ApplicationSet injects it via the `aegis.binhsu.org/ecr-repository`
annotation. A kustomize `replacements` rule in each overlay splices it into
the image's repo part while preserving the digest. This repo carries only the
image name and digest.

**Promotion is a digest copy.** A promotion PR copies the verified
`sha256:…` digest from `overlays/staging` into `overlays/prod`. ArgoCD
detects the git change and reconciles. No artifact is rebuilt.

> `kustomize edit set image` rewrites the kustomization into
> kustomize-canonical formatting (list indent, key order). The only semantic
> change is the digest — review the bump diff for the digest line, not the
> reflow.

For the full rationale see `aegis-platform-aws` ADR-10 ("Release model: build
once, promote by digest, single shared registry in a dedicated Deployment
account").

## Layout

```text
k8s/
  base/                 namespace, Deployment, Service, Ingress (ALB), HPA, NetworkPolicy
  overlays/
    staging/            digest pin + 1 replica — CI bumps here; ingress host: greeter.staging.aws.binhsu.org
    prod/               digest pin + 2 replicas, HPA 2→10; ingress host: greeter.prod.aws.binhsu.org
    talos/              on-prem overlay (WS0): drops ALB Ingress, adds Gateway API HTTPRoute
argocd/
  application.yaml      self-registration marker (platform renders the effective Application)
.github/workflows/
  validate.yml          renders both overlays, guards digest pin, simulates platform annotation injection
```

The ingress host follows the pattern `greeter.<env>.aws.binhsu.org`, injected
from the overlay patch. (PR #10 completes the base host migration from the
retired `.test` placeholder.)

## Render an overlay locally

```bash
kubectl kustomize k8s/overlays/staging
kubectl kustomize k8s/overlays/prod
kubectl kustomize k8s/overlays/talos
```

The staging and prod overlays use placeholder digests (`sha256:0000…`) until
the build CI or a promotion PR writes the real digest. `kustomize build`
renders without error; the image simply won't resolve until replaced.

## CI — validate.yml

Every pull request and push to `main` runs two checks:

1. **Digest-presence guard** — every rendered Deployment image must be
   `aegis-greeter@sha256:<64 hex>`. This catches the kustomize v5.8.1 wipe
   bug where a `spec.source.kustomize.images` newName-only injection silently
   deleted the overlay's `digest:` field and rendered `:latest`.

2. **Injection simulation** — replays the platform ApplicationSet's
   `kustomize edit add annotation` (without `--force`, exactly as ArgoCD runs
   it), then asserts the rendered image is `<ECR URL>@<same digest>` and the
   injected region reached `HELLO_TAG`. This catches the annotation-conflict
   class that broke ArgoCD on 2026-06-12.
