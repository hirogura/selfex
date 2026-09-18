# selfEx

セルフホストして使うファイラーです。 `/` 配下のファイルをブラウザから閲覧・編集できます。

SelfExplorer をホスト用に移植したものです。見た目・操作感はそのままに、インストール先・ポート・ブラウズ対象・名称を変更しています。

![ロゴ画像](images/selfexplorer-ph.png)

## インストール方法

### クイックインストール（推奨）

```bash
sudo bash -c "$(curl -fsSL https://raw.githubusercontent.com/hirogura/selfex/main/install-selfex1.sh)"
```

### 手動インストール

```bash
# 1. リポジトリをクローン
git clone https://github.com/hirogura/selfex.git /tmp/selfex
sudo rsync -a /tmp/selfex/ /opt/selfex/
rm -rf /tmp/selfex

# 2. 設定を編集 (必要に応じて)
# /opt/selfex/server/server.js の PORT, ROOT_DIR を編集
# フロントの表示ルートは /api/config から取得するため app.js の書き換えは不要

# 3. npm 依存関係をインストール
cd /opt/selfex/server
npm install --omit=dev

# 4. systemd サービスをセットアップ
sudo tee /etc/systemd/system/selfex.service <<'EOF'
[Unit]
Description=selfEx File Manager
After=network.target

[Service]
Type=simple
WorkingDirectory=/opt/selfex/server
ExecStart=/usr/bin/node server.js
Restart=always
RestartSec=5
Environment=NODE_ENV=production

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable selfex
sudo systemctl start selfex

# 5. Tailscale Serve で Tailnet 内のみ HTTPS 公開
sudo tailscale serve --bg --https=3362 "http://127.0.0.1:3362"
```

### インストールスクリプトを使用する場合

`install-selfex1.sh` をダウンロードして実行するか、上記のクイックインストールコマンドを実行してください。

## OnlyOffice フォント追加（オプション）

OnlyOffice で日本語などのフォントを正しく表示するには、`/Fonts/` にフォントファイル（ttf / otf / woff / woff2）を配置してから、以下のスクリプトを実行します。

```bash
sudo bash -c "$(curl -fsSL https://raw.githubusercontent.com/hirogura/selfex/main/add-fonts.sh)"
```

このスクリプトは以下の処理を行います:

1. `/Fonts/` を OnlyOffice コンテナの `/usr/share/fonts/custom` に read-only マウントで追加（`docker-compose.yml` を自動更新）
2. 変更前の `docker-compose.yml` をタイムスタンプ付きでバックアップ
3. コンテナを再作成（データは保持）
4. コンテナ内でフォントキャッシュを再生成（`fc-cache` / AllFonts.js / テーマ再生成）

注意:

- 実行前に `/Fonts/` にフォントファイルを配置しておいてください（ファイルが 1 件も無い場合は中断します）
- 再実行しても安全です（マウントが既にあればキャッシュ再生成のみ行います）
- 完了後はブラウザキャッシュをクリアしてから OnlyOffice でフォントを確認してください

## 要件

- Node.js 18 以上
- git
- systemd (推奨)
- OnlyOffice 連携には Docker

## アクセス

selfEx は `127.0.0.1` にのみバインドされているため、**Tailnet（Tailscale）内からしかアクセスできません**。LAN やインターネットからの直接アクセスはできません。

- サーバー上: http://localhost:3362
- Tailnet 内: https://`<hostname>.ts.net:3362` (Tailscale Serve が有効な場合)
- OnlyOffice: https://`<hostname>.ts.net:3322` (こちらも Tailnet 内のみ)

### ネットワーク構成

| サービス | バインド | 公開範囲 |
|---|---|---|
| selfEx (3362) | `127.0.0.1` | Tailnet 内のみ |
| OnlyOffice (3322) | `127.0.0.1` | Tailnet 内のみ |

### アンインストール方法

以下の手順でアンインストールできます。

```bash
# 1. サービスを停止・無効化
sudo systemctl stop selfex
sudo systemctl disable selfex

# 2. systemd サービスファイルを削除
sudo rm /etc/systemd/system/selfex.service
sudo systemctl daemon-reload

# 3. インストールディレクトリを削除
sudo rm -rf /opt/selfex

# 4. Tailscale Serve の設定を解除
tailscale serve --https=3362 off
```

注意点:

ブラウズ対象の `/` 配下の実データは削除されません — selfEx はフォルダの中身を表示していただけなので、実データはそのまま残ります。
OnlyOffice(/opt/onlyoffice)は、インストール時に「既存環境を再利用」を選んだ場合は他のサービス(nextExplorer等)とも共有されている可能性があります。selfEx専用に新規インストールしていて、かつ他で使っていないなら削除可能です

```bash
cd /opt/onlyoffice && sudo docker compose down
sudo rm -rf /opt/onlyoffice
```

ただし共有している場合は残しておいてください。

ポート 3362(selfEx本体)・3322(OnlyOffice)を他で使っていないか確認してから解放するのが安全です。

## デバイスのマウント / アンマウント

フォルダツリー上部の「デバイス」欄に、ホストのディスク・パーティション一覧（servEX と同じ方式）を表示します。

- ドライブ名をクリックするとパーティションを展開
- 未マウントのパーティションは「マウント」ボタンからマウント先（例: `/mnt/usb`）を指定してマウント
- マウント済みのパーティションはマウント先リンクのクリックでそのフォルダに移動、「アンマウント」ボタンでアンマウント

## バージョン

v.1.8.0
