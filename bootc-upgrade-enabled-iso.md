# `bootc upgrade` 対応のための ISO ビルド設定

## なぜ `bootc switch` を使う必要があるのか？

`bootc-image-builder` は ISO 作成時に `skopeo` を使ってコンテナイメージを展開しますが、  
この方法では **インストールされたシステムに元のイメージ情報（URLやタグ）が記録されません**。

結果として、`bootc upgrade` が機能しなくなります：

---

## 解決策：Kickstart の `%post` セクションで `bootc switch` を実行

以下を `config.toml` に記述することで、インストール時に元イメージ情報を記録できます：

```toml
[post]
inline = '''
bootc switch --mutate-in-place --transport registry quay.io/USERNAME/custom-rhel95-bootc:1.0.0
'''
```

---

## `bootc switch` 対応済み `config.toml`（バージョン 1.0.0）

```toml
[ostree]
ref = "rhel/9/x86_64/edge"
url = "quay.io/USERNAME/custom-rhel95-bootc:1.0.0"
boot_location = "local"
contenturl = "container"

[installation]
user = "demo"
password = "redhat"
hostname = "bootc-demo-v1"
root-password = "redhat"
ssh-key = ""

[services]
enabled = ["sshd", "NetworkManager"]

[post]
inline = '''
bootc switch --mutate-in-place --transport registry quay.io/USERNAME/custom-rhel95-bootc:1.0.0
'''
```

---

## ISO 作成コマンド

```bash
sudo podman run --rm -it --privileged \
  --pull=always \
  --security-opt label=type:unconfined_t \
  -v /var/lib/containers/storage:/var/lib/containers/storage \
  -v ~/rhel9.5-imagemode/config-bootc-1.0.0.toml:/config.toml \
  -v ~/rhel9.5-imagemode/output:/output \
  registry.redhat.io/rhel9/bootc-image-builder:latest \
  --type iso \
  --config /config.toml \
  quay.io/USERNAME/custom-rhel95-bootc:1.0.0
```

