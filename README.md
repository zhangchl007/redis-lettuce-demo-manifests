# GitOps Demo
redis-lettuce-demo-manifests

Key files:
- Root Kustomize entry: [gitops-demo/kustomization.yaml](redis-lettuce-demo-manifests/gitops-demo/kustomization.yaml)
- Example policy‑scanned Argo CD Application & hook: [gitops-demo/kustomize-example/test-application.yaml](redis-lettuce-demo-manifests/gitops-demo/kustomize-example/test-application.yaml)
- Example privileged test pod (fails policy if disallowed): [gitops-demo/kustomize-example/app/test-pod.yaml](redis-lettuce-demo-manifests/gitops-demo/kustomize-example/app/test-pod.yaml)
- App Helm source override (image tag): [gitops-demo/helm-app/.argocd-source-redisdemo-prod1.yaml](redis-lettuce-demo-manifests/gitops-demo/helm-app/.argocd-source-redisdemo-prod1.yaml)
- Bitnami Redis chart documentation snapshot: [gitops-demo/helm-infra/README.md](redis-lettuce-demo-manifests/gitops-demo/helm-infra/README.md)

## GitOps Model

1. Argo CD points at `gitops-demo/` (Kustomize).  
2. The root Kustomize aggregates:
   - ApplicationSet(s) (workload & infra Helm releases)
   - Example standalone Application (Kustomize) with a PreSync policy scan hook Job.
3. Image & deployment changes are performed by editing small YAML parameter files (e.g. Helm parameter sources) and committing.

## Application Delivery (Helm)

The application (Spring Boot OTP service) is delivered via an Argo CD ApplicationSet (under `helm-app/`). Image/tag overrides are passed using Argo CD source parameter file [.argocd-source-redisdemo-prod1.yaml](redis-lettuce-demo-manifests/gitops-demo/helm-app/.argocd-source-redisdemo-prod1.yaml).

Update image (example):
```sh
# Edit [.argocd-source-redisdemo-prod1.yaml](http://_vscodecontentref_/5) and bump tag
git add [.argocd-source-redisdemo-prod1.yaml](http://_vscodecontentref_/6)
git commit -m "chore: bump redisdemo image to v54"
git push

# validate application deployment
```bash
curl -X POST -d @- -H 'Content-Type: application/json' \
http://localhost:8080/api/v1/otp/generate <<'EOF'
{
	"email": "user2@gmail.com"
}
EOF
```

## Policy / Security Scan (PreSync Hook)

```bash
The example Kustomize application (kustomize-example) demonstrates enforcing policy before sync:

Definition: test-application.yaml
The Job annotated with argocd.argoproj.io/hook: PreSync clones the repo, builds Kustomize manifests, and runs Conftest against them (expecting optional policies under policy/).
Hook behavior:

Fails fast if a manifest (e.g. test-pod.yaml) violates Rego rules (e.g. disallowing privileged: true).
Uses HookSucceeded delete policy so successful scan Jobs are cleaned up.
Adding Policies
Place Rego files under policy/ops/ in the same repository (or adjust POLICY_DIR env var in the hook Job). Example (deny privileged):


package kubernetes.security.privilegeddeny[msg] {  input.kind == "Pod"  some c  container := input.spec.containers[c]  container.securityContext.privileged == true  msg = sprintf("Privileged container not allowed: %s", [container.name])}
Commit and push; subsequent syncs will block if a privileged pod appears.

Typical Workflow
Action	Change	Result
Bump application version	Edit Helm source param file	Argo CD deploys new image
Adjust Redis topology	Update infra values / chart params	StatefulSet reconciliation
Strengthen policy	Add/update Rego under policy/	PreSync hook enforces before apply
Add new Kustomize app	Create folder + kustomization + Application manifest	Argo CD syncs new app-of-apps entry
Local Validation
Render root Kustomize:


kustomize build gitops-demo/
Render example app only:


kustomize build gitops-demo/kustomize-example/app
(Install Kustomize if needed.)

## Troubleshooting

```bash
Symptom	Check
Sync blocked at PreSync	Inspect Job logs from opa-scan-kustomize-example
Redis auth failures	Ensure auth.enabled & secret values in Helm infra parameters
No rollout of new image	Confirm commit updated tracked parameter file & Argo CD repo refresh
Policy not applied	Verify hook annotation & that Job name not colliding (Replace / delete policy)
Extending
Add Argo Rollouts CRDs + Rollout manifests in a new Helm or Kustomize component for progressive delivery.
Introduce analysis / metric gates (e.g. Prometheus) before full traffic shift.
Externalize policy bundle via OCI & pull in PreSync Job for supply chain hardening.
Reference
Application Kustomize root: gitops-demo/kustomization.yaml
Policy scan example: gitops-demo/kustomize-example/test-application.yaml
Helm parameter override: gitops-demo/helm-app/.argocd-source-redisdemo-prod1.yaml
Bitnami Redis docs snapshot: gitops-demo/helm-infra/README.md
Minimal API test (after app exposure):


curl -X POST -H 'Content-Type: application/json' \  http://<app-host>/api/v1/otp/generate -d '{"email":"user@example.com"}'
(Host/Ingress depends on platform configuration.)