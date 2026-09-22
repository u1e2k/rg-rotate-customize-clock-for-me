# rg-rotate-customize-clock-for-me

Anbernic 製 Android ゲーム機 **RG Rotate** のロック画面に備わっているカスタム時計機能（`customize_clock`）の内部仕様調査および、カスタム時計作成のためのリポジトリです。

---

## 📖 経緯・調査プロセス

公式のドキュメントや詳細な仕様が公開されていなかったため、接続された実機からロック画面の描画を担う `SystemUI.apk` を抽出し、リバースエンジニアリング（静的コード解析）によって内部仕様を特定しました。

### 1. APK の抽出
ADB 経由でパッケージマネージャから `com.android.systemui` のインストールパスを取得し、ローカルに pull しました。
```bash
adb shell pm path com.android.systemui
# 出力: package:/system_ext/priv-app/SystemUI/SystemUI.apk

adb pull /system_ext/priv-app/SystemUI/SystemUI.apk ./SystemUI.apk
```

### 2. 逆コンパイルとコード特定
`jadx-cli` を導入し、DEX バイトコードを Java ソースコードに逆コンパイル、およびリソース XML を展開しました。
- 検索キーワード `customize_clock` / `clock_background.png` から以下のコア実装を特定：
  - **`com.ylm.main.keyguard.lockclock.CustomizeClockView`**: カスタム画像の読み込み処理
  - **`com.ylm.main.keyguard.lockclock.BasePictureClockView`**: 時計針の回転角度計算・定期再描画ループ
  - **`com.android.keyguard.KeyguardClockSwitch`**: ロック画面時計タイプの切り替えと監視
  - **`res/layout/customize_round_clock.xml`**: カスタム時計のレイアウト構造

### 3. 仕様の解明
ソースコードおよびレイアウト定義の解析により、画像仕様、回転軸、読み込みファイル名、運針仕様の全容が判明しました。詳細な技術仕様は [SPEC.md](SPEC.md) にまとめています。

---

## 🔍 主要仕様サマリー

### ファイル仕様一覧

| ファイル名 | 役割 | 推奨解像度 | 必須要件 / 特記事項 |
| :--- | :--- | :--- | :--- |
| `clock_background.png` | 文字盤背景 | 500 x 500 px | 枠サイズ500x500px（超えた場合は `FIT_CENTER` で自動縮小） |
| `clock_hour.png` | 時針 | 500 x 500 px | **500x500px透過キャンバスの中心 (250, 250) に根本を置き、12時（上）に向けて描画** |
| `clock_minute.png` | 分針 | 500 x 500 px | 同上（500x500px透過キャンバス） |
| `clock_second.png` | 秒針 | 500 x 500 px | 同上（単数形 `second`。ファイルが存在しない場合は2針時計として動作） |

> **⚠️ 最重要ポイント（回転軸）:**  
> 針の回転軸（Pivot）は View の中心 `(Width / 2, Height / 2)` 固定で、座標指定などの設定機能はありません。  
> 針のみをトリミングした画像（例: 40x200px）を使うと針自身の中心でプロペラのように回転してしまうため、**必ず 500x500px の正方形キャンバスの中央 (250, 250) に回転中心を合わせた透過画像** を作成してください。

### 配置先ディレクトリ
```
/sdcard/customize_clock/
（または /storage/emulated/0/customize_clock/）
```
※設定ファイル（JSON、XML 等）やフォントファイルは一切存在せず、上記 4 点の PNG のみ読み込まれます。

### 動作・運針ロジック
- **運針方式**: **ステップ運針（1秒ごとにジャンプ）** です。スイープ運針ではありません。
  - 秒針: 1秒ごとに 6度 ずつ回転
  - 分針: 1秒ごとに 0.1度 ずつ回転
  - 時針: 1分ごとに 0.5度 ずつ回転
- **更新頻度**: 約 1000ms（1秒）に1回

---

## 🛠️ お役立ち ADB コマンド集

### 1. カスタム時計を有効化する
RG Rotate の時計設定（`lock_clock_type = 7` がカスタム時計）を変更します。
```bash
adb shell settings put system lock_clock_type 7
```

### 2. 画像ファイルを端末へ転送する
PC 上で作成した画像を端末へプッシュします。
```bash
# 端末側のディレクトリ作成
adb shell mkdir -p /sdcard/customize_clock

# 画像の転送（例: local_assets フォルダ内の PNG を転送）
adb push ./clock_background.png /sdcard/customize_clock/
adb push ./clock_hour.png /sdcard/customize_clock/
adb push ./clock_minute.png /sdcard/customize_clock/
adb push ./clock_second.png /sdcard/customize_clock/
```

### 3. 画像変更を即時反映（ホットリロード）
画像を差し替えた後、端末を再起動することなく SystemUI に画像を再読み込みさせます。
```bash
adb shell settings put system lock_clock_customize_refresh 1
```

---

## 📂 関連ドキュメント
- [SPEC.md](SPEC.md): クラス構成・ソースコード引用を含む詳細な内部実装仕様書
