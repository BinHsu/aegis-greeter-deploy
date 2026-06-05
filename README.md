# aegis-greeter-deploy

Kubernetes manifests for the `aegis-greeter` workload — a kustomize base plus
per-environment overlays. ArgoCD (run by the platform) reconciles the overlays
into the matching clusters. This repo is config only; the application image is
built in the sibling [`aegis-greeter`](https://github.com/BinHsu/aegis-greeter)
repo.

## Release model — build once, promote by digest

The image is built **once** and the same immutable artifact is promoted across
environments by its content digest (`sha256:...`), never rebuilt per env. Env
differences (replicas, limits, host, role-trust) live in the overlays, not in
the artifact.

```
aegis-greeter build CI
  │  build + push image ONCE, capture sha256 digest
  ▼
overlays/staging   ← CI bumps the digest here
  │                  (kustomize edit set image aegis-greeter@sha256:<digest>)
  ▼  ArgoCD syncs → aegis-staging cluster
verify
  │
  ▼
promotion PR       ← copies the SAME digest into overlays/prod (no rebuild)
  ▼  ArgoCD auto-syncs → aegis-prod cluster
```

- **Pinned by digest, not tag.** Each overlay's `images[]` entry carries the
  bare image name and a `digest:` field. A git-sha tag is mutable by
  convention and lets a rebuild diverge between envs; the digest is the image's
  content hash, so staging and prod provably run the identical artifact.
- **Registry injected at sync, not committed.** The `newName` (ECR registry
  URL) is absent on purpose — it embeds an AWS account ID. The platform
  ApplicationSet injects it via `kustomize.images` from its gitignored
  `registries.auto.tfvars.json`. One shared registry lives in a dedicated
  `aegis-deployment` account; this repo carries only name + digest.
- **Promotion is a digest copy.** Promoting staging → prod copies the verified
  `sha256:` digest into `overlays/prod`; ArgoCD reconciles the git change. No
  artifact is rebuilt at promotion time.

See the platform repo for the full rationale: `aegis-platform-aws` README
"Release model" section and ADR-10 ("Release model: build once, promote by
digest, single shared registry in a dedicated Deployment account").

## Layout

```
k8s/
  base/                 namespace, deployment, service, ingress, hpa, networkpolicy
  overlays/
    staging/            digest pin + leaner footprint (1 replica) — CI bumps here
    prod/               digest pin + base footprint (2 replicas, HPA 2→10)
argocd/
  application.yaml      self-registration marker (the platform renders the effective Application)
```

## Render an overlay

```bash
kubectl kustomize k8s/overlays/staging
kubectl kustomize k8s/overlays/prod
```

> The placeholder digests (`sha256:0000…`) render but do not resolve a real
> image. The build CI and the promotion PR replace them with the published
> digest.
