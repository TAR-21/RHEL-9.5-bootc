# bootc upgrade / rollback シナリオ（httpd 組み込み、RHEL 9.5 ベース）

このドキュメントは、httpd を組み込んだ bootc イメージ（v2.0.0）を作成し、ベアメタル環境に適用して `bootc switch` と `sudo bootc rollback` の動作を体験する手順をまとめたものです。

---

## 概要

以下の操作を体験できます：

- httpd を含む bootc イメージ（v2.0.0）をビルド
- bootc 環境で `bootc switch` による切り替えを実施
- `sudo bootc rollback` によるバージョン復元を体験

---

## 作業用PC側（Podmanでビルド）

### ステップ 1：作業ディレクトリを用意

```bash
mkdir -p ~/rhel9.5-imagemode-v2
cd ~/rhel9.5-imagemode-v2
```

### ステップ 2：`Containerfile` を作成（バージョン 2.0.0）

ファイル名：`Containerfile`

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

### ステップ 3：digest を変えるため `--no-cache` 付きでビルド・push

```bash
podman build --no-cache -t quay.io/<USERNAME>/custom-rhel95-bootc:2.0.0 .
podman push quay.io/<USERNAME>/custom-rhel95-bootc:2.0.0
```

`<USERNAME>` は任意の Quay.io アカウント名に置き換えてください。

---

## bootc インストール済みマシン側（ベアメタルまたはVM）

### ステップ 4：現在の状態を確認

```bash
rpm-ostree status
```

### ステップ 5：bootc switch による切り替え

```bash
sudo bootc switch quay.io/<USERNAME>/custom-rhel95-bootc:2.0.0
sudo systemctl reboot
```

### ステップ 6：httpd が起動しているか確認

```bash
systemctl status httpd
curl http://localhost
```

→ httpd のデフォルトページが表示されれば成功です。

---

### ステップ 7：前のバージョンに戻す（rollback）

```bash
sudo sudo bootc rollback
sudo systemctl reboot
```

再起動後に以下で確認：

```bash
rpm-ostree status
```
