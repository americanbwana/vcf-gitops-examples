# VCF GitOps examples

A guestbook sample rebuilt to run cleanly on **VCF 9 / vSphere Supervisor** with the
**Argo CD Supervisor Service**. The upstream
[`argocd-example-apps/guestbook`](https://github.com/argoproj/argocd-example-apps/tree/master/guestbook)
runs a root-based image with no security context, so VKS clusters refuse its pod under the
`restricted` Pod Security level. Everything here is built for that level from the start.

## Layout

| Path | What it is |
| --- | --- |
| `app/` | Dockerfile, nginx config and page. Non-root (uid 101), port 8080. |
| `apps/guestbook/base/` | Deployment and Service, `restricted`-compliant. |
| `apps/guestbook/overlays/lab/` | Namespace, image tag, replicas, LoadBalancer Service. |
| `argocd/guestbook-app.yaml` | The Argo CD Application, applied with `kubectl`. |
| `argocd/examples/` | Untested ApplicationSet for several clusters. |
| `supervisor/` | Supervisor-side pieces: Argo CD instance, RoleBinding, EntityManagementPolicy. |
| `.github/workflows/` | Builds and pushes the image to GHCR on a version tag. |

## What "restricted" needs, and where it is set

| Requirement | Where |
| --- | --- |
| `runAsNonRoot: true`, numeric `runAsUser` | Pod `securityContext` in `deployment.yaml` |
| `seccompProfile: RuntimeDefault` | Pod `securityContext` |
| `allowPrivilegeEscalation: false` | Container `securityContext` |
| `capabilities.drop: ["ALL"]` | Container `securityContext` |
| Non-root image on an unprivileged port | `app/Dockerfile` (nginx-unprivileged, 8080) |
| Writable paths with a read-only root filesystem | `emptyDir` on `/tmp` |

## Setup order

1. **Supervisor side** (once per Supervisor and namespace). Sign in with a context that has
   Supervisor admin rights. With the `vcf` CLI, force basic auth if OIDC auto-detection gives
   you a user with no rights:

   ```
   vcf context create sup --endpoint https://<supervisor-ip> --type k8s \
     --auth-type basic --username administrator@vsphere.local --insecure-skip-tls-verify
   ```

   Install the Argo CD service on the Supervisor (vCenter: Supervisor Management > Services >
   Actions > Manage Service), then edit the namespaces in `supervisor/` and apply:

   ```
   kubectl apply -f supervisor/argocd-instance.yaml
   kubectl apply -f supervisor/rolebinding.yaml
   kubectl apply -f supervisor/entitymanagementpolicy.yaml
   ```

   The policy registers VKS clusters in the namespace as Argo CD destinations automatically.

2. **Image.** Push a tag (`git tag v0.1.0 && git push --tags`). The workflow builds a
   `linux/amd64` image and pushes `ghcr.io/<owner>/vcf-gitops-guestbook:0.1.0`. Make the package
   public, or add an image pull secret to the namespace. Set the same version as `newTag` in
   `apps/guestbook/overlays/lab/kustomization.yaml`.

3. **Application.** Edit `repoURL`, the cluster `server` and the Argo CD namespace in
   `argocd/guestbook-app.yaml`, then:

   ```
   kubectl apply -f argocd/guestbook-app.yaml
   ```

   Sync it in the Argo CD UI.

## Notes from real use

- The in-cluster destination (`https://kubernetes.default.svc`) is disabled in this service.
  Applications must target a registered VKS cluster.
- Because of that, an app-of-apps does not work: child Application resources would have to be
  created in the Supervisor namespace where Argo CD runs. Apply Applications with `kubectl`.
- A RoleBinding for `argocd-k8s-sa` alone did not register a cluster. The
  `EntityManagementPolicy` is what makes the controller register it.
- Kustomize patches must target the base resource name, not the name after a `namePrefix`.
- Build images for `linux/amd64`. An image built on Apple Silicon without `--platform` fails on
  VKS nodes with an exec format error.
- The Supervisor blocks user RoleBindings (service accounts only). Grant people access in
  vCenter (Namespaces Owner, Edit or View) or VCFA.
- Not covered: deploying to Supervisor namespaces (for example VM Service) through Argo CD.
