```bash
podman pull registry.access.redhat.com/ubi8/ubi-minimal
```

```bash
cat > Dockerfile <<EOF
FROM registry.access.redhat.com/ubi8/ubi-minimal
USER 1001
COPY vmware-vix-disklib-distrib /vmware-vix-disklib-distrib
RUN mkdir -p /opt
ENTRYPOINT ["cp", "-r", "/vmware-vix-disklib-distrib", "/opt"]
EOF
```

```bash
podman build . -t <registry_route_or_server_path>/vddk:<tag>
```

```bash  
podman push <registry_route_or_server_path>/vddk:<tag>
```
