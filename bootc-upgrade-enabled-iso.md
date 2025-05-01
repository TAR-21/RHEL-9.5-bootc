
# `bootc upgrade` 対応のための ISO ビルド設定（バージョン 1.1.0）

## なぜ `bootc switch` を使う必要があるのか？

`bootc-image-builder` は ISO 作成時に `skopeo` を使ってコンテナイメージを展開し、ファイルを ISO に注入しますが、  
この時点では **どのイメージからインストールされたか（イメージURLやタグ）** の情報が OS に保存されません。

そのため、インストール後に以下のような問題が起こります：

```
bootc upgrade
→ エラー：アップグレード元のイメージが不明
```

---

## 解決策：Kickstart の `%post` セクションで `bootc switch` を実行

```toml
[post]
inline = '''
bootc switch --mutate-in-place --transport registry quay.io/USERNAME/custom-rhel95-bootc:1.1.0
'''
```

これにより、インストールされたシステムが自分のベースイメージを記録し、  
将来 `bootc upgrade` が `quay.io/USERNAME/custom-rhel95-bootc:1.1.0` から更新を取得できるようになります。

---

## 完全な `config.toml`（v1.1.0 対応）

```toml
[ostree]
ref = "rhel/9/x86_64/edge"
url = "quay.io/USERNAME/custom-rhel95-bootc:1.1.0"
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
bootc switch --mutate-in-place --transport registry quay.io/USERNAME/custom-rhel95-bootc:1.1.0
'''
```

※ `USERNAME` は Quay.io 上のご自身の名前に置き換えてください。

---

## ISO 作成コマンド（`config.toml` を使う）

```bash
sudo podman run --rm -it --privileged \
  --pull=always \
  --security-opt label=type:unconfined_t \
  -v /var/lib/containers/storage:/var/lib/containers/storage \
  -v ~/rhel9.5-imagemode/config-bootc-1.1.0.toml:/config.toml \
  -v ~/rhel9.5-imagemode/output:/output \
  registry.redhat.io/rhel9/bootc-image-builder:latest \
  --type iso \
  --config /config.toml \
  quay.io/USERNAME/custom-rhel95-bootc:1.1.0
```
