# Using capabilities in AKS

This is an exploratory repo with information on how to use Linux capabilities(7) in AKS.

## What we know

1. After Kubernetes 1.21 it looks like [Capabilities(7)](https://man7.org/linux/man-pages/man7/capabilities.7.html) will only work when `runAsUser` is set to `0` - meaning, `root`. 

this is the [code](https://github.com/containerd/containerd/blob/main/pkg/cri/server/container_create_linux.go#L260-L267) that prevents any user other than `root` to have capabilities. This was added by the commit referenced [here](https://github.com/containerd/containerd/commit/50c73e6dc550c2cdb579e303ac26394497f9f331):

```golang
	// Clear all ambient capabilities. The implication of non-root + caps
	// is not clearly defined in Kubernetes.
	// See https://github.com/kubernetes/kubernetes/issues/56374
	// Keep docker's behavior for now.
	specOpts = append(specOpts,
		customopts.WithoutAmbientCaps,
		customopts.WithSelinuxLabels(processLabel, mountLabel),
	)
```

1. On the previous note, we can add/remove capabilities to `root` - which essentially removes a lot of the superpowers that `root` have on by default (e.g.: cap_net_admin).

## Approaches

### 1. Using `RunAsUser: 0` but restricting the capabilities in the account

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: gbbapp
  name: gbbapp
  namespace: ns-gbb
spec:
  replicas: 1
  selector:
    matchLabels:
      app: gbbapp
  template:
    metadata:
      labels:
        app: gbbapp
    spec:
      containers:
      - name: gbbapp
        image: gbbapp/k8s:cfa
        command: ["/bin/bash"]
        args: ["-c", "sleep 3600"]
        securityContext:
           runAsUser: 0
           capabilities:
             drop: ["ALL"]
             add: ["IPC_LOCK"]
```

In the example above we will be granting `cap_ipc_lock` to the running user (root) and nothing else.

### 2. Adding capabilities to binaries during the build process

It is possible to add capabilities during the build process with docker/podman. With this approach you can remove the `RunAsUser` parameter altogether. The capabilities added there will persist when the image runs as a container. The following example adds `cap_ipc_lock` to python3.8

```Dockerfile
FROM ubuntu

RUN apt-get update && apt-get install -y libcap2-bin && \
    setcap cap_ipc_lock+eip /usr/bin/python3.8

CMD ["/bin/bash"]
```

We can verify that the capability was added by running the `getcap` command against a binary, which in this case is `python3.8`:

```bash
$ getcap /usr/bin/python3.8
/usr/bin/python3.8 = cap_ipc_lock+eip
```

## References:

1. https://github.com/kubernetes/kubernetes/issues/56374
