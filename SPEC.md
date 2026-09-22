# RG Rotate ロック画面カスタム時計（customize_clock）内部実装仕様書

本書は、接続された Android 端末（RG Rotate）の `SystemUI.apk`（`/system_ext/priv-app/SystemUI/SystemUI.apk`）を逆コンパイル・静的解析し、カスタム時計機能（`customize_clock`）の内部実装および動作仕様をまとめた仕様書です。

---

## 1. 概要・関連コンポーネント

カスタム時計機能は以下のクラスおよびレイアウト XML によって構成されています。

| 構成要素 | パス / クラス名 | 役割 |
| :--- | :--- | :--- |
| **Java クラス** | `com.ylm.main.keyguard.lockclock.CustomizeClockView` | カスタム画像の読み込みと初期化 |
| **基底クラス** | `com.ylm.main.keyguard.lockclock.BasePictureClockView` | アナログ時計の描画・時刻計算・定期再描画制御 |
| **制御クラス** | `com.android.keyguard.KeyguardClockSwitch` | ロック画面上の時計切り替え・設定監視 |
| **レイアウト** | `res/layout/customize_round_clock.xml` | カスタム時計の View 階層定義 |

---

## 2. 調査項目別の詳細仕様

### ① 画像サイズ制限（500x500px）：超えた場合の挙動

#### ソースコードの該当箇所
- **`customize_round_clock.xml`**:
  ```xml
  <FrameLayout xmlns:android="http://schemas.android.com/apk/res/android"
      android:layout_gravity="center"
      android:layout_width="500px"
      android:layout_height="500px">
      <ImageView
          android:id="@+id/iv_background"
          android:layout_width="500px"
          android:layout_height="500px"/>
      <ImageView
          android:layout_gravity="center"
          android:id="@+id/iv_hour"
          android:layout_width="wrap_content"
          android:layout_height="wrap_content"/>
      <ImageView
          android:layout_gravity="center"
          android:id="@+id/iv_minute"
          android:layout_width="wrap_content"
          android:layout_height="wrap_content"/>
      <ImageView
          android:layout_gravity="center"
          android:id="@+id/iv_seconds"
          android:layout_width="wrap_content"
          android:layout_height="wrap_content"/>
  </FrameLayout>
  ```
- **`CustomizeClockView.java`**:
  ```java
  BitmapFactory.decodeFile(str2); // オプションなしでデコード
  ```

#### 挙動の分析
1. **エラーについて**:
   - 画像サイズが 500x500px を超えていても、**エラーや例外（クラッシュ）は発生しません**（※極端に巨大で OutOfMemory になる場合を除く）。
2. **文字盤背景（`clock_background.png`）**:
   - `iv_background` のサイズは `500px` x `500px` に固定されています。
   - `android:scaleType` が未指定のため、Android デフォルトの `FIT_CENTER` で描画されます。
   - そのため、500x500px より大きい（または小さい）画像を指定した場合、アスペクト比を維持したまま **500x500px 枠内に自動的にスケーリング（縮小・拡大）** されます。正方形（1:1）でない場合は余白（レターボックス）が生じます。
3. **各針（`clock_hour.png` / `clock_minute.png` / `clock_second.png`）**:
   - `iv_hour`, `iv_minute`, `iv_seconds` のレイアウト属性は **`layout_width="wrap_content"`, `layout_height="wrap_content"`** です。
   - このため、**画像はリサイズ（縮小・拡大）されず、元画像の解像度そのままの大きさ** で配置されます。
   - もし 500x500px より大きい画像を配置した場合、親の FrameLayout（500x500px）の中央にそのまま配置されるため、枠外にはみ出して描画が切れる（クリッピングされる）可能性があります。
   - 逆に、針の部分だけを小さく切り抜いた画像（例: 40x200px 等）にした場合、後述の回転軸（Pivot）が狂う決定的な問題が発生します。

---

### ② 回転軸（Pivot）：針の中心座標の計算方法

#### ソースコードの該当箇所
- **`BasePictureClockView.java`**:
  ```java
  @Override
  protected void onDraw(Canvas canvas) {
      super.onDraw(canvas);
      setTime();
      this.iv_hour.setRotation(this.houDegree);
      this.iv_minute.setRotation(this.miDegree);
      this.iv_seconds.setRotation(this.secDegree);
      postInvalidateDelayed(1000L);
  }
  ```

