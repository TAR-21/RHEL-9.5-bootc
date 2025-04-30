# RHEL 9.5 bootc-based Bare Metal Installation Guide

## 🧭 Overview

This guide explains how to create a **bootc-based, RHEL 9.5 custom OS image** using `bootc-image-builder`, convert it into an ISO file, and install it on a bare-metal machine. No Kickstart file is required, and this method is fully compatible with Red Hat's latest **Image Mode** deployment model.

---

## ✅ 1. Preparation (on your workstation)

### 🔹 1-1. Requirements

```bash
sudo dnf install podman -y
podman login registry.redhat.io
```

---

### 🔹 1-2. Create working directories

```bash
mkdir -p ~/rhel9.5-imagemode/output
cd ~/rhel9.5-imagemode
```

---

### 🔹 1-3. Create a `Containerfile` (RHEL bootc base)

Create a file named `Containerfile` with the following content:

```Dockerfile
FROM registry.redhat.io/rhel9/rhel-bootc:latest

RUN rpm-ostree install \
    NetworkManager \
    openssh-server && \
    useradd demo && echo "demo:redhat" | chpasswd && \
    systemctl enable sshd && \
    echo "bootc-demo" > /etc/hostname && \
    rpm-ostree cleanup -m && \
    ostree container commit

LABEL bootc-image="true"
LABEL org.opencontainers.image.title="custom-rhel95-bootc"
LABEL org.opencontainers.image.version="1.0.0"
CMD ["/usr/lib/systemd/systemd"]
```

---

### 🔹 1-4. Build and push the container image

```bash
podman build -t quay.io/USERNAME/custom-rhel95-bootc:1.0.0 .
podman push quay.io/USERNAME/custom-rhel95-bootc:1.0.0
```

> Replace `USERNAME` with your actual quay.io username.

---

## ✅ 2. Generate a bootable ISO using `bootc-image-builder`

```bash
sudo podman run --rm -it --privileged \
  --pull=always \
  --security-opt label=type:unconfined_t \
  -v /var/lib/containers/storage:/var/lib/containers/storage \
  -v ~/rhel9.5-imagemode/output:/output \
  registry.redhat.io/rhel9/bootc-image-builder:latest \
  --type iso \
  --output /output \
  quay.io/USERNAME/custom-rhel95-bootc:1.0.0
```

---

## ✅ 3. Install on bare-metal hardware

### 🔹 Write the ISO to a USB stick

```bash
sudo dd if=~/rhel9.5-imagemode/output/install.iso of=/dev/sdX bs=4M status=progress
```

> Replace `sdX` with your actual USB device name.

### 🔹 BIOS/UEFI settings

- Enable **UEFI Boot**
- Disable **Secure Boot**

---

## ✅ 4. After booting: verify system

### Default credentials:

- **Username**: `demo`
- **Password**: `redhat`

### Verify the deployment:

```bash
rpm-ostree status
cat /etc/os-release
```

---

## ✅ Common enhancements and operational tips

| Feature             | Example                                                                 |
|---------------------|-------------------------------------------------------------------------|
| Preload SSH key     | `RUN mkdir -p /home/demo/.ssh && echo "pubkey" > authorized_keys`       |
| Add MicroShift      | `RUN rpm-ostree install microshift`                                     |
| Enable SELinux relabeling | `RUN touch /.autorelabel`                                        |

---
