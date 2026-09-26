# DYA Studio レベル2対応メモ（torabo-tsuki-lp）

`main+dya-studio` ブランチで行った作業の記録。ハマったポイントと未解決の TODO をまとめておく。
安定版フォールバックは `v0.3+dya-studio` ブランチ（マクロ/コンボ・5レイヤー・ミニトラックパッド以前、実機確認済み）。

## やったこと概要

1. **DYA Studio レベル2対応**（トラックボール調整・ランタイムマクロ・ランタイムコンボ）
   - `zmk-module-runtime-input-processor` / `zmk-feature-runtime-macro` / `zmk-feature-runtime-combo` などを導入
   - マクロ/コンボは ZMK master（`main+custom-studio-protocol` 系統）専用のため、ZMK を `v0.3+custom-studio-protocol` から **`cormoran/zmk@main+dya`** へ全面移行（Zephyr 4.1）
2. **キーマップを5レイヤー化**（空の `layer_4` を追加）
3. **BLE 接続の安定化**（`CONFIG_ZMK_STUDIO_LOCKING` を巡る試行錯誤の末、元の `n` に確定）
4. **ミニトラックパッド対応**（`sekigon-gonnoc/torabo-tsuki-lp` の mini-trackpad-option）
   - 一旦「常時スワイプでブラウザタブ切替」ジェスチャーを試すも、通常のトラックパッド機能（1本指カーソル/クリック、2本指スクロール/右クリック）の方が優れていたため撤回
   - トラックボールとトラックパッドの runtime input processor を**独立インスタンス化**（速度などを別々に調整可能に）

## ハマったポイント（時系列）

### 1. CI コンテナが Zephyr 4.1 を扱えない
`build-user-config.yml@v0.3`（Zephyr 3.5 用イメージ）で ZMK main+dya をビルドしようとすると
`soc/Kconfig.defconfig not found` で configure 段階から失敗。**`@main` に変更**して解決。

### 2. `bmp_boost` v0.2 は hwmv1、Zephyr 4.1 は hwmv1 非対応
`zmk-component-bmp-boost` を **`v0.2` → `master`**（ZMK v0.4 / hardware-model-v2 移行版）に変更。
併せて `zmk-feature-cdc-acm-bootloader-trigger` も `v0.2` → `main`。

### 3. `CONFIG_ZMK_BOARD_COMPAT` は `.conf` に書けない
`build-user-config.yml@main` の新設チェック（board が ZMK 対応済みか）に `bmp_boost@master` が
引っかかる。`ZMK_BOARD_COMPAT` は promptless シンボルで `.conf` から代入できず
（"assigned in a configuration file, but is not directly user-configurable"）、
リポジトリ直下の `Kconfig`（`config ZMK_BOARD_COMPAT / default y if BOARD_BMP_BOOST`）＋
`zephyr/module.yml` の `build.kconfig` で解決。

### 4. `src/board.c` が Zephyr 4.1 の `INPUT_CALLBACK_DEFINE` API と不一致
3引数（`dev, callback, user_data`）に変更されていたため、コールバック関数も
`(struct input_event *evt, void *user_data)` に修正。

### 5. `zmk-feature-fast-keymap` は `main+dya` と非互換
`main+custom-studio-protocol+fast-keymap` という専用ブランチ前提のモジュールで、
`main+dya` では `proto/zmk/custom.pb.h` 等のヘッダが見つからずビルド失敗。**導入を諦めた**（マクロ/コンボには不要）。

### 6. ランタイムマクロ/コンボは central 限定にする必要がある
`CONFIG_ZMK_RUNTIME_MACRO`/`RUNTIME_COMBO` を左右共通 conf に置くと、
peripheral ビルドでも `runtime_combo.c` がコンパイルされ、peripheral には存在しない
keymap/keycode API を参照してリンクエラー（`zmk_keymap_highest_layer_active` など）。
`snippets/split-central/split-central.conf`（central 限定）に移動して解決。

### 7. `&rmacro`（ランタイムマクロのビヘイビア）が DYA Studio の選択肢に出ない
`zmk-feature-runtime-macro` はビヘイビアノードを自動登録しない。
`#include <behaviors/runtime_macro.dtsi>` を central 限定 snippet
（`snippets/split-central/split-central.overlay`）に追加して解決。

### 8. ZMK バージョンジャンプでフラッシュ設定が全消去
v0.3 → main+dya のような**メジャーバージョンジャンプ**では設定ストレージ形式が変わり、
Web で保存していたキーマップ/トラックボール設定が消える（通常のファーム更新では起きない）。
DYA Studio のエクスポート（Keyboard Abyss JSON）から `config/keymap.keymap` を復元。
このとき **エクスポートは M Layout の表示順**で、L Layout の canonical 順とは異なるため
`position_map_m_1` で変換する必要があった（1回目は変換ミスで「1段ズレ」た）。

### 9. `CONFIG_ZMK_STUDIO_LOCKING=y` にすると BLE 出力が死ぬ
DYA Studio を BLE 接続するには `&studio_unlock` 経由の directed advertising が必要
（`CONFIG_ZMK_STUDIO_LOCK_BLE_DIRECT_ADVERTISING_ON_UNLOCK`、`LOCKING=y` の時だけ有効化）。
しかしこれを有効にすると、Studio 接続中にキーボードが**接続済みのまま connectable advertising
を開始**してしまい、2本目の BLE 接続が張られることで HID（キー入力・クリック）が詰まる不具合を確認。
**`LOCKING=n` に戻して確定**。DYA Studio の BLE 接続は今のところ利用しない方針（USB で編集）。

