# Enabling a Pod in OpenShift to run rootless container builds

### Create the following SCC as a cluster-admin

__Note:__ This SCC is a copy of the default `container-build` SCC which is bundled with OpenShift Dev Spaces to allow container builds within user workspaces.

```bash
cat << EOF | oc apply -f -
apiVersion: security.openshift.io/v1
metadata:
  name: nested-podman-scc
allowHostDirVolumePlugin: false
allowHostIPC: false
allowHostNetwork: false
allowHostPID: false
allowHostPorts: false
allowPrivilegeEscalation: true
allowPrivilegedContainer: false
allowedCapabilities:
- SETUID
- SETGID
defaultAddCapabilities: null
fsGroup:
  type: MustRunAs
groups: []
kind: SecurityContextConstraints
priority: null
readOnlyRootFilesystem: false
requiredDropCapabilities:
- KILL
- MKNOD
runAsUser:
  type: MustRunAsRange
seLinuxContext:
  type: MustRunAs
supplementalGroups:
  type: RunAsAny
users: []
volumes:
- configMap
- downwardAPI
- emptyDir
- persistentVolumeClaim
- projected
- secret
EOF
```

## Minimal Containerfile which enables builds

```bash
cat << EOF > image-builder.Containerfile
FROM registry.access.redhat.com/ubi9-minimal

ENV BUILDAH_ISOLATION=chroot
ENV HOME="/home/builder"

COPY --chown=0:0 entrypoint.sh /
RUN microdnf --disableplugin=subscription-manager install -y shadow-utils bash zsh podman podman-docker buildah skopeo fuse-overlayfs slirp4netns; \
    microdnf update -y ; \
    microdnf clean all ; \
    chgrp -R 0 /home ; \
    #
    # Setup for root-less podman
    #
    setcap cap_setuid+ep /usr/bin/newuidmap ; \
    setcap cap_setgid+ep /usr/bin/newgidmap ; \
    touch /etc/subgid /etc/subuid ; \
    chmod +x /entrypoint.sh ; \
    chmod -R g=u /etc/passwd /etc/group /etc/subuid /etc/subgid /home
WORKDIR ${HOME}
ENTRYPOINT [ "/entrypoint.sh" ]
EOF
```

### Entrypoint Script

```bash
cat << EOF > entrypoint.sh
#!/usr/bin/env bash

if [ ! -d "${HOME}" ]; then
  mkdir -p "${HOME}"
fi

if ! whoami &> /dev/null; then
  if [ -w /etc/passwd ]; then
    echo "builder:x:$(id -u):0:image builder:${HOME}:/bin/bash" >> /etc/passwd
    echo "builder:x:$(id -u):" >> /etc/group
  fi
fi
USER=$(whoami)
START_ID=$(( $(id -u)+1 ))
echo "${USER}:${START_ID}:2147483646" > /etc/subuid
echo "${USER}:${START_ID}:2147483646" > /etc/subgid
exec "$@"
EOF
```

### Sample Pod

```bash
cat << EOF | oc apply -f -
apiVersion: v1
kind: Pod
metadata:
 name: image-builder
 annotations:
   io.kubernetes.cri-o.Devices: "/dev/fuse"
spec:
  containers:
  - name: image-builder
    image: <your-image-builder-path>
    args: ["tail", "-f", "/dev/null"]
    securityContext:
      allowPrivilegeEscalation: true
      capabilities:
        add:
        - "SETUID"
        - "SETGID"
EOF
```