#### 挙動の分析
1. **Pivot の設定・計算ロジック**:
   - ソースコード上で `setPivotX()` や `setPivotY()` は **一切呼び出されていません**。
   - また、回転軸の座標を指定する設定ファイル（JSON や XML、プロパティ等）も存在しません。
   - したがって、Android の `View` の標準仕様に従い、**常に各 ImageView の中心座標 `(Width / 2, Height / 2)` が固定の回転軸（Pivot）** となります。
2. **カスタム針を作成する際の必須要件**:
   - システム標準のアナログ時計（`classic_clock_*.png` 等）の針画像は、すべて **「500x500px の透明キャンバスの中央 (250, 250) に回転軸を配置し、12時方向（上向き）に向けて描画」** されています。
   - カスタム時計でも針画像を **500x500px の正方形キャンバスとして作成し、針の根元（回転中心）をキャンバスの中心 (250, 250) に配置する** 必要があります。
   - ※針の輪郭のみをトリミングした画像（例: 30x200px）を配置すると、ImageView の中心 `(15, 100)` を軸にプロペラのように回転してしまい、時計の文字盤中心から外れてしまいます。

---

### ③ 読み込まれるファイル名

#### ソースコードの該当箇所
- **`CustomizeClockView.java`**:
  ```java
  StringBuilder sb = new StringBuilder();
  sb.append(Environment.getExternalStorageDirectory().getPath());
  String str = File.separator;
  sb.append(str);
  sb.append("customize_clock");
  sb.append(str);
  String string = sb.toString();

  String str2 = string + "clock_background.png";
  if (new File(str2).exists() && (bitmapDecodeFile4 = BitmapFactory.decodeFile(str2)) != null) {
      this.iv_background.setImageBitmap(bitmapDecodeFile4);
  }
  String str3 = string + "clock_hour.png";
  if (new File(str3).exists() && (bitmapDecodeFile3 = BitmapFactory.decodeFile(str3)) != null) {
      this.iv_hour.setImageBitmap(bitmapDecodeFile3);
  }
  String str4 = string + "clock_minute.png";
  if (new File(str4).exists() && (bitmapDecodeFile2 = BitmapFactory.decodeFile(str4)) != null) {
      this.iv_minute.setImageBitmap(bitmapDecodeFile2);
  }
  String str5 = string + "clock_second.png";
  if (!new File(str5).exists() || (bitmapDecodeFile = BitmapFactory.decodeFile(str5)) == null) {
      return;
  }
  this.iv_seconds.setImageBitmap(bitmapDecodeFile);
  ```

#### 挙動の分析
1. **保存先パス**:
   - `Environment.getExternalStorageDirectory().getPath() + "/customize_clock/"`
   - 実機パス: `/sdcard/customize_clock/` または `/storage/emulated/0/customize_clock/`
2. **読み込まれるファイル名一覧**:
   - `clock_background.png` （文字盤背景）
   - `clock_hour.png` （時針）
   - `clock_minute.png` （分針）
   - `clock_second.png` （秒針）
   ※注意: システム標準の `classic_clock` などの秒針アセット名は `classic_clock_seconds.png`（複数形）ですが、本カスタム時計では **`clock_second.png`（単数形）** が指定されています。
3. **その他のファイル（設定ファイル、json、フォント等）の有無**:
   - **一切存在しません**。
   - テキストフォントや座標設定用の JSON、XML、INI ファイル等を読み込むロジックは存在せず、上記 4 点の PNG 画像ファイルのみを直接 `BitmapFactory.decodeFile()` で読み込みます。
4. **ファイル省略時の挙動**:
   - 各ファイルごとに `new File(...).exists()` のチェックが行われているため、存在しない画像は単にスキップされ、該当の View には何も表示されません。
   - 特に `clock_second.png` が存在しない場合は秒針が非表示になるため、2針時計（時針・分針のみ）としても動作します。

---

### ④ 描画・更新ロジック

#### ソースコードの該当箇所
- **`BasePictureClockView.java`**:
  ```java
  @Override
  protected void onMeasure(int i, int i2) {
      super.onMeasure(i, i2);
      setMeasuredDimension(
          View.MeasureSpec.getMode(i) == Integer.MIN_VALUE ? 500 : View.MeasureSpec.getSize(i),
          View.MeasureSpec.getMode(i2) != Integer.MIN_VALUE ? View.MeasureSpec.getSize(i2) : 500
      );
      postInvalidateDelayed(1000L);
  }

  @Override
  protected void onDraw(Canvas canvas) {
      super.onDraw(canvas);
      setTime();
      this.iv_hour.setRotation(this.houDegree);
      this.iv_minute.setRotation(this.miDegree);
      this.iv_seconds.setRotation(this.secDegree);
      postInvalidateDelayed(1000L);
  }

  private void setTime() {
      String[] strArrSplit = new SimpleDateFormat("HH:mm:ss").format(new Date(System.currentTimeMillis())).split(":");
      int i = Integer.parseInt(strArrSplit[0]);   // 時 (0-23)
      int i2 = Integer.parseInt(strArrSplit[1]);  // 分 (0-59)
      int i3 = Integer.parseInt(strArrSplit[2]);  // 秒 (0-59)
      this.secDegree = i3 * 6;
      this.miDegree = (i2 * 6) + ((i3 * 1.0f) / 10.0f);
      this.houDegree = (i * 30) + ((i2 * 1.0f) / 2.0f);
  }
  ```

#### 挙動の分析
1. **運針方式（ステップ運針 vs スイープ運針）**:
   - **完全な「ステップ運針（1秒刻みでジャンプ）」** です。スムーズなスイープ運針ではありません。
   - `SimpleDateFormat("HH:mm:ss")` で秒単位（整数）までしか取得しておらず、ミリ秒は取得・反映されていません。
   - 秒針の角度計算: `secDegree = i3 * 6`（1秒ごとに 6度 ずつ不連続に変化）。
   - 分針の角度計算: `miDegree = (i2 * 6) + ((i3 * 1.0f) / 10.0f)`（1秒ごとに 0.1度 ずつ変化）。
   - 時針の角度計算: `houDegree = (i * 30) + ((i2 * 1.0f) / 2.0f)`（1分ごとに 0.5度 ずつ変化。秒は影響しません）。
2. **更新頻度・インターバル**:
   - `onDraw()` および `onMeasure()` の末尾で `postInvalidateDelayed(1000L)` が呼ばれています。
   - 描画完了から **1000ms（1秒）ごと** に再描画（invalidate）が要求されます。
   - ミリ秒境界への同期ロジックはなく、前回の描画から約1000ms後に再描画されるループとなっています。

---

## 3. 追加仕様：カスタム時計の有効化とリアルタイム反映（リロード）

`KeyguardClockSwitch.java` の解析により判明した、Android Settings 経由の制御仕様です。

### カスタム時計の切り替え
SystemUI はシステム設定 `lock_clock_type` の値を監視しています。
```java
changeLockClock(Settings.System.getInt(getContext().getContentResolver(), "lock_clock_type", 0));
```
- `lock_clock_type = 7` を設定することで、カスタム時計（`CustomizeClockView`）が有効化されます。
- ADB コマンド例:
  ```bash
  adb shell settings put system lock_clock_type 7
  ```

### 画像変更時の即時リロード（再読み込み）
`KeyguardClockSwitch` では `lock_clock_customize_refresh` の ContentObserver も登録されています。
```java
context.getContentResolver().registerContentObserver(
    Settings.System.getUriFor("lock_clock_customize_refresh"), true, contentObserver
);
```
- `/sdcard/customize_clock/` 配下の PNG 画像を差し替えた際、以下のコマンドで `lock_clock_customize_refresh` の値を変更（インクリメント等）すると、**端末を再起動することなく、即座にカスタム時計の View が再生成され新しい画像が読み込まれます**。
- ADB コマンド例:
  ```bash
  adb shell settings put system lock_clock_customize_refresh $(($(adb shell settings get system lock_clock_customize_refresh 2>/dev/null || echo 0) + 1))
  ```

---

## 4. 推奨デザインガイドラインまとめ

カスタム時計用アセットを作成する際は、以下のガイドラインに準拠してください。

| ファイル | 推奨解像度 | 回転軸 / 描画基準 | 備考 |
| :--- | :--- | :--- | :--- |
| `clock_background.png` | **500 x 500 px** | - | 正方形 PNG（透過可） |
| `clock_hour.png` | **500 x 500 px** | **中心 (250, 250)** | 背景と同じ500x500透過キャンバスの(250,250)を針の根本にし、12時（真上）に向けて配置 |
| `clock_minute.png` | **500 x 500 px** | **中心 (250, 250)** | 同上（500x500透過キャンバス） |
| `clock_second.png` | **500 x 500 px** | **中心 (250, 250)** | 同上（500x500透過キャンバス）。省略時は2針時計となる |