### 10. BLE の古いボンドが残っていると再ペアリングできない
プロファイルがボンド済みの相手に対して directed advertising するため、
Mac 側でペアリング解除しても、キーボード側のプロファイルをクリア/切替しないと新しい端末から見えない。
DYA Studio の「接続」タブでプロファイルの「ペアリング解除」→ 空きプロファイルへ「切り替え」で解決。

### 11. E/D/C キーが反応しない → 犯人はハードウェア
同じマトリクス列（QWERTY で中指のキー）がまとめて反応しなくなった。キーマップは Studio 上で
正しく表示されており、ファーム側の要因は一通り除外。最終的に **BMP Boost 基板が浮いていた**
（未接触）のが原因で、押し込んだら解消。ミニトラックパッド取り付け時の分解が影響したと思われる。

### 12. ミニトラックパッド追加でトラックボールのクリック反応が悪化
リレーされたトラックパッドの入力イベントが split 通信路を圧迫し、クリックキー入力が遅延する
症状。**根本原因は未特定**（トラフィック量自体は変えていないので、再発の可能性は残る）。

### 13. スワイプジェスチャー実験（`zmk-input-processor-keybind`, te9no 製・開発中モジュール）
左右スワイプ→ Ctrl+Tab/Ctrl+Shift+Tab、上下スワイプ→タブ閉じる/新規タブ、を実装したが、
「常時スワイプ」だとトラックパッド本来の機能（カーソル・クリック・2本指スクロール）を
犠牲にしてしまうため撤回。**`snippets/trackpad-gesture/` として未使用のままリポジトリに残置**
（再検討する場合の土台として）。

### 14. `zmk-driver-iqs7211e` は 1本指クリック・2本指スクロール/右クリックを標準実装済み
ドライバのソースで確認済み（ZMK 側の追加設定は不要）。ただし **2本指の分離検出は
IQS7211E チップ自身のオンチップ判定**（`finger_count = info_flags[1] & 0x03`）に依存し、
ミニトラックパッド用に ATI（自動調整）パラメータが最適化されていない可能性が高く不安定。

### 15. ダブルタップ→ドラッグが固まるドライバのバグ
素早く2回タップすると「押しっぱなしドラッグ」モードに入るが、指のリフトオフ検出に失敗すると
**クリック押下状態が永久固定**される（Kconfig にタイムアウトや無効化オプションなし）。
回避策：しっかり大きく・長めにドラッグしてから指を離す（`touch_duration ≥ 200ms` かつ
`移動量 ≥ 50` でリリース条件を満たしやすくなる）。根本修正には上流ドライバへのパッチが必要。

### 16. `#include <dt-bindings/zmk/input.h>` は存在しないヘッダ
`INPUT_EV_REL`/`INPUT_REL_X`/`INPUT_REL_Y` 等は実際には `<dt-bindings/zmk/keys.h>`
（`<input/processors/runtime-input-processor.dtsi>` が transitively include）で定義されている。
存在しないヘッダの `#include` はプリプロセッサの致命的エラーとなり、そのファイルを使う
ビルドターゲットだけが全滅する（気づきにくい）。

### 17. `processor-label` は最大8文字（BLE 制約）
`zmk,input-processor-runtime` ノードの `processor-label` は8文字を超えると使えない。
"Left Track Pad" は不可、"LeftPad"（7文字）で対応。

## 現在の構成（`main+dya-studio` HEAD）

- ZMK: `cormoran/zmk@main+dya`、Zephyr: 付属の `zmkfirmware/zephyr@v4.1.0+zmk-fixes`
- `bmp_boost@master`（hwmv2）、`zmk-feature-cdc-acm-bootloader-trigger@main`
- DYA Studio Level2: トラックボール調整・ランタイムマクロ・ランタイムコンボ・BLE管理・設定RPC・電池履歴
- `CONFIG_ZMK_STUDIO_LOCKING=n`（常時アンロック、BLE 編集は使わない）
- レイヤー数: 5（`layer_0`〜`layer_4`、`layer_4` は空）
- ミニトラックパッド: 通常トラックパッドモード（`input-split-listener` snippet、独立
  processor インスタンス `relay_runtime_input_processor` / ラベル "LeftPad"、速度デフォルト半速）
- 未使用のまま残置: `snippets/trackpad-gesture/`、`zmk-input-processor-keybind`（west.yml）

## TODO / 未解決事項

- [ ] ミニトラックパッドのダブルタップ→ドラッグ固着バグの根本修正（`zmk-driver-iqs7211e` への
      パッチ or 上流への Issue/PR。タップ判定タイムアウトの追加、または無効化 Kconfig の新設）
- [ ] ミニトラックパッドの2本指検出の安定化（`mini_trackpad_iqs7211e_init.h` の ATI/感度パラメータ
      をこのセンサー個体向けに再チューニング。実機での試行錯誤が必要）
- [ ] トラックパッド追加時にトラックボールのクリック反応が悪化した件の根本原因調査
      （split 通信路の混雑を実際に軽減する対応：レポートレート制限、バッファ/スタック見直し等）
- [ ] `trackpad-gesture` snippet（スワイプジェスチャー）を今後使うか判断。使うなら
      修飾キー/レイヤーでスクロールと排他制御する設計に作り直す
- [ ] DYA Studio の BLE 接続機能はディレクテッド広告の不具合で使用不可のまま。
      cormoran フォーク側で directed advertising の実装が改善されたら再検討
      （`app/src/ble.c` の `// TODO: Change back to ZMK_ADV_DIR when fixed` コメント参照）
- [ ] `double_ball`（2トラックボール）構成は今回のリレー processor 独立化以降、未再テスト
