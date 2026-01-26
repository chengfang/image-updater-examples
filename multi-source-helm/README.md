This example contains a helm-based multi-source app:
* source 1: nginx chart from helm repository
* source 2: a custom values file in this repo to override the nginx image version to older version

After running image-updater, the image version in the live cluster should be updated to the new version.

## ImageUpdater Custom Resource Configuration

This sample uses the ImageUpdater Custom Resource (CR) introduced in v1.0.0, instead of the legacy
annotation-based configuration. The ImageUpdater CR is defined in `app/image-updater.yaml`:

```yaml
apiVersion: argocd-image-updater.argoproj.io/v1alpha1
kind: ImageUpdater
metadata:
  name: multi-source-helm
  namespace: argocd
spec:
  namespace: argocd
  applicationRefs:
    - namePattern: "multi-source-helm"
      images:
        - alias: "nginx"
          imageName: "docker.io/bitnami/nginx:1.27.x"
          commonUpdateSettings:
            updateStrategy: "semver"
            forceUpdate: true
```

Key fields:
- `applicationRefs.namePattern`: selects the ArgoCD Application by name
- `images.alias`: unique identifier for the image within this CR
- `images.imageName`: the image to track with version constraint
- `commonUpdateSettings.updateStrategy`: "semver" for semantic versioning
- `commonUpdateSettings.forceUpdate`: force update even if image appears unchanged

This sample uses the default `argocd` write-back method, which updates the Application spec
directly via the ArgoCD API without committing changes to git.

## test with multi-source-app
```bash
# to install the multi-source-helm Argo CD application and ImageUpdater CR
kubectl apply -f multi-source-helm/app/multi-source-app.yaml
kubectl apply -f multi-source-helm/app/image-updater.yaml

# to view the application pod
kubectl get pod/multi-source-helm-nginx-75b44767bb-gtnnd -n argocd -o yaml

# to view the current container image name and image tag
kubectl get pod/multi-source-helm-nginx-75b44767bb-gtnnd -n argocd -o jsonpath='{.spec.containers[0].image}'
docker.io/bitnami/nginx:1.27.1

# to run image-updater from command line
../image-updater/dist/argocd-image-updater run --once --registries-conf-path=""

# to check the updated image tag
kubectl get pod multi-source-helm-nginx-598b65b7bd-s854v -n argocd -o jsonpath='{.spec.containers[0].image}'
docker.io/bitnami/nginx:1.27.2

# to delete the multi-source-app and ImageUpdater CR
kubectl delete -f multi-source-helm/app/image-updater.yaml
kubectl delete -f multi-source-helm/app/multi-source-app.yaml
```
