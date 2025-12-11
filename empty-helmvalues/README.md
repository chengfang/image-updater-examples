This example contains a helm-based multi-source app:
* source 1: nginx chart from helm repository
* source 2: an empty custom values file to serve as the write-back-target for image updater

After running image-updater, the image version in the live cluster should be updated to the new version.
The changes are written to `empty-helmvalues/source2/values.yaml` file. See commit history of source2 repo for diffs
```shell
  -  repository: bitnami/nginx
  -  tag: 1.27.1
  
  +  repository: docker.io/bitnami/nginx
  +  tag: 1.27.2
```
To configure the app to write updates to the correct repo, branch, values file path and values elements,
the following application annotations are required:
```yaml
argocd-image-updater.argoproj.io/nginx.helm.image-name: image.repository
argocd-image-updater.argoproj.io/nginx.helm.image-tag: image.tag
argocd-image-updater.argoproj.io/git-repository: https://github.com/chengfang/image-updater-examples.git
argocd-image-updater.argoproj.io/git-branch: main
argocd-image-updater.argoproj.io/write-back-method: git:secret:argocd/git-creds
argocd-image-updater.argoproj.io/write-back-target: helmvalues:/empty-helmvalues/source2/values.yaml
```

To configure git write-back to use the default target file, `.argocd-source-<appName>.yaml`,
remove the following line from application annotations:
```yaml
argocd-image-updater.argoproj.io/write-back-target: helmvalues
```

To configure the application to use the default write-back-method, `argocd`, remove these lines
from application annotations. With `argocd` write-back-method, no updates will be committed to
the git repository.
```yaml
argocd-image-updater.argoproj.io/git-repository: https://github.com/chengfang/image-updater-examples.git
argocd-image-updater.argoproj.io/git-branch: main
argocd-image-updater.argoproj.io/write-back-method: git:secret:argocd/git-creds
argocd-image-updater.argoproj.io/write-back-target: helmvalues:/empty-helmvalues/source2/values.yaml
```

## test with write-helmvalues
```bash
# to create the secret to access the git repo
kubectl -n argocd create secret generic git-creds --from-literal=username=xxx --from-literal=password=xxx

# to install the empty-helmvalues Argo CD application as a plain manifest
kubectl apply -f empty-helmvalues/app/empty-helmvalues.yaml

# to view the application pod
kubectl get pod empty-helmvalues-nginx-xxx -n argocd -o yaml

# to view the current container image name and image tag
kubectl get pod empty-helmvalues-nginx-75b44767bb-gtnnd -n argocd -o jsonpath='{.spec.containers[0].image}'
docker.io/bitnami/nginx:1.27.1

# to run image-updater from command line
../image-updater/dist/argocd-image-updater run --once --registries-conf-path=""


# to check the updated image tag
kubectl get pod empty-helmvalues-nginx-xxx -n argocd -o jsonpath='{.spec.containers[0].image}'
docker.io/bitnami/nginx:1.27.2

# to delete the write-helmvalues app
kubectl delete -f empty-helmvalues/app/write-helmvalues.yaml 
application.argoproj.io "empty-helmvalues" deleted
```