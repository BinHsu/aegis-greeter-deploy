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

```text
aegis-greeter build CI
  │  build + push image ONCE to GHCR, capture sha256 digest
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
  bare image name, a `newName:`, and a `digest:` field. A git-sha tag is
  mutable by convention and lets a rebuild diverge between envs; the digest is
  the image's content hash, so staging and prod provably run the identical
  artifact.
- **Registry — public GHCR, committed directly (2026-07-21).** Each overlay's
  `images.newName` is the static ref `ghcr.io/binhsu/aegis-greeter`. Unlike the
  old AWS ECR path, GHCR has no account id and no region to hide, so there is
  nothing for the platform to inject — no `aegis.binhsu.org/ecr-repository`
  annotation, no `replacements` splice. This mirrors aegis-core's earlier move
  to GHCR (`aegis-platform-aws` ADR-23); see ADR-24 for greeter's version of
  that decision. The earlier `kustomize.images` newName-only injection is a
  separate, older, already-fixed bug (it provably wiped the overlay's digest
  field and rendered `:latest`) — unrelated to this registry change.
- **Promotion is a digest copy.** Promoting staging → prod copies the verified
  `sha256:` digest into `overlays/prod`; ArgoCD reconciles the git change. No
  artifact is rebuilt at promotion time.
  > `kustomize edit set image` rewrites the whole kustomization into
  > kustomize-canonical formatting (list indent, key order). The only
  > *semantic* change is the digest — the registry/region replacements still
  > render identically — so review the bump diff for the digest line, not the
  > reflow.

See the platform repo for the full rationale: `aegis-platform-aws` README
"Release model" section, ADR-10 ("build once, promote by digest"), and ADR-24
("greeter image distribution: public GHCR").

> ⚠️ **Post-migration follow-up.** The digests currently committed in both
> overlays were promoted from the OLD ECR-hosted image and do not exist in
> GHCR. Staging self-heals on the next push to aegis-greeter's `main` (its
> publish workflow now targets GHCR and will overwrite the digest here as
> always). Prod requires a fresh, manual promotion PR copying that new GHCR
> digest in — do not treat prod as pullable until that promotion lands. Also
> confirm the `ghcr.io/binhsu/aegis-greeter` package visibility: if it is
> private (GHCR's default), pulling nodes need a GHCR `imagePullSecret`,
> which nothing in this repo currently wires up.

## Layout

```text
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
