FROM registry.opensuse.org/opensuse/bci/golang:1.25 AS builder
WORKDIR /app
COPY  --chown=1001:0 ./ ./
RUN zypper -n in libvirt-devel
RUN go build -ldflags="-w -s" -o virt-v2v-monitor ./cmd/virt-v2v-monitor/virt-v2v-monitor.go
RUN go build -ldflags="-w -s" -o image-converter ./cmd/image-converter/image-converter.go
RUN go build -ldflags="-w -s" -o virt-v2v-wrapper ./cmd/virt-v2v/entrypoint.go

## OBS build https://build.opensuse.org/projects/isv:Rancher:Harvester:Containers:Dev/packages/guestfs-appliance-container/files/Dockerfile?expand=1
FROM registry.opensuse.org/isv/rancher/harvester/containers/dev/15.7/guestfs-appliance:latest

COPY --from=builder /app/virt-v2v-monitor /usr/local/bin/virt-v2v-monitor

COPY --from=builder /app/image-converter /usr/local/bin/image-converter

COPY --from=builder /app/virt-v2v-wrapper /usr/bin/virt-v2v-wrapper

ENTRYPOINT ["/usr/bin/virt-v2v-wrapper"]