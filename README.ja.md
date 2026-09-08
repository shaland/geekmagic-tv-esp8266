# GeekMagic SmallTV — カスタムファーム（shaland フォーク）

[English](README.md) · [日本語](README.ja.md)

[![License](https://img.shields.io/github/license/shaland/geekmagic-tv-esp8266)](/LICENSE)
[![Release](https://img.shields.io/github/v/release/shaland/geekmagic-tv-esp8266)](https://github.com/shaland/geekmagic-tv-esp8266/releases)

GeekMagic SmallTV 系デバイス（ESP8266）向けファーム。GeekMagic の HTTP API 互換。

> [!NOTE]
> **系譜:** [bvweerd/geekmagic-tv-esp8266](https://github.com/bvweerd/geekmagic-tv-esp8266)
> → [aydarik/geekmagic-tv-esp8266](https://github.com/aydarik/geekmagic-tv-esp8266)
> → このフォーク。元の実装をされた [@bvweerd](https://github.com/bvweerd) と
> [@aydarik](https://github.com/aydarik) に感謝 ❤️
>
> これは**個人フォーク**で、下記の機能を少し足しただけのものです。upstream と同期する予定はなく、
> ベースファーム自体の issue はここではなく upstream へお願いします。

![Clock](/assets/photo_clock.jpg)

> [!WARNING]
> **SmallTV** / **SmallTV-Ultra** は ESP8266。**SmallTV-Pro** は ESP32 で**非対応**。
> カスタムファームの書き込みは自己責任で。

## このフォークの追加点

| 変更 | 詳細 |
|---|---|
| **自動ローテーション** | 時計と `/image` 内の全 JPEG（名前順）をタイマーで巡回。外部の制御なしで回転ダッシュボードになる。間隔は `/set?rotateSec=N`（秒、`0` = 無効）または Web UI。物理ボタンも同じ巡回を1コマ進める。ローテ稼働中は `/set?img=`（`timeout` なし）が画像を差し替えるだけで画面を奪わない — 定期的な画像 push が巡回を邪魔しない。 |
| **時計の日付フォーマットを設定可能に** | `/set?dateFmt=<urlencoded strftime>` または Web UI。既定 `%Y/%m/%d %a`（例 `2026/09/08 Mon`）。以前は `%d-%m-%Y` 固定だった。 |
| **バグ修正** | `handleOTAForm()` が OTA ページを誤った `Content-Length` で送出していた／`settingsValidate()` が `defaultTheme` のつもりで `brightness` を書いていた。 |

追加した設定は `app.json`（`rotateSec`, `dateFmt`）と Web UI の *Settings* カードに出ます。
設定構造体が変わったため、**旧バージョンからのアップグレード時は設定がリセット**されます。

### Home Assistant ダッシュボード化

このファーム自体には Home Assistant クライアント機能はありません。
[aydarik/hass-geekmagic](https://github.com/aydarik/hass-geekmagic) でも使われている方式は、
HA 側で HTML を 240×240 画像にレンダリングして `/doUpload?dir=/image/` へ push するというもの。
これと上記 `rotateSec` を組み合わせると `/image/*.jpg` の各枚がローテーションのページになります。
時計ページはネイティブのまま。

## 🛠️ インストール

### 初回書き込み（UART 必須）

工場出荷デバイスは純正ファームなので、**初回**はシリアル経由で書き込む必要があります。

1. USB/シリアルで接続。
2. [最新リリース](https://github.com/shaland/geekmagic-tv-esp8266/releases)の `firmware.bin` を
   [web.esphome.io](https://web.esphome.io/)（または esptool）で書き込む。
3. 配線・ピン配置は upstream の
   [FLASHING.md](https://github.com/bvweerd/geekmagic-tv-esp8266/blob/dev/FLASHING.md) 参照。

<details>
<summary>見た目は雑だが動く 🫢</summary>

![Flashing 1](/assets/photo_flash_1.jpg) ![Flashing 2](/assets/photo_flash_2.jpg)

</details>

### 初回起動

1. デバイスは AP モードで起動 — 画面に SSID とパスワードが出る。
2. 接続して `http://192.168.4.1` を開き、Wi-Fi 情報を入力。
3. 再起動して自宅ネットワークに接続。IP は起動時と時計画面上部に表示される。

### OTA アップデート（初回書き込み以降）

デバイスの IP を開く → 下部の **Firmware Update (OTA)** → `firmware.bin` をアップロード。
失敗時は現ファームを維持。

## ビルド

CI（`.github/workflows/release.yml`）が `X.Y.Z` 形式のタグ push で `firmware.bin` をビルドし、
**下書きリリース**に添付します。公開するには `gh release edit <tag> --draft=false --latest`。

ローカルビルド（PlatformIO）:

```bash
mkdir -p src/generated
gzip -9 -c res/index.html > src/generated/index.html.gz && xxd -i src/generated/index.html.gz > src/generated/index_html.h
gzip -9 -c res/ota.html   > src/generated/ota.html.gz   && xxd -i src/generated/ota.html.gz   > src/generated/ota_html.h
pio run -e nodemcuv2
```

## 💬 対応文字

![Charset](assets/charset.png)

## 📡 HTTP API

```bash
# 自動ローテーション間隔（秒、0 = 無効）        [fork]
curl "http://DEVICE_IP/set?rotateSec=8"

# 時計の日付フォーマット（urlencoded strftime） [fork]
curl "http://DEVICE_IP/set?dateFmt=%25Y%2F%25m%2F%25d%20%25a"

# 輝度 / テーマ / 秒表示
curl "http://DEVICE_IP/set?brt=50"
curl "http://DEVICE_IP/set?theme=1"
curl "http://DEVICE_IP/set?sec=true"

# タイムゾーン（POSIX TZ）
curl "http://DEVICE_IP/set?tz=JST-9"

# カスタムメッセージ / ゲージ / スティッキーノート
curl "http://DEVICE_IP/set?msg=Hello%20world!&sbj=Notification&style=center&timeout=10"
curl 'http://DEVICE_IP/set?msg=21.4%2F40%20%E2%84%83&sbj=Living%20room&style=big_num&timeout=60'
curl "http://DEVICE_IP/set?note=+8%E2%84%83%20cloudy&rpm=6&timeout=3600"

# カウントダウン
curl "http://DEVICE_IP/set?cnt=2026-02-19T09%3A30&sbj=Next%20call&timeout=5"

# 画像
curl -F "file=@photo.jpg" "http://DEVICE_IP/doUpload?dir=/image/"
curl "http://DEVICE_IP/set?img=/image/photo.jpg&timeout=30"
curl "http://DEVICE_IP/filelist?dir=/image/"

# ステータス
curl "http://DEVICE_IP/v.json"      # ファームバージョン
curl "http://DEVICE_IP/app.json"    # デバイス状態（rotateSec, dateFmt 含む）
curl "http://DEVICE_IP/space.json"  # ファイルシステム
```

![Custom Message](/assets/photo_message.jpg) ![Gauge](/assets/photo_gauge.jpg) ![Sticky Note](/assets/photo_note.jpg)

## 📜 ライセンス

MIT — [LICENSE](/LICENSE) 参照。upstream と同じ。
