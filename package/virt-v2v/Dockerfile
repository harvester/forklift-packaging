# Self-contained virt-v2v conversion image.
#
# Previously this was layered on the OBS guestfs-appliance image (SLE 15 SP7),
# which ships virt-v2v 2.7.7 and no Windows firstboot helpers:
#   - harvester/harvester#11657: virt-v2v 2.7.7 fails Windows Server 2022
#     inspection with an unhandled Hivex.Error
#   - harvester/harvester#11775: /usr/share/virt-tools is empty, so Windows
#     firstboot (driver install, guest agent) cannot be set up
#   - harvester/harvester#11851: SLES 15 guests fail conversion because a
#     perl-Bootloader failure is fatal in virt-v2v
#
# Base: SLE 15 SP7 only has virt-v2v 2.7.7, and Leap/SLE 16.0 ships virt-v2v
# 2.8.0 with a libvirt built without the 'vpx' driver, so vSphere sources fail
# with "libvirt was built without the 'vpx' driver". Tumbleweed provides
# virt-v2v 2.12 / libguestfs 1.60 with vpx; pin a digest for releases.
ARG BASE_IMAGE=registry.opensuse.org/opensuse/tumbleweed:latest
# Source repo matching BASE_IMAGE, used to rebuild virt-v2v with patches.
ARG SOURCE_REPO=http://download.opensuse.org/source/tumbleweed/repo/oss/
# virtio-win (noarch) as imported into OBS from CentOS.
ARG EXTRA_REPO=https://download.opensuse.org/repositories/isv:/Rancher:/Harvester:/ExtraPackages:/Dev/15.7/
ARG RHSRVANY_REF=v1.1

FROM registry.opensuse.org/opensuse/bci/golang:1.26 AS builder
ENV GOTOOLCHAIN=auto
WORKDIR /app
COPY  --chown=1001:0 ./ ./
RUN zypper -n in libvirt-devel
RUN go build -ldflags="-w -s" -o virt-v2v-monitor ./cmd/virt-v2v-monitor/virt-v2v-monitor.go
RUN go build -ldflags="-w -s" -o image-converter ./cmd/image-converter/image-converter.go
RUN go build -ldflags="-w -s" -o virt-v2v-wrapper ./cmd/virt-v2v/entrypoint.go

# rhsrvany.exe + pnp_wait.exe: Windows service wrapper virt-v2v uses to run
# firstboot scripts (#11775). Built from source with mingw, as Fedora does.
FROM ${BASE_IMAGE} AS srvany
ARG RHSRVANY_REF
RUN zypper -n --gpg-auto-import-keys in mingw32-cross-gcc mingw32-cross-binutils \
        mingw32-filesystem autoconf automake make git-core && \
    git clone --depth 1 -b ${RHSRVANY_REF} https://github.com/rwmjones/rhsrvany /src && \
    cd /src && autoreconf -i && \
    ./configure --host=i686-w64-mingw32 && make && \
    install -D -m 0755 RHSrvAny/rhsrvany.exe /out/rhsrvany.exe && \
    install -D -m 0755 pnp_wait/pnp_wait.exe /out/pnp_wait.exe

# virt-v2v rebuilt from the distribution source package with our patches.
FROM ${BASE_IMAGE} AS v2vbuild
ARG SOURCE_REPO
COPY harvester/virt-v2v/*.patch /usr/src/packages/SOURCES/
RUN zypper -n ar -f ${SOURCE_REPO} harvester-source && \
    zypper -n --gpg-auto-import-keys ref && zypper -n in rpm-build && \
    zypper -n --gpg-auto-import-keys si virt-v2v && \
    cd /usr/src/packages/SPECS && n=9000 && \
    for p in $(cd ../SOURCES && ls 0*-*.patch | sort); do \
      sed -i "0,/^Source0*:/s//Patch${n}: ${p}\n&/" virt-v2v.spec; \
      if ! grep -q '^%autosetup' virt-v2v.spec; then \
        sed -i "0,/^%setup.*/s//&\n%patch -P${n} -p1/" virt-v2v.spec; fi; \
      n=$((n+1)); \
    done && \
    rpmbuild -bb --nocheck virt-v2v.spec

FROM ${BASE_IMAGE}
ARG EXTRA_REPO
ENV LIBGUESTFS_DEBUG=1 LIBGUESTFS_TRACE=1
# LIBGUESTFS_BACKEND=direct works around RHEL-104684 (appliance build) and
# RHEL-105490 (runtime).
ENV LIBGUESTFS_BACKEND=direct

# The appliance is assembled by supermin from host packages, so install the
# full set (recommends included): with a minimal set inspection fails, e.g.
# "inspect_os: execvp: file: No such file or directory".
# harvester-extra is only for virtio-win. It also carries SLE 15 SP7 builds of
# virt-v2v/libguestfs that outrank Tumbleweed's by version but cannot install
# on it (libxml2.so.2), so give it a lower priority than the base repos.
RUN zypper -n ar -f -p 150 ${EXTRA_REPO} harvester-extra && \
    zypper -n --gpg-auto-import-keys ref && \
    zypper -n in \
        virt-v2v virtio-win libvirt-client qemu-x86 qemu-img qemu-tools \
        qemu-ovmf-x86_64 qemu-seabios kernel-default supermin \
        libguestfs libguestfs-devel libguestfs-appliance libguestfs-xfs \
        libguestfs-winsupport btrfsprogs ntfs-3g ntfsprogs lvm2 device-mapper \
        nbdkit nbdkit-server nbdkit-curl-plugin nbdkit-basic-plugins \
        nbdkit-basic-filters nbdkit-nbd-plugin nbdkit-ssh-plugin \
        nbdkit-vddk-plugin nbdkit-python-plugin \
        attr binutils bzip2 cpio curl diffutils file glibc-locale-base gzip \
        iproute2 less libxml2-tools lsscsi lzop procps psmisc tar xz zstd && \
    for p in exfatprogs f2fs-tools scrub sleuthkit squashfs syslinux zerofree; do \
      zypper -n in $p || echo "optional appliance package $p not available"; \
    done && \
    depmod $(ls /lib/modules/ | tail -n1) && \
    zypper -n clean -a

COPY --from=v2vbuild /usr/src/packages/RPMS/x86_64/ /tmp/v2vrpms/
RUN rpm -Uvh --force --nodeps $(ls /tmp/v2vrpms/virt-v2v-[0-9]*.rpm) && \
    rm -rf /tmp/v2vrpms

RUN mkdir -p /disks /usr/lib64/guestfs/appliance && \
    cd /usr/lib64/guestfs/appliance && \
    libguestfs-make-fixed-appliance . && \
    qemu-img convert -c -O qcow2 root root.qcow2 && \
    mv -vf root.qcow2 root

COPY --from=srvany /out/ /usr/share/virt-tools/
COPY --from=builder /app/virt-v2v-monitor /usr/local/bin/virt-v2v-monitor
COPY --from=builder /app/image-converter /usr/local/bin/image-converter
COPY --from=builder /app/virt-v2v-wrapper /usr/bin/virt-v2v-wrapper

ENTRYPOINT ["/usr/bin/virt-v2v-wrapper"]
