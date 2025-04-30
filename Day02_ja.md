# bootc upgrade / rollback シナリオ（nginx 組み込み、RHEL 9.5 ベース）

このドキュメントは、nginx を組み込んだ bootc イメージ（v2.0.0）を作成し、ベアメタル環境に適用して `bootc switch` と `rpm-ostree rollback` の動作を体験する手順をまとめたものです。

---

## 概要

以下の操作を体験できます：

- nginx を含む bootc イメージ（v2.0.0）をビルド
- bootc 環境で `bootc switch` による切り替えを実施
- `rpm-ostree rollback` によるバージョン復元を体験

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

### ステップ 3：digest を変えるため `--no-cache` 付きでビルド・push

```bash
podman build --no-cache -t quay.io/<USERNAME>/custom-rhel95-bootc:2.0.0 .
podman push quay.io/<USERNAME>/custom-rhel95-bootc:2.0.0
```

`<USERNAME>` は任意の Quay.io アカウント名に置き換えてください。

---

## 🖧 bootc インストール済みマシン側（ベアメタルまたはVM）

### ステップ 4：現在の状態を確認

```bash
rpm-ostree status
```

### ステップ 5：bootc switch による切り替え

```bash
sudo bootc switch quay.io/<USERNAME>/custom-rhel95-bootc:2.0.0
sudo systemctl reboot
```

### ステップ 6：nginx が起動しているか確認

```bash
systemctl status nginx
curl http://localhost
```

→ nginx のデフォルトページが表示されれば成功です。

---

### ステップ 7：前のバージョンに戻す（rollback）

```bash
sudo rpm-ostree rollback
sudo systemctl reboot
```

再起動後に以下で確認：

```bash
rpm-ostree status
```
