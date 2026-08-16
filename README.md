# Secure Kubernetes Deployment — Payments Microservice

Hardening a Flask-based payments microservice (tokenization + transaction
metadata) from an insecure baseline to a defensible, production-oriented
Kubernetes deployment — as part of learning zero-trust and DevSecOps practices.

**App source**: adapted from a technical assessment exercise provided by
Dodo Payments; the Kubernetes hardening, secrets management, and RBAC design
in this repo are my own work.

## What was wrong with the original deployment

- Plaintext API keys and database passwords committed directly in `deployment.yaml`
- Containers running as root with no `securityContext` restrictions
- No resource limits or health probes
- Using the default Kubernetes ServiceAccount instead of a scoped one
- Mutable/ambiguous image tags

## What this repo demonstrates

- **Least-privilege RBAC**: a dedicated ServiceAccount (`ledger-api-sa`) with a
  narrowly-scoped Role, `automountServiceAccountToken: false`
- **Secrets out of git**: Bitnami Sealed Secrets — secrets are encrypted with
  the cluster's public key before being committed, decryptable only by the
  in-cluster controller (`deploy/sealed-secret.yaml`)
- **Hardened `securityContext`**: non-root user, read-only root filesystem,
  all Linux capabilities dropped, `seccompProfile: RuntimeDefault`
- **Resource limits and health probes** on every container
- **A real, pushed container image** (GHCR) rather than a local-only build

## Architecture

`ledger-api` (Flask, tokenization + transaction endpoints) and a `reporting`
neighbour service run in the `payments` namespace on a local `kind` cluster.

## Running it locally

\`\`\`bash
kind create cluster --name ledger-cluster
kubectl apply -f deploy/namespace.yaml
kubectl apply -f deploy/serviceaccount.yaml
kubectl apply -f deploy/sealed-secret.yaml
kubectl apply -f deploy/deployment.yaml
kubectl apply -f deploy/service.yaml
kubectl apply -f deploy/neighbour.yaml
\`\`\`

Note: `deploy/sealed-secret.yaml` can only be decrypted by a Sealed Secrets
controller holding the matching private key — you'll need your own controller
installed (`helm install sealed-secrets sealed-secrets/sealed-secrets -n kube-system`)
and your own sealed secret if reproducing this from scratch.

## Remaining Improvements

- Signed and digest-pinned images using Cosign
- TLS termination at the Ingress
- Further workload identity/RBAC personas if required

## Admission control & runtime enforcement (Task 1, continued)

- **Ingress**: ingress-nginx, verified end-to-end via port-forward and Host header routing to the hardened pods.
- **Kyverno**: three ClusterPolicy resources enforce no root containers, no latest tags, and images restricted to an approved registry. Verified live by attempting to apply the original insecure manifest with plaintext secrets and a root container. Kyverno's admission webhook rejected it outright, citing both violated policies. See deploy/kyverno-rejection-proof.txt.
- **Pod Security Standards**: the payments namespace enforces the restricted profile at the Kubernetes API level. Proven by deleting the neighbour reporting pod, deliberately left unhardened to match the original assignment. Its ReplicaSet could not recreate it, failing PSS validation on every attempt. See deploy/pss-enforcement-proof.txt. ledger-api, already built to restricted-level security, was unaffected.

This two-layer approach of Kyverno plus native PSS demonstrates defense-in-depth: even if one admission control were misconfigured or bypassed, the other still blocks non-compliant workloads.
