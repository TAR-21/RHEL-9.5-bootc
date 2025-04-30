# RHEL 9.5 bootc ベース ベアメタルインストールガイド

## 🧭 概要

このガイドでは、`bootc-image-builder` を使って **RHEL 9.5 ベースのカスタムOSイメージ**を作成し、それをISOファイルに変換してベアメタルマシンへインストールする手順を解説します。Kickstartファイルは不要で、Red Hatの最新「Image Mode」方式に準拠しています。

---

## ✅ 1. 準備（作業用PC）

### 🔹 1-1. 必要なパッケージとログイン

```bash
sudo dnf install podman -y
podman login registry.redhat.io
```

---

### 🔹 1-2. 作業ディレクトリの作成

```bash
mkdir -p ~/rhel9.5-imagemode/output
cd ~/rhel9.5-imagemode
```

---

### 🔹 1-3. Containerfile の作成（RHEL bootc ベース）

`Containerfile` を以下の内容で作成します：

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

### 🔹 1-4. コンテナイメージのビルドとPush

```bash
podman build -t quay.io/USERNAME/custom-rhel95-bootc:1.0.0 .
podman push quay.io/USERNAME/custom-rhel95-bootc:1.0.0
```

> `USERNAME` は自分の quay.io ユーザー名に置き換えてください。

---

## ✅ 2. bootc-image-builder を使ってISOを作成

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

## ✅ 3. ベアメタルへのインストール

### 🔹 USBメモリへの書き込み

```bash
sudo dd if=~/rhel9.5-imagemode/output/install.iso of=/dev/sdX bs=4M status=progress
```

> `sdX` は対象のUSBデバイス名に置き換えてください。

### 🔹 BIOS/UEFIの設定

- **UEFIブート** を有効に
- **Secure Boot** は無効に

---

## ✅ 4. 起動後の確認

### 初期ログイン情報：

- **ユーザー名**：`demo`
- **パスワード**：`redhat`

### デプロイ確認コマンド：

```bash
rpm-ostree status
cat /etc/os-release
```

---

## ✅ よく使われるカスタマイズ・運用の例

| 機能                  | 設定例                                                                 |
|-----------------------|------------------------------------------------------------------------|
| SSH鍵の事前配置       | `RUN mkdir -p /home/demo/.ssh && echo "pubkey" > authorized_keys`       |
| MicroShift の導入     | `RUN rpm-ostree install microshift`                                     |
| SELinux再ラベル       | `RUN touch /.autorelabel`                                               |

---
