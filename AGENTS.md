# daxs ZMK設定開発ガイド

daxs のファームウェア。日々のキーマップ変更は [daxial](https://wadackel.github.io/daxial/) で行い、この repo は「daxial が動く土台」と「daxial では変えられないもの」を持つ。構成は [zmk-config-dax6](https://github.com/wadackel/zmk-config-dax6) を移植したもので、判断の経緯は dax6 側の `docs/adr/` にある。

## ビルド手順

ビルドコマンドは `nix develop --command` 経由で実行する。対話型シェルに入る必要はない。

```bash
nix develop --command just setup                  # 初回のみ。west ワークスペースの初期化
nix develop --command just build                  # 全ターゲット
nix develop --command just build-target daxs_R    # daxs_R / daxs_L / settings_reset のどれか
nix develop --command just clean                  # build/ を消す
nix develop --command just pristine               # west ワークスペースごと消す
```

成果物は `build/` に出る:

- `build/daxs_R.uf2` - 右手側（central、トラックボール、ZMK Studio）
- `build/daxs_L.uf2` - 左手側（peripheral、エンコーダー）
- `build/settings_reset.uf2` - 設定リセット用

CI は無い。ビルドは手元でだけ行う。

- `just pristine` は `zephyr/` を丸ごと消すので、git 管理下の `zephyr/module.yml` も消える。実行後は `git checkout zephyr/module.yml` で戻す
- `nix flake update` はしない。新しい nixpkgs は `python310` を持たず、zephyr-nix の評価が darwin で失敗する。Zephyr SDK は 0.17.0 固定（Zephyr 4.1 が想定する版）

## ディレクトリ構造

- `boards/shields/daxs/` - Shield 定義（dtsi、overlay、conf、Kconfig）
- `config/` - キーマップ（`daxs.keymap`）、ジェスチャーの既定値（`gestures.dtsi`）、`west.yml`
- `module/` - 空の CMakeLists.txt / Kconfig。west が Zephyr を `./zephyr` に取得するため、明示しないとこの repo の Zephyr モジュール定義（`zephyr/module.yml`）が Zephyr 本体の `zephyr/CMakeLists.txt` を拾ってしまう
- `.gitignore` の `zephyr/*` と `!zephyr/module.yml` は組で必要
- dax 系で共有する自作モジュール（マウスジェスチャー、OS レイヤー、トラックボールの CPI 設定）は [`wadackel/zmk-dax-modules`](https://github.com/wadackel/zmk-dax-modules) にあり、`config/west.yml` から SHA で参照する

## ハードウェア仕様

- 左右分割の 39 キー（L = 20、R = 19）。両側とも Seeed XIAO nRF52840（ボード名 `xiao_ble//zmk`）
- `daxs_R`（右手）が **central**。キーマップ、レイヤー、トラックボール、Studio を担当する
- `daxs_L`（左手）が **peripheral**。keymap に関わる設定は `daxs_R.conf` にだけ書く
- ダイオードは col2row。row は `GPIO_ACTIVE_HIGH | GPIO_PULL_DOWN`、col は `GPIO_ACTIVE_HIGH`
- 設計データは `~/develop/keyboard/daxs`（KiCad）。下の表は PCB の XIAO パッドの net から起こした

### ピン割り当て

| 信号 | ピン | devicetree |
|---|---|---|
| Row0 / Row1 / Row2 / Row3（左右共通） | D1 / D2 / D3 / D6 | `&xiao_d 1` `2` `3` `6` |
| Col0 / Col1 / Col2 / Col3（左右共通） | D10 / D9 / D8 / D7 | `&xiao_d 10` `9` `8` `7` |
| Col4（左右共通） | NFC2 = P0.10 | `&gpio0 10` |
| Col5（L のみ） | NFC1 = P0.09 | `&gpio0 9` |
| エンコーダー A / B（L） | D5 = P0.05 / D0 = P0.02 | `&xiao_d 5` / `&xiao_d 0` |
| トラックボール SCLK / SDIO（R） | P0.05 / P0.04 | `spi0`。MOSI と MISO はどちらも P0.04（3 線式） |
| トラックボール CS / MOTION（R） | P0.09 / P0.02 | `&gpio0 9` / `&gpio0 2` |

- **XIAO の既定のバスは全部無効にしている**（`daxs.dtsi`）。`xiao_serial`（D6 / D7）、`xiao_spi`（D8〜D10）、`xiao_i2c`（D4 / D5）は、どれもマトリクス、エンコーダー、トラックボールのピンと重なる
- **`&uicr { nfct-pins-as-gpios; };` は必須。** P0.09 / P0.10 は NFC ピンで、これが無いと Col4、Col5、トラックボールの CS が動かない
- L の Col0 は外側（小指側）、R の Col0 は内側（人差し指側）

### キー位置

```
 0  1  2  3  4              5  6  7  8  9
10 11 12 13 14         15  16 17 18 19 20
21 22 23 24 25  26     27  28 29 30 31 32
      33 34 35 36      37 38
```

- 15 と 27 は R の内側列（縦 2 個）、26 は L の内側キー、33〜36 は L の親指、37〜38 は R の親指
- **R の内側 2 キーは電気的には Row3 の Col0 / Col1。** 物理位置に合わせて、matrix transform では Row1 と Row2 の行に置いている。R の親指は Row3 の Col2 / Col3
- transform は 4×11 で、R は `col-offset = <6>`
- physical layout（`daxs.dtsi`）は PCB のスイッチ中心座標から作った（17 mm = 100）。daxial はこれだけを見てキーを描く。左右の間隔は任意の値

### ロータリーエンコーダー

L に Alps EC12E が 1 個。`steps = <80>`、`triggers-per-rotation = <40>` は dax6 の値をそのまま置いたもので、daxs の実機では未調整。

- `triggers-per-rotation` は `steps` の整数約数から選ぶ。非整数比だと、休止からの最初の 1 デテントが閾値に届かず空振りする
- スクロールの量は `ZMK_POINTING_DEFAULT_SCRL_VAL`（`daxs.keymap`、既定 120）で調整する。macOS は 60 だと小さすぎて反応しない
- どちらも daxial からは変えられない

### PAW3222 トラックボール

R に SPI0 + MOTION 割り込みで接続する。ドライバーは生の `INPUT_REL_X` / `INPUT_REL_Y` を出すだけで、向きや倍率は input-processor のチェーンで決める（`daxs_R.overlay` の `trackball_listener`）。

- **base chain**: `<&tb_pointer>, <&dax_gestures>`
  - `tb_pointer` が向きの補正と倍率を担う（`trackball_processors.dtsi`）。既定の `xy-swap-enabled; x-invert;` は dax6 の値で、daxs のセンサーの向きに合うかは実機で未確認。daxial の Trackball 画面で直せる
  - `dax_gestures` がマウスジェスチャーを判定する（`config/gestures.dtsi`）
- **scroll_override**（Scroll レイヤー = 6、`process-next` 付き）: `<&tb_pointer>, <&zip_xy_to_scroll_mapper>, <&tb_scroll>`
  - `tb_pointer` はここでも先頭にいるので、その倍率はスクロールにも掛かる
- **android_override**（Android レイヤー = 1、`process-next` 付き、**必ず最後に宣言する**）: WHEEL を反転するだけ
  - override は宣言順に評価される。Android レイヤーは常時アクティブなので、最後に置かないと Scroll を飲み込む
- CPI は `res-cpi`（既定 1400）。範囲は 608〜4826

エンコーダーのスクロールは `trackball_listener` ではなく `&msc_input_listener`（`daxs.keymap` の `android_encoder_override`）を通る。

## レイヤー構成

| 番号 | 名前 | 説明 |
|---|---|---|
| 0 | default_layer | デフォルト |
| 1 | Android | BLE プロファイル 3 を選んでいる間、常時アクティブになる上書きレイヤー |
| 2 | Symbol | 記号 |
| 3 | Num | 数字 |
| 4 | Function | ファンクション、矢印、メディア |
| 5 | Mouse | マウスボタン |
| 6 | Scroll | トラックボールがスクロールになる。K+L の combo で入る |
| 7 | Device | BLE 設定、bootloader |
| 8 | MacGesture | macOS 用ジェスチャーの目印（Mouse レイヤーから保持） |
| 9 | Gesture | 通常のジェスチャーの目印（Mouse レイヤーから保持） |

**daxial はレイヤーの追加、削除、並べ替えができない。** 枚数と順序は `daxs.keymap` で決まる。レイヤー番号は次の場所に直書きされているので、並べ替えるときは全部直す:

- `daxs_R.overlay` の `layers = <6>`（Scroll）と `layers = <1>`（Android）
- `daxs_R.conf` の `CONFIG_ZMK_OS_LAYER_PROFILE_3=1`
- `daxs.keymap` 冒頭の `#define`、`&mo 7` / `&tog 7` / `&lt 5` などの引数、combo の `layers`

Android レイヤーまわりで壊れやすい点:

- **combo の `layers` は `<0 1>` と書く。** combo のレイヤー絞り込みは最上位のアクティブレイヤー 1 つしか見ない
- **Android レイヤーは index 1 に置く。** 上に置くと最上位が常に Android になり、combo のレイヤー限定が効かなくなる
- **レイヤーの出入りは `&tog` / `&mo` で行い、`&to` は使わない。** 常時アクティブな上書きレイヤーを落とす恐れがある（dax6 でも実機未確認）

dax6 の keymap から外側の列を落として移植したので、`LCTRL` は初期状態でどこにも割り当てていない。

## 実行時設定（daxial）

`daxs_R` を USB でつなぎ、Chromium 系ブラウザで daxial を開く。`CONFIG_ZMK_STUDIO_LOCKING=n` なので unlock 操作は要らない。左右とも USB でつなぐとポートが 2 つ出るが、応答するのは右手側だけ。

| 画面 | 仕組み（custom subsystem の識別子） |
|---|---|
| Keymap（キー割当、レイヤー名） | ZMK Studio の RPC |
| Encoders | `zmk-behavior-runtime-sensor-rotate`（`cormoran_rsr`） |
| Trackball | `zmk-module-runtime-input-processor`（`cormoran_rip`）、CPI は `zmk-dax-modules`（`dax_trackball`） |
| Gestures | `zmk-dax-modules`（`dax_gesture`） |
| Combos | `zmk-feature-runtime-combo`（`cormoran__runtime_combo`） |
| Macros | `zmk-feature-runtime-macro`（`cormoran__runtime_macro`） |

daxial から変えられず、この repo に書くもの:

- レイヤーの枚数と順序
- behavior の定義そのもの（`esc_lang2_with_layer` のような独自の hold-tap、mod-morph、tap-dance など）。割り当てられるのはファームウェアが持っている behavior だけ
- hold-tap のタイミング（`tapping-term-ms`、`flavor`、`quick-tap-ms`）、エンコーダーの `triggers-per-rotation`
- input-listener のチェーンの構造、ジェスチャーの組（プロファイル）の個数と名前
- BLE、スリープ、OS レイヤーなどの Kconfig

### 落とし穴

- **keymap / overlay に書いた値は既定値で、アプリで保存した値が優先される。** 既定値を書き換えてフラッシュしても、保存済みの項目には反映されない。戻すにはアプリの reset か `settings_reset.uf2`（BLE のボンドも消える）
- **`rsr_*` の `tap-ms` は 16 以上にする**（今は全部 20）。`&msc` の内部周期は 16 ms で、それより短いとスクロールが一切出ない。ビルドエラーにはならない。アプリ側で tap を 16 未満にした場合も同じ
- **`sensor-bindings` を rsr 以外の behavior にしたレイヤーは、Encoders 画面から編集できない**
- **エンコーダーの割当は書いた瞬間に flash へ保存される。** 試してから取り消す操作は無い
- **既定値のマクロを増やすなら `CONFIG_ZMK_STUDIO_RPC_TX_BUF_SIZE` も広げる。** シリアルポートを開いているプログラムが無い間、起動時の Studio 通知は送信バッファ（256 バイト）に溜まり、満杯になるとキー入力ごとキーボードが止まる。既定値のマクロ 1 個で通知 2 件（`esc_lang2` で計 108 バイト）
- **マクロの既定値のスロット番号はビルド時に決まらない。** 既定値が `esc_lang2` だけなら settings reset の直後はスロット 0 に入り、`&esc_lang2_with_layer 4 0` の `0` と合う。既定値を足したあとは Macros 画面で番号を確かめる。ずれると ESC / LANG2 のキーが効かない
- **保存したことのない既定値のマクロは、アプリの Discard で devicetree の内容に戻る。** flash に記録が無いので、`zmk-dax-modules` の `CONFIG_ZMK_DAX_MACRO_DEFAULTS`（既定で有効、`0379076` 以降）が書き戻す。それより古い revision に戻すと編集が残る
- combo は `runtime_combo_defaults` に、マクロは `cormoran,runtime-macro-default` に書く。`zmk,combos` と `zmk,behavior-macro` のノードは置かない
- ジェスチャー 1 つにつき behavior は 1 つ。複数の操作はマクロにして割り当てる

## ZMK 本体とモジュール

`config/west.yml` の `zmk` は upstream ではなく `cormoran/zmk` の `main+custom-studio-protocol` を SHA で固定している（ZMK main / Zephyr 4.1 に custom Studio RPC を足したもの）。revision は全モジュールとも dax6 と揃えてあり、daxial が vendoring している proto とも一致する。上げるときは dax6 と daxial も合わせて確かめる。

`zmk-rgbled-widget` の `rgbled_adapter` shield で XIAO 本体の LED に状態を出している（`justfile` の `SHIELD`）。

## 実機で未確認のこと

この repo はビルドと生成された devicetree の確認までで、実機には書き込んでいない。

- 全キーが上の位置番号どおりに反応すること（特に R の内側 2 キーと親指）
- エンコーダーの `steps` / `triggers-per-rotation` が EC12E に合うこと
- トラックボールの向きと CPI
- daxial の各画面が出ること（daxial の repo の `pnpm identify` と `pnpm test:device` で確かめられる。proto の一致は `pnpm check:proto <この repo>`）
