This example contains a single-source app and a multi-source app, both using the same
`deployment.yaml`. You can compare the observe how image-updater works for these 2 types
of applications. `deployment.yaml` uses an old version of nginx: `1.16.0`, and upon running
image-updater, the container image tag should be updated to `1.16.1`

## ImageUpdater Custom Resource Configuration

This sample uses the ImageUpdater Custom Resource (CR) introduced in v1.0.0, instead of the legacy
annotation-based configuration. A single ImageUpdater CR (`app/image-updater.yaml`) manages both
applications using a glob pattern in `namePattern`:

```yaml
apiVersion: argocd-image-updater.argoproj.io/v1alpha1
kind: ImageUpdater
metadata:
  name: multi-source-kustomize
  namespace: argocd
spec:
  namespace: argocd
  commonUpdateSettings:
    updateStrategy: "semver"
    forceUpdate: true
  applicationRefs:
    - namePattern: "*-source-kustomize"
      images:
        - alias: "nginx"
          imageName: "nginx:1.16.x"
```

Key features:
- `commonUpdateSettings` is defined at the spec level as global defaults for all matched applications
- `namePattern: "*-source-kustomize"` uses a glob pattern to match both `multi-source-kustomize` and `single-source-kustomize` apps

Both samples use the default `argocd` write-back method, which updates the Application spec
directly via the ArgoCD API without committing changes to git.

## test with multi-source-app
```bash
# to install the multi-source-app Argo CD application and ImageUpdater CR
kubectl apply -f multi-source-kustomize/app/multi-source-app.yaml
kubectl apply -f multi-source-kustomize/app/image-updater.yaml

# to view the application pod
kubectl get pod -l app=multi-source-kustomize -n argocd
NAME                                     READY   STATUS    RESTARTS   AGE
multi-source-kustomize-6965f6974-vn7mw   1/1     Running   0          9m22s

# to view the current container image name and image tag
kubectl get pod -l app=multi-source-kustomize -n argocd -o jsonpath='{.items[0].spec.containers[0].image}'
nginx:1.16.0

# to run image-updater from command line
../image-updater/dist/argocd-image-updater run --once --registries-conf-path=""

# to check the updated image tag
kubectl get pod -l app=multi-source-kustomize -n argocd -o jsonpath='{.items[0].spec.containers[0].image}'
nginx:1.16.1

# to delete the multi-source-app and ImageUpdater CR
kubectl delete -f multi-source-kustomize/app/image-updater.yaml
kubectl delete -f multi-source-kustomize/app/multi-source-app.yaml
```

## test with single-source-app
```bash
# to install single-source-app Argo CD application and ImageUpdater CR
kubectl apply -f multi-source-kustomize/app/single-source-app.yaml
kubectl apply -f multi-source-kustomize/app/image-updater.yaml

# to view the current container image name and image tag
kubectl get pod -l app=multi-source-kustomize -n argocd -o jsonpath='{.items[0].spec.containers[0].image}'
nginx:1.16.0

# to run image-updater from command line
../image-updater/dist/argocd-image-updater run --once --registries-conf-path=""

# to delete the app and ImageUpdater CR
kubectl delete -f multi-source-kustomize/app/image-updater.yaml
kubectl delete -f multi-source-kustomize/app/single-source-app.yaml
```
