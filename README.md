# poko split-wireless — ZMK firmware

XIAO nRF52840 Plus × 2 の完全無線分割（左右 BLE）＋右手トラックボール（PMW3610）用の ZMK config。
設計の背景は `wireless-poko-design.md`、実装判断は `zmk-implementation-notes.md` を参照。

## 構成

| | |
|---|---|
| board | `xiao_ble//zmk`（ZMK main / Zephyr 4.1 の HWMv2 バリアント表記） |
| shield | `poko_left`（ペリフェラル）/ `poko_right`（セントラル＋トラボ） |
| マトリクス | 4行6列 × 左右、`col2row`、外部抵抗なし |
| 論理グリッド | 4行12列。右は `col-offset = 6`。`RC(3,6)` はトラボ穴 → 計 **47キー** |
| 電池監視 | 外部分圧 1MΩ/470kΩ → A0 (P0.02/AIN0)。ボード標準 `vbatt` は無効化 |

```
poko-zmk-config/
├── build.yaml                     ビルド対象（board × shield）
├── .github/workflows/build.yml    本家の再利用ワークフローを呼ぶだけ
├── wireless-poko-design.md        設計書（PCB・電源・機構まで含む）
├── zmk-implementation-notes.md    ZMK 実装ノート
└── config/
    ├── west.yml                   ZMK 本体 + PMW3610 ドライバ
    ├── zephyr/module.yml          config/boards を shield 探索対象にする
    ├── poko.keymap              左右共通キーマップ（47キー・6レイヤ）
    ├── poko_left.conf           左（ペリフェラル）
    ├── poko_right.conf          右（セントラル・トラボ）
    └── boards/shields/poko/
        ├── Kconfig.shield
        ├── Kconfig.defconfig      SPLIT / 右=セントラル / キーボード名
        ├── poko.dtsi            kscan・matrix-transform・電池分圧（左右共通）
        ├── poko_left.overlay
        ├── poko_right.overlay   col-offset=6 + SPI + PMW3610 + input-listener
        └── poko.zmk.yml
```

## ビルド

GitHub Actions（`.github/workflows/build.yml`）で自動ビルド。Artifacts の
`poko-split-wireless.zip` に5つの `.uf2` が入る。

| uf2 | 用途 |
|---|---|
| `poko_left` / `poko_right` | 常用。ログ無し（省電力のため） |
| `poko_left-logging` / `poko_right-logging` | ブリングアップ用。USB シリアルにログが出る |
| `settings_reset` | ペアリング初期化。焼いたら必ず本ファームを焼き直す |

### ログの見かた（macOS）

`zmk-usb-logging` スニペット付きのファームは USB CDC-ACM のシリアルポートを生やす。

```sh
brew install tio
ls /dev/cu.usbmodem*
tio /dev/cu.usbmodemXXXXX
```

`tio` はリセットでポートが消えても自動再接続するので、`screen` より扱いやすい
（XIAO はリセットのたびに USB を再列挙するため）。`screen /dev/cu.usbmodemXXXXX 115200`
でも見られる（終了は `Ctrl-A` → `K`）。CDC-ACM は仮想シリアルなのでボーレートは何でもよい。

ローカルで回す場合はこのリポジトリを west workspace のルートにして：

```sh
west init -l config
west update
west zephyr-export

west build -p -s zmk/app -d build/left  -b 'xiao_ble//zmk' -- -DSHIELD=poko_left  -DZMK_CONFIG="$PWD/config"
west build -p -s zmk/app -d build/right -b 'xiao_ble//zmk' -- -DSHIELD=poko_right -DZMK_CONFIG="$PWD/config"
```

`xiao_ble//zmk` の `//` はシェルに食われへんようクォートしとくこと。

## 書き込み

XIAO nRF52840 のリセットボタンを**ダブルタップ** → UF2 ドライブが出る → `.uf2` をドロップ。
自作基板側にブートボタンは不要。ペアリングが壊れたら **左右両方**に `settings_reset` を焼いてから
本ファームを焼き直す（左→右の順で電源を入れると再ペアリングされる）。

## キーマップ

`low-profile-with-trackball` の QMK/Vial 版（`lp_tb/v0_1/keymaps/vial/keymap.c`）からの移植。
BASE / LOWER / UPPER / FN / MOUSE / SCROLL の6レイヤ。

- QMK の `set_auto_mouse_layer(_MOUSE)` → `poko_right.overlay` の `&zip_temp_layer 4 400`
- QMK の `set_scroll_layer(_SCROLL)` → 同 `scroller` ノード（`zip_xy_to_scroll_mapper`）
- QMK の `pointing_device_task_user()` の v/h 反転 → `zip_scroll_transform` の X/Y invert
- FN レイヤに BLE 操作を追加（無線化に伴う新規）。プロファイル選択 `&bt BT_SEL 0-4` は **Q W E R T**、
  `&bt BT_CLR` は誤爆防止で BSPC の位置。`&out OUT_TOG` / `&bootloader` / `&sys_reset` は右半分。

## 進捗

素の XIAO nRF52840 × 2（基板・センサー・電池なし）で以下まで確認済み。詳細は
`zmk-implementation-notes.md` §16。

- [x] CI でビルド成功（左・右・`settings_reset` ＋ ログ版）
- [x] `&spi0` が使える（SPIM0/TWIM0 の衝突なし）
- [x] PMW3610 ドライバ実体化（`pixart,pmw3610-alt` / `CONFIG_PMW3610_ALT` で正解。非同期 init なのでセンサー無しでも止まらない）
- [x] 外部分圧の ADC 初期化成功（`bvd_init: AIN0 setup returned 0`）
- [x] kscan が §3 のピン割当どおりに設定される
- [x] **左右の BLE 接続成立**（`split_svc_pos_state_ccc: value 1` / `security_changed level 2`）
- [x] ホスト（macOS）の Bluetooth に `poko` が出る

## 実機で潰す項目

- [ ] ダイオードの物理向きが `col2row`（ROW=プルダウン, COL=ACTIVE_HIGH）と一致するか → メイン基板実装後
- [ ] キー入力・NKRO・ゴースト（47キー）
- [ ] トラボの向き（`swap-xy` / `invert-x` / `invert-y`）と CPI → センサー実装後
- [ ] MOTION の外部プルアップ要否
- [ ] NiMH の残量表示（notes §16-6 の ×3 リスケール案。実測 mV を見てから）
- [ ] 消費電流の実測 → スリープ・BLE 接続間隔の詰め
- [ ] 技適の決着（電波を出す運用の前。USB 有線でのキー検証は技適前でも可）
