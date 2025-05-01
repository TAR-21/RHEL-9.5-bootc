# bootc upgrade / rollback scenario with httpd (RHEL 9.5)

This guide walks through how to build a custom bootc image with httpd, deploy it to a bare-metal system, and test both `bootc switch` and `bootc rollback`.

## Overview

You will:
- Build a custom bootc image (`v2.0.0`) with httpd pre-installed
- Deploy it to a bootc-installed machine using `bootc switch`
- Test rollback to a previous version using `bootc rollback`

---

## On Workstation (Podman build environment)

### Step 1: Prepare working directory

```bash
mkdir -p ~/rhel9.5-imagemode-v2
cd ~/rhel9.5-imagemode-v2
```

### Step 2: Create Containerfile for version 2.0.0

Save as: `Containerfile`

```Dockerfile
FROM registry.redhat.io/rhel9/rhel-bootc:latest

# Install required packages using DNF
RUN dnf install -y \
    NetworkManager \
    openssh-server \
    httpd && \
    dnf clean all

# Create demo user, set password, configure sudo, hostname, and enable services
RUN useradd demo && echo "demo:redhat" | chpasswd && \
    usermod -aG wheel demo && \
    echo "%wheel ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers && \
    echo "bootc-demo-v2" > /etc/hostname && \
    systemctl enable sshd && \
    systemctl enable httpd

# Move web content to a versioned path and adjust Apache configuration
RUN mv /var/www /usr/share/www && \
    sed -ie 's,/var/www,/usr/share/www,' /etc/httpd/conf/httpd.conf && \
    sed -i 's,/var/www,/usr/share/www,g' /etc/httpd/conf.d/*.conf && \
    rm -rf /usr/share/httpd/noindex

# Embed custom index.html content directly into the image
RUN mkdir -p /usr/share/www/html && \
    echo '<!DOCTYPE html><html><head><title>bootc Apache Demo</title></head><body><h1>Welcome to Apache on bootc 2.0.0</h1><p>This page is served from an immutable image using RHEL bootc.</p></body></html>' > /usr/share/www/html/index.html

# Ensure log directory exists and Apache can write logs
RUN mkdir -p /var/log/httpd && \
    ln -s /var/log/httpd /etc/httpd/logs && \
    chown -R apache:apache /var/log/httpd

# Validate image compatibility with bootc
RUN bootc container lint

# Expose HTTP port
EXPOSE 80

# Image metadata
LABEL bootc-image="true"
LABEL org.opencontainers.image.title="custom-rhel95-bootc"
LABEL org.opencontainers.image.version="2.0.0"

# Set systemd as the default entrypoint
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

### Step 6: Validate httpd service

```bash
systemctl status httpd
curl http://localhost
```

---
#### Cause and On-the-Spot Fix for Apache Startup Failure (bootc Environment)

##### Problem Summary

Apache HTTP Server fails to start when running `systemctl start httpd` with the following error:

```
AH02291: Cannot access directory '/etc/httpd/logs/' for main error log
AH00014: Configuration check failed
```

This means Apache is unable to access the log directory it expects to use.


##### Root Cause

- `/etc/httpd/logs/` is supposed to be a symbolic link to `/var/log/httpd`.
- In bootc or OSTree-based systems, `/var` is ephemeral and recreated at each boot.
- Therefore, the symbolic link or the log directory may not exist after a reboot, causing Apache to fail when trying to write logs.


##### On-the-Spot Recovery Steps

```bash
sudo mkdir -p /var/log/httpd
sudo ln -s /var/log/httpd /etc/httpd/logs
sudo chown -R apache:apache /var/log/httpd

# Check Apache config and restart
sudo httpd -t
sudo systemctl restart httpd
```

### Step 7: Roll back to previous version

```bash
sudo bootc rollback
sudo systemctl reboot
```

Then check:

```bash
rpm-ostree status
```
