This example demonstrates Argo CD Image Updater support for applications using [Config Management Plugins (CMP)](https://argo-cd.readthedocs.io/en/stable/operator-manual/config-management-plugins/).

The plugin-type application support was introduced in Argo CD Image Updater v1.3.0 (commit cfe5fe0).

## Overview

This sample showcases **first-class plugin support** using `manifestTargets.plugin` to specify environment variables that the plugin reads from the Application's `spec.source.plugin.env` list.

The example demonstrates using **separate name/tag env vars** where the image repository and tag are stored in separate environment variables (`NGINX_IMAGE_NAME` and `NGINX_IMAGE_TAG`).

The sample includes a working echo-based plugin that:
- Uses busybox as the sidecar container
- Detects the plugin marker file (`echo-plugin.marker`)
- Generates deployment manifests using `echo` with environment variables from the Application spec

## ImageUpdater Custom Resource Configuration

The ImageUpdater CR is defined in `app/image-updater.yaml`:

```yaml
apiVersion: argocd-image-updater.argoproj.io/v1alpha1
kind: ImageUpdater
metadata:
  name: plugin-app
  namespace: argocd
spec:
  applicationRefs:
    - namePattern: "plugin-app"
      images:
        - alias: "nginx"
          imageName: "nginx:1.x"
          manifestTargets:
            plugin:
              name: "NGINX_IMAGE_NAME"
              tag: "NGINX_IMAGE_TAG"
```

### Key Fields

- `manifestTargets.plugin.name`: Environment variable for the image repository (e.g., `nginx`)
- `manifestTargets.plugin.tag`: Environment variable for the image tag (e.g., `1.27.0`)

**Alternative configuration:** You can also use `spec` instead of `name`/`tag` for a single env var containing the full image reference:
```yaml
manifestTargets:
  plugin:
    spec: "NGINX_IMAGE"  # Would be set to "nginx:1.27.0"
```

## Write-Back Methods

This example uses the **`git`** write-back method by default, which commits changes to the git repository.

Two write-back methods are supported for plugin applications:

### 1. Git Write-Back Method (Default)

Updates are written to the `.argocd-source-plugin-app.yaml` file in the git repository, which Argo CD merges into the Application source at sync time.

```yaml
spec:
  writeBackConfig:
    method: "git:secret:git-creds"
    gitConfig:
      repository: "https://github.com/chengfang/image-updater-examples.git"
      branch: "main"
```

When Image Updater detects a new version, it creates `.argocd-source-plugin-app.yaml` in the repository root:

```yaml
plugin:
  env:
  - name: NGINX_IMAGE_NAME
    value: nginx
  - name: NGINX_IMAGE_TAG
    value: 1.27.0
```

At sync time, Argo CD merges these values into the Application's `spec.source.plugin.env`.

### 2. ArgoCD Write-Back Method

Updates the Application's `spec.source.plugin.env` list directly via the ArgoCD API (no git commits).

To use this method, update the ImageUpdater CR:

```yaml
spec:
  # Remove or comment out writeBackConfig section to use argocd method
  # writeBackConfig:
  #   method: "git:secret:git-creds"
```

With the argocd method, Image Updater modifies the Application resource directly in the cluster.

## How It Works

After running image-updater with the **git** write-back method:

1. Image Updater detects the latest nginx version matching `1.x` constraint (e.g., `1.27.0`)
2. Image Updater commits `.argocd-source-plugin-app.yaml` to the git repository:

```yaml
plugin:
  env:
  - name: NGINX_IMAGE_NAME
    value: nginx
  - name: NGINX_IMAGE_TAG
    value: 1.27.0          # Updated from 1.20.0
```

3. Argo CD detects the git change and triggers a sync
4. Argo CD merges the values from `.argocd-source-plugin-app.yaml` into the Application's plugin env vars
5. The plugin reads these environment variables and generates the final Kubernetes manifests with the updated image

With the **argocd** write-back method, step 2 is replaced by directly updating the Application's `spec.source.plugin.env` via the ArgoCD API (no git commit).

## Plugin Installation

This example includes a complete working plugin configuration. Install the plugin as a sidecar to the Argo CD repo-server:

### 1. Install the plugin ConfigMap

```bash
kubectl apply -f plugin/echo-plugin-config.yaml
```

This creates a ConfigMap named `echo-plugin` that defines:
- **discover.find**: Looks for `echo-plugin.marker` file
- **init**: Logs the environment variables for debugging
- **generate**: Uses shell `echo` with heredoc to generate the deployment YAML, substituting `${NGINX_IMAGE_NAME}` and `${NGINX_IMAGE_TAG}` from environment variables

### 2. Patch the Argo CD repo-server to add the plugin sidecar

Add the following to your `argocd-repo-server` Deployment:

```yaml
spec:
  template:
    spec:
      containers:
      # Main repo-server container
      - name: argocd-repo-server
        # ... existing config ...
        volumeMounts:
        - name: plugin-tools
          mountPath: /home/argocd/cmp-server/plugins
      
      # Echo plugin sidecar
      - name: echo-plugin
        image: busybox:1.36
        command: [/var/run/argocd/argocd-cmp-server]
        securityContext:
          runAsNonRoot: true
          runAsUser: 999
        volumeMounts:
        - name: var-files
          mountPath: /var/run/argocd
        - name: plugins
          mountPath: /home/argocd/cmp-server/plugins
        - name: tmp
          mountPath: /tmp
        - name: echo-plugin-config
          mountPath: /home/argocd/cmp-server/config/plugin.yaml
          subPath: plugin.yaml
      
      volumes:
      - name: var-files
        emptyDir: {}
      - name: plugins
        emptyDir: {}
      - name: tmp
        emptyDir: {}
      - name: echo-plugin-config
        configMap:
          name: echo-plugin
```

Or use these patch commands:

```bash
# 1. Add the echo-plugin-config volume
kubectl patch deployment argocd-repo-server -n argocd --type=json -p='[
  {
    "op": "add",
    "path": "/spec/template/spec/volumes/-",
    "value": {"name": "echo-plugin-config", "configMap": {"name": "echo-plugin"}}
  }
]'

# 2. Add the echo-plugin sidecar container
kubectl patch deployment argocd-repo-server -n argocd --type=json -p='[
  {
    "op": "add",
    "path": "/spec/template/spec/containers/-",
    "value": {
      "name": "echo-plugin",
      "image": "busybox:1.36",
      "command": ["/var/run/argocd/argocd-cmp-server"],
      "securityContext": {
        "runAsNonRoot": true,
        "runAsUser": 999
      },
      "volumeMounts": [
        {"name": "var-files", "mountPath": "/var/run/argocd"},
        {"name": "plugins", "mountPath": "/home/argocd/cmp-server/plugins"},
        {"name": "tmp", "mountPath": "/tmp"},
        {"name": "echo-plugin-config", "mountPath": "/home/argocd/cmp-server/config/plugin.yaml", "subPath": "plugin.yaml"}
      ]
    }
  }
]'

# 3. Wait for rollout
kubectl rollout status deployment argocd-repo-server -n argocd
```

**Note:** The common volumes (`var-files`, `plugins`, `tmp`) typically already exist in Argo CD installations. If you get duplicate volume errors, those volumes are already present and you can proceed with just steps 1 and 2 above.

### 3. Wait for repo-server to restart

```bash
kubectl rollout status deployment argocd-repo-server -n argocd
```

## Testing

### Prerequisites

- Argo CD installation with the echo-plugin sidecar configured (see above)
- Argo CD Image Updater v1.3.0+

### Test with git write-back method (default)

```bash
# 1. Create the git credentials secret
kubectl -n argocd create secret generic git-creds --from-literal=username=xxx --from-literal=password=xxx

# 2. Install the plugin-app Application and ImageUpdater CR
kubectl apply -f plugin/app/plugin-app.yaml
kubectl apply -f plugin/app/image-updater.yaml

# 3. Verify initial application state
kubectl describe -n argocd apps/plugin-app

# 4. Check the initial plugin env vars in the Application
kubectl get -n argocd apps/plugin-app -o jsonpath='{.spec.source.plugin.env}'
[
  {"name":"NGINX_IMAGE_NAME","value":"nginx"},
  {"name":"NGINX_IMAGE_TAG","value":"1.20.0"}
]

# 5. Run image-updater from command line
../image-updater/dist/argocd-image-updater run --once --registries-conf-path=""

# 6. Check the git repository for .argocd-source-plugin-app.yaml
# This file contains the plugin env overrides that Argo CD merges at sync time
cat .argocd-source-plugin-app.yaml

# 7. After Argo CD syncs, verify the updated image in the deployed pod
kubectl get pod -n argocd -l app=plugin-app -o jsonpath='{.items[0].spec.containers[0].image}'
nginx:1.27.0

# 8. Cleanup
kubectl delete -f plugin/app/image-updater.yaml
kubectl delete -f plugin/app/plugin-app.yaml
kubectl -n argocd delete secret git-creds
```

### Test with argocd write-back method

To test with the argocd write-back method instead:

```bash
# 1. Modify app/image-updater.yaml to remove the writeBackConfig section
# OR use this command to create a version without git write-back:
cat > /tmp/image-updater-argocd.yaml <<EOF
apiVersion: argocd-image-updater.argoproj.io/v1alpha1
kind: ImageUpdater
metadata:
  name: plugin-app
  namespace: argocd
spec:
  applicationRefs:
    - namePattern: "plugin-app"
      images:
        - alias: "nginx"
          imageName: "nginx:1.x"
          commonUpdateSettings:
            updateStrategy: "semver"
            forceUpdate: false
          manifestTargets:
            plugin:
              name: "NGINX_IMAGE_NAME"
              tag: "NGINX_IMAGE_TAG"
EOF

# 2. Install the plugin-app Application and ImageUpdater CR
kubectl apply -f plugin/app/plugin-app.yaml
kubectl apply -f /tmp/image-updater-argocd.yaml

# 3. Verify initial application state
kubectl describe -n argocd apps/plugin-app

# 4. Check the initial plugin env vars
kubectl get -n argocd apps/plugin-app -o jsonpath='{.spec.source.plugin.env}'
[
  {"name":"NGINX_IMAGE_NAME","value":"nginx"},
  {"name":"NGINX_IMAGE_TAG","value":"1.20.0"}
]

# 5. Run image-updater from command line
../image-updater/dist/argocd-image-updater run --once --registries-conf-path=""

# 6. Check the updated plugin env vars (modified directly in the Application spec)
kubectl get -n argocd apps/plugin-app -o jsonpath='{.spec.source.plugin.env}'
[
  {"name":"NGINX_IMAGE_NAME","value":"nginx"},
  {"name":"NGINX_IMAGE_TAG","value":"1.27.0"}
]

# 7. After Argo CD syncs, verify the updated image
kubectl get pod -n argocd -l app=plugin-app -o jsonpath='{.items[0].spec.containers[0].image}'
nginx:1.27.0

# 8. Cleanup
kubectl delete -f /tmp/image-updater-argocd.yaml
kubectl delete -f plugin/app/plugin-app.yaml
```

## How the Echo Plugin Works

The echo-plugin is a minimal Config Management Plugin that demonstrates plugin support:

1. **Discovery**: Checks for the `echo-plugin.marker` file in the application source
2. **Initialization**: Logs the environment variables for debugging:
   ```bash
   echo "NGINX_IMAGE_NAME=${NGINX_IMAGE_NAME}"
   echo "NGINX_IMAGE_TAG=${NGINX_IMAGE_TAG}"
   ```
3. **Generation**: Uses shell heredoc to generate the deployment YAML:
   ```bash
   cat <<EOF
   apiVersion: apps/v1
   kind: Deployment
   ...
     containers:
     - name: nginx
       image: ${NGINX_IMAGE_NAME}:${NGINX_IMAGE_TAG}
   EOF
   ```
4. **Environment Variables**: The plugin receives `NGINX_IMAGE_NAME` and `NGINX_IMAGE_TAG` from the Application's `spec.source.plugin.env` list
5. **Image Updater Integration**: When Image Updater detects a new nginx version, it updates these env vars in the Application spec, triggering a new sync with the updated image

## Notes

- `manifestTargets.plugin` is mutually exclusive with `manifestTargets.helm` and `manifestTargets.kustomize`
- When using `manifestTargets.plugin`, do not set `writeBackConfig.gitConfig.writeBackTarget` to `helmvalues` or `kustomization` - use the default write-back target instead
- **Plugin discovery**: This example uses auto-discovery - the Application spec omits the `plugin.name` field, so Argo CD runs each registered plugin's discover command until one succeeds. Alternatively, you can explicitly specify `name: my-custom-plugin` in `spec.source.plugin` to directly use that plugin
- The `source/` directory only contains the `echo-plugin.marker` file for plugin discovery - all manifests are generated dynamically by the plugin

## Alternative Approach: Plugin Apps Consuming Helm/Kustomize Files

If your plugin consumes Helm-style or Kustomize-style values files from git (e.g., helmfile), you can use `manifestTargets.helm` or `manifestTargets.kustomize` with the git write-back method instead. See the `kustomize-helmvalues` example for a similar pattern.
