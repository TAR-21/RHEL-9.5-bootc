# bootc upgrade / rollback scenario with nginx (RHEL 9.5)

This guide walks through how to build a custom bootc image with nginx, deploy it to a bare-metal system, and test both `bootc switch` and `rpm-ostree rollback`.

## Overview

You will:
- Build a custom bootc image (`v2.0.0`) with nginx pre-installed
- Deploy it to a bootc-installed machine using `bootc switch`
- Test rollback to a previous version using `rpm-ostree rollback`

---

## On Workstation (Podman build environment)

### Step 1: Prepare working directory

```bash
mkdir -p ~/rhel9.5-imagemode-v2
cd ~/rhel9.5-imagemode-v2
```

### Step 2: Create `Containerfile` for version 2.0.0

Save as: `Containerfile`

```Dockerfile
FROM registry.redhat.io/rhel9/rhel-bootc:latest

RUN rpm-ostree install \
    NetworkManager \
    openssh-server \
    nginx && \
    useradd demo && echo "demo:redhat" | chpasswd && \
    usermod -aG wheel demo && \
    echo "%wheel ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers && \
    systemctl enable sshd && \
    systemctl enable nginx && \
    mkdir -p /var/log/nginx && chown nginx:nginx /var/log/nginx && \
    mkdir -p /var/lib/nginx/tmp/client_body && chown -R nginx:nginx /var/lib/nginx && \
    echo "bootc-demo-v2" > /etc/hostname && \
    rpm-ostree cleanup -m && \
    ostree container commit

LABEL bootc-image="true"
LABEL org.opencontainers.image.title="custom-rhel95-bootc"
LABEL org.opencontainers.image.version="2.0.0"
CMD ["/usr/lib/systemd/systemd"]
```

### Step 3: Build & push the image (force new digest with `--no-cache`)

```bash
podman build --no-cache -t quay.io/<USERNAME>/custom-rhel95-bootc:2.0.0 .
podman push quay.io/<USERNAME>/custom-rhel95-bootc:2.0.0
```

Replace `<USERNAME>` with your Quay.io username.

---

## On bootc-installed machine (bare metal or VM)

### Step 4: Check current image version

```bash
rpm-ostree status
```

### Step 5: Switch to the new image

```bash
sudo bootc switch quay.io/<USERNAME>/custom-rhel95-bootc:2.0.0
sudo systemctl reboot
```

### Step 6: Validate nginx service

```bash
systemctl status nginx
curl http://localhost
```

### Step 7: Roll back to previous version

```bash
sudo rpm-ostree rollback
sudo systemctl reboot
```

Then check:

```bash
rpm-ostree status
```

---
