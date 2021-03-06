---
---

## kubectl rollout resume

Resume a paused resource

### 摘要


Resume a paused resource

Paused resources will not be reconciled by a controller. By resuming a
resource, we allow it to be reconciled again.
Currently only deployments support being resumed.

```
kubectl rollout resume RESOURCE
```

### 示例

```
# Resume an already paused deployment
kubectl rollout resume deployment/nginx
```

### 选项

```
  -f, --filename=[]: Filename, directory, or URL to a file identifying the resource to get from a server.
  -R, --recursive[=false]: Process the directory used in -f, --filename recursively. Useful when you want to manage related manifests organized within the same directory.
```

{% include_relative parent_commands.md %}

### 参见

* [kubectl rollout](kubectl_rollout.md)	 - rollout manages a deployment
