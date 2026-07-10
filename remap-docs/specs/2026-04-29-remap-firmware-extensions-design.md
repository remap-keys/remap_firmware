# Remap Firmware Extensions Design

## 文書情報

- **初版日**: 2026-04-29
- **改訂日**: 2026-05-02
- **ステータス**: 設計フェーズ完了 → 実装計画策定前
- **対象**: Remap Firmware（本リポジトリ）
- **ブランチ**: `remap-develop`（QMK 0.32.12 ベース）
- **関連 Spec doc**: `2026-04-29-remap-webapp-integration-design.md`（ウェブアプリ側）

---

## 1. Goals & Scope

### 1.1 Goal 1: VIA JSON 登録ワークフローの撤廃

従来、Remap web app に新キーボードを登録するには JSON 形式の VIA 互換定義ファイルを作成・登録する必要があった。本仕様により、ファームウェア自体が**全メタデータ（マトリックス、KLE レイアウト、ベンダー情報、レイアウトオプション、エンコーダ情報、カスタムキーコード等）を動的に応答**する仕組みを実現し、VIA JSON の事前登録を不要化する。

### 1.2 Goal 2: C-only カスタマイズ機能の動的設定化

QMK の以下機能を、ウェブから動的に設定変更可能にする：

- Tap-Hold timing（TAPPING_TERM 等）
- Combo
- Tap Dance
- Auto Shift
- Caps Word
- One Shot
- Key Override
- Tri Layer
- Repeat Key
- Mouse Keys
- Per-Key Tapping Term

これらは従来、`config.h` / `keymap.c` 等で C コード記述 + 再ビルドが必要だったが、Remap web app から GUI で設定変更 → EEPROM に永続化 → 即時反映できるようにする。

### 1.3 Goal 3: VIA Protocol からの完全分離

本仕様では Remap Firmware が VIA protocol への依存を完全に断ち切る：

- Remap Firmware は **Remap protocol のみ**を喋り、`raw_hid_receive` も Remap dispatcher 一段のみ
- VIA `via.c` 等の実装はリンクしない
- VIA eeconfig (32B) は廃止し、Remap 独自 header に統合
- 既存 dynamic_keymap (VIA 互換) も廃止し、Remap Keymap module で再実装

採択理由：
1. VIA protocol は歴史的経緯で「最適解ではない」点が多い（API 二重化、channel × value_id の二段階 lookup、eeconfig 領域固定）
2. Remap は QMK fork として `raw_hid_receive` の制御権を完全に握れる立場にあり、protocol layer を Remap 専用に最適化可能
3. VIA 互換性を保つコストを払う必要がない（Remap Webapp 側で VIA protocol を別途継続サポートするため、エンドユーザー体験は保たれる）

#### Webapp 側との棲み分け

- **Remap Firmware 側**：Remap protocol のみ
- **Remap Webapp 側**：VIA protocol（既存 1100+ VIA-only キーボード対応継続）+ Remap protocol（本仕様対応キーボード）の両対応
- 接続時 probe で判別 → 該当 client を起動

### 1.4 Out of Scope

本仕様の対象外（Tier 3 = C コード直書きで利用）：

- Tap Dance / Combo の **callback 関数ポインタ機能**（C 関数ポインタは EEPROM 保存不可）
- **Auto Shift Retro モード**（複雑度過剰）
- **Mouse Keys ACCELERATED モード**（QMK 公式推奨外）
- **16-key 以上の Combo**（slot 8B 制約、将来 "Combo Plus" として keycode 帯予約のみ）
- **3-tap 以上の Tap Dance**（slot 8B 制約、将来 "Tap Dance Plus" として予約のみ）
- **Override の per-layer mask**（全レイヤー固定で割り切り）
- **VIA protocol との互換性**（Remap firmware と VIA クライアントは接続不可、Webapp 側で VIA をサポート）

---

## 2. Core Design Philosophy

### 2.1 Remap as QMK Fork

Remap Firmware は QMK Firmware の **fork**（`upstream/master` をベースに `remap-develop` ブランチで派生）。これにより QMK の core 部分（`quantum/quantum.c` 等）を**直接改造可能**であり、Remap モジュールを QMK の標準 dispatch パイプラインに組み込める。

**この設計の利点**：
- キーボード製作者・keymap 製作者が**追加コードを書かずに** Remap 機能が自動有効化
- 既存 QMK 機能の挙動を最大限尊重しつつ、動的設定層を上乗せ
- `raw_hid_receive` の制御権を完全に握れるため、VIA protocol を介さず Remap 独自 protocol で運用可能

**コスト**：
- QMK upstream への追従義務（`core_patches/` 台帳で patch 管理）
- merge 時のコンフリクト解決

### 2.2 Remap = QMK 機能の動的設定 Facade

Remap モジュールは QMK 機能を**置き換える独自実装ではなく**、QMK の既存機能を**動的に設定可能にする facade（アダプター層）**である。

| 構成要素 | 担当 |
|----------|------|
| Tap Dance state machine（タイマー・tap カウント・状態遷移）| **QMK 既存実装そのまま** |
| `tap_dance_actions[]` 配列（動作定義）| **Remap が動的構築（RAM 上）** |
| TAPPING_TERM、permissive_hold 等の挙動制御 | **QMK の既存仕組み流用** |
| 設定 UI（ウェブから動的変更）| **Remap 独自** |
| EEPROM 永続化 | **Remap 独自** |

この思想により：
- 独自 state machine 開発・テストコスト不要
- QMK upstream の改善・バグ修正が自動的に Remap に反映
- 動作の一貫性（Remap 機能と QMK 標準機能で挙動差異なし）

### 2.3 「Lite」の意味

Remap モジュール群（Tap Dance Lite、Combo Lite、Key Override Lite 等）が「Lite」と銘打たれる理由は、**実装手抜きではなく動的設定の物理制約**：

1. **callback 関数ポインタは EEPROM 保存不可** → QMK の `ACTION_TAP_DANCE_FN` 系統は使えない
2. **動的サイズの制約** → `combo_t` の n-key trigger（1-16）のうち、固定 8B slot で表現できるのは 2-key まで
3. **QMK 内部の `static const` 配列を一部変更必要** → core patch でファーム fork を要する

「Lite」=「**動的設定可能な範囲のみ提供**」と定義される。実装ロジックそのものは QMK 機能を 100% 活用する。

### 2.4 Tier 1/2/3 機能分類

| Tier | 内容 | 例 |
|------|------|------|
| **Tier 1** | 基幹機能（Keymap、Macro、Lighting/Audio 等の VIA 相当再設計分） | Keymap、Macro、Backlight、RGB Light/Matrix |
| **Tier 2** | 本仕様で新規追加するモジュール群（Goal 2 中核） | Tap Dance Lite、Combo Lite、他 |
| **Tier 3** | 動的設定不可能な高度機能。C コード直書きで利用 | Tap Dance callback、Auto Shift Retro |

ユーザーは Tier 3 機能を使いたい場合、QMK の従来手段（`config.h` 編集 + 再ビルド）で対応する。Remap fork はこれを妨げない。

---

## 3. Architecture Overview

### 3.1 Module Registry（PROGMEM 配列方式）

各 Remap モジュールは **module_id とハンドラ関数を中央配列に登録**する。

```c
// quantum/remap/remap.h

typedef uint8_t (*remap_handler_fn)(
    uint8_t        sub_cmd,
    const uint8_t *req_payload,
    uint8_t        req_len,
    uint8_t       *resp_payload);

typedef void (*remap_post_init_fn)(void);

typedef struct {
    uint8_t            module_id;
    remap_handler_fn   handler;
    remap_post_init_fn post_init;
} remap_module_t;
```

```c
// quantum/remap/remap_modules.c
const remap_module_t remap_modules[] PROGMEM = {
    { 0x01, remap_meta_handler,         remap_meta_init },
    { 0x02, remap_keymap_handler,       remap_keymap_init },
#ifdef REMAP_MACRO_ENABLE
    { 0x03, remap_macro_handler,        remap_macro_init },
#endif
    { 0x05, remap_system_handler,       remap_system_init },
    { 0x10, remap_slot_table_handler,   remap_slot_table_init },
#ifdef REMAP_CAPS_WORD_ENABLE
    { 0x14, remap_caps_word_handler,    remap_caps_word_init },
#endif
    // ... 他モジュール
#ifdef BACKLIGHT_ENABLE
    { 0x20, remap_backlight_handler,    remap_backlight_init },
#endif
#ifdef RGBLIGHT_ENABLE
    { 0x21, remap_rgblight_handler,     remap_rgblight_init },
#endif
    // ... 他 Lighting/Audio モジュール
};
const uint8_t remap_module_count = sizeof(remap_modules) / sizeof(remap_module_t);
```

**選択理由**：
- AVR では linker section magic（`__attribute__((section("remap_modules")))`）が安定しないため、中央配列 + PROGMEM が確実
- ビルド時にモジュール ON/OFF が `#ifdef` で明確
- 読み手にとって全モジュール一覧が 1 ファイルで把握できる

### 3.2 3-layer × 4-pattern 統合モデル

各モジュールは 3 つの責務層を持つ：

| 層 | 責務 | 実装ファイル例 |
|----|------|----------------|
| **Layer 1: Protocol** | HID raw payload の parse / 応答生成 | `modules/<name>/<name>_handler.c` |
| **Layer 2: Storage** | EEPROM 読み書き、RAM cache 管理 | `modules/<name>/<name>_storage.c` |
| **Layer 3: Runtime** | QMK 機能との連携（実際の挙動変更）| `modules/<name>/<name>_runtime.c` |

Runtime 層の QMK 統合方式は 4 パターンのいずれか：

| パターン | 内容 | 例 |
|---------|------|------|
| **A. weak callback** | QMK の `__attribute__((weak))` 関数を Remap が override | `caps_word_press_user`、`get_tapping_term` |
| **B. runtime API 注入** | QMK 公開 API を呼び出して設定変更 | `set_tri_layer_lower_layer()`、`rgblight_sethsv()` |
| **C. process_record** | キー入力イベントに介入 | TD slot keycode → QK_TAP_DANCE 変換 |
| **D. QMK core patch** | QMK 内部のソースを直接改造 | `process_record_quantum`、`tap_dance_actions[]` 外部参照化 |

### 3.3 EEPROM Region Layout

ATmega32U4（1 KB EEPROM）想定の典型レイアウト（VIA eeconfig 廃止 → Remap 独自 header 統合）：

```
0x000 ─┬─────────────────────┐
       │ Remap Header         │   16 B  (magic / protocol_version /
       │                      │         layout_options / QMK 互換コア設定)
0x010 ─┼─────────────────────┤
       │ Keymap Region        │  560 B  (Keymap module 管理、4 layer × 70 key × 2B 想定)
0x240 ─┼─────────────────────┤
       │ Slot Table           │  192 B  (TD 64 + Combo 64 + OVR 64)
0x300 ─┼─────────────────────┤
       │ Caps Word config     │   4 B   (#ifdef REMAP_CAPS_WORD_ENABLE)
       │ One Shot config      │   4 B   (#ifdef REMAP_ONE_SHOT_ENABLE)
       │ Tri Layer config     │   4 B   (#ifdef REMAP_TRI_LAYER_ENABLE)
       │ Repeat Key flags     │   1 B   (#ifdef REMAP_REPEAT_KEY_ENABLE)
       │ Auto Shift config    │   4 B   (#ifdef REMAP_AUTO_SHIFT_ENABLE)
       │ Mouse Keys config    │   8 B   (#ifdef REMAP_MOUSE_KEYS_ENABLE)
       │ Per-Key Term tbl     │  32 B   (#ifdef REMAP_PER_KEY_TERM_ENABLE)
       │ Combo TERM           │   2 B   (#ifdef REMAP_COMBO_LITE_ENABLE)
       │ Backlight state      │   2 B   (#ifdef BACKLIGHT_ENABLE)
       │ RGB Light state      │   5 B   (#ifdef RGBLIGHT_ENABLE)
       │ RGB Matrix state     │   5 B   (#ifdef RGB_MATRIX_ENABLE)
       │ LED Matrix state     │   3 B   (#ifdef LED_MATRIX_ENABLE)
       │ Audio state          │   2 B   (#ifdef AUDIO_ENABLE)
0x34C ─┼─────────────────────┤
       │ Macro Region         │ ~180 B  (残り全部、remap.json で固定)
0x3FF ─┴─────────────────────┘
```

**消費試算（典型 60% キーボード、全モジュール ON、slot 配分 TD=Combo=OVR=8 既定）**：
```
Remap Header:      16 B
Keymap Region:    560 B
Slot Table:       192 B
Module configs:    59 B  (Caps Word 4 + One Shot 4 + Tri Layer 4 + Repeat 1 +
                          Auto Shift 4 + Mouse Keys 8 + Per-Key Term 32 + Combo TERM 2)
Lighting/Audio:    17 B  (Backlight 2 + RGB Light 5 + RGB Matrix 5 +
                          LED Matrix 3 + Audio 2)
Macro Region:     180 B  (残り)
─────────────────────────
合計:            1024 B  ✅ ピッタリ
```

**注**：slot 配分を `remap.json` で変更すると slot table サイズと Macro 枠が連動して伸縮する。
たとえば slot 合計を論理上限の 32 まで拡張すると slot table は 256 B（+64 B）となり、
Macro 枠は 116 B（−64 B）に縮小される。逆に slot 配分を絞れば Macro 枠を増やせる。

VIA eeconfig (32B) を廃止したことで、旧 spec の典型試算（Macro 枠 161B）から **+19B 拡大**している（Header 縮小 +36B − Lighting/Audio 追加 17B = +19B）。

### 3.4 Remap Header 構造

EEPROM 先頭 16B に配置する Remap header：

```c
typedef struct __attribute__((packed)) {
    uint8_t  magic[4];          // 0x52,0x4D,0x41,0x50 = ASCII "RMAP"
    uint8_t  protocol_version;  // probe と同値、整数連番 1, 2, 3...
    uint8_t  reserved_a;        // alignment
    uint8_t  default_layer;     // QMK 互換（旧 eeconfig 同義）
    uint8_t  keymap_config;     // QMK 互換 bit field（swap caps/escape, autocorrect 等）
    uint32_t layout_options;    // VIA `id_layout_options` 相当を Remap header 内に統合
    uint8_t  unicode_mode;      // QMK unicode 機能用
    uint8_t  reserved_tail[3];  // 将来拡張用
} remap_header_t;  // 16B
```

**フィールド役割**：

| field | 役割 |
|-------|------|
| `magic[4]` | "RMAP" 固定。EEPROM 破損・初回起動の判別 |
| `protocol_version` | 整数連番。version mismatch 検出（§4.2 probe と必ず同値） |
| `default_layer` | QMK 起動時のデフォルト layer（互換維持） |
| `keymap_config` | QMK keymap_config bit field（swap caps/escape, autocorrect 等） |
| `layout_options` | レイアウトオプション 32bit。Metadata module の GET/SET_LAYOUT_OPTIONS_STATE で読み書き |
| `unicode_mode` | QMK unicode 入力モード（互換維持） |
| `reserved_a` / `reserved_tail` | 将来拡張用 |

**捨てられた旧 QMK eeconfig フィールド**：
- 旧 `magic`（`0xFEED 0xFEED`）→ Remap "RMAP" magic に置き換え
- 旧 `backlight_config` → Backlight module 領域へ移動
- 旧 `rgblight_config` → RGB Light module 領域へ移動
- 旧 `audio_config` → Audio module 領域へ移動
- 旧 `keyboard_specific[15]`（VIA layout_options 32-35B 含む）→ Header `layout_options` に統合、それ以外は捨てる
- 旧 `steno_mode`、`handedness` → 必要なら `remap.json` の独自フィールド or 別領域で対応（Open Items 参照）

### 3.5 RAM Cache + Save-on-Demand

EEPROM の書き込み回数制限（10 万回 / cell）に対応するため、Remap モジュールは：

- **RAM 上にキャッシュ**を持ち、SET 時はまず RAM のみ更新（実行時挙動は即時反映）
- ウェブから明示的な `COMMIT_TO_EEPROM` sub_command で初めて EEPROM フラッシュ
- 起動時は EEPROM → RAM へ一括ロード

```c
// 例: slot_table の RAM cache
static custom_slot_t td_slots_ram[REMAP_TD_SLOT_COUNT];

uint8_t remap_slot_table_set(uint8_t type, uint8_t idx, const custom_slot_t *data) {
    // RAM 即時更新
    if (type == TD_TYPE) td_slots_ram[idx] = *data;
    return STATUS_OK;
}

uint8_t remap_slot_table_commit(void) {
    // EEPROM へフラッシュ
    eeprom_update_block(td_slots_ram, (void *)REMAP_OFFSET_TD_SLOTS, sizeof(td_slots_ram));
    return STATUS_OK;
}
```

`eeprom_update_block` は QMK 提供の API で、**変更があったセルのみ書き込む**ため余分な摩耗が発生しない。

このポリシーにより、VIA の `id_custom_save` 相当の明示的 save 命令は不要となる（Save-on-Demand で吸収）。

#### 初回起動・EEPROM 整合性ポリシー

新規ファーム書き込み直後の EEPROM は出荷時 `0xFF` で埋まっているか、旧版データが残存している場合がある。Remap はこの状態を検知し、安全な初期状態にロールバックする。

**検知メカニズム**：
- Remap Header の `magic[4]` = `"RMAP"` を検証
- 同 Header の `protocol_version` を ROM 内 expected 値と比較
- 起動時に EEPROM → RAM ロード前に検証
- 不一致、`0xFF` 連続、または `protocol_version` 不一致の場合、**EEPROM 全領域をゼロ初期化** + Header magic / version 書き込み

**ゼロ初期化後の挙動**：
- Header: `default_layer=0, layout_options=0, ...` → QMK 標準デフォルト
- Keymap Region: 全 0x00 → 起動時に `keymaps[]` PROGMEM 値で初期化（Keymap module の `RESET` 相当）
- Slot Table: 全 type=0x00 → unused slot 扱い（ファーム側 no-op で安全）
- Module configs: 全 0 → QMK 標準デフォルト動作（Remap モジュールは何も override しない）
- Lighting/Audio state: 全 0 → QMK のデフォルト（多くの場合 disabled 状態）
- Macro Region: 全 0x00 → macro 全消去状態

**`protocol_version` 不一致時の扱い**：
- 異なる version で書かれたデータ構造を誤って解釈するリスクを避けるためゼロ初期化
- ウェブ UI は再接続時に「ファーム更新により設定がリセットされました」を表示する責務を持つ

このポリシーにより、初回起動・EEPROM 破損・ファーム version up のいずれも追加ロジックなく安全に処理される。

### 3.6 累積オフセットマクロ（`#ifdef` 駆動）

各モジュールが ON/OFF に応じて占有サイズが変わる。これを `#ifdef` 累積マクロで表現：

```c
// quantum/remap/remap_storage_layout.h

#define REMAP_OFFSET_HEADER       (0x000)
#define REMAP_SIZE_HEADER         (16)

#define REMAP_OFFSET_KEYMAP       (REMAP_OFFSET_HEADER + REMAP_SIZE_HEADER)
#define REMAP_SIZE_KEYMAP \
    (REMAP_LAYER_COUNT * MATRIX_ROWS * MATRIX_COLS * 2)

#define REMAP_OFFSET_SLOT_TABLE   (REMAP_OFFSET_KEYMAP + REMAP_SIZE_KEYMAP)
#define REMAP_SIZE_SLOT_TABLE \
    ((REMAP_TD_SLOT_COUNT + REMAP_COMBO_SLOT_COUNT + REMAP_OVR_SLOT_COUNT) * 8)
// 論理上限は 32 slots × 8B = 256B（remap.json の合計 ≤ 32 制約）。
// 実サイズは TD/Combo/OVR の各 SLOT_COUNT に応じてビルド時決定される。

#define REMAP_OFFSET_CAPS_WORD    (REMAP_OFFSET_SLOT_TABLE + REMAP_SIZE_SLOT_TABLE)
#ifdef REMAP_CAPS_WORD_ENABLE
#  define REMAP_SIZE_CAPS_WORD    (4)
#else
#  define REMAP_SIZE_CAPS_WORD    (0)
#endif

#define REMAP_OFFSET_ONE_SHOT     (REMAP_OFFSET_CAPS_WORD + REMAP_SIZE_CAPS_WORD)
#ifdef REMAP_ONE_SHOT_ENABLE
#  define REMAP_SIZE_ONE_SHOT     (4)
#else
#  define REMAP_SIZE_ONE_SHOT     (0)
#endif

// ... Tri Layer / Repeat Key / Auto Shift / Mouse Keys / Per-Key Term / Combo TERM
//     も同様

#define REMAP_OFFSET_BACKLIGHT    (REMAP_OFFSET_COMBO_TERM + REMAP_SIZE_COMBO_TERM)
#ifdef BACKLIGHT_ENABLE
#  define REMAP_SIZE_BACKLIGHT    (2)
#else
#  define REMAP_SIZE_BACKLIGHT    (0)
#endif

#define REMAP_OFFSET_RGBLIGHT     (REMAP_OFFSET_BACKLIGHT + REMAP_SIZE_BACKLIGHT)
#ifdef RGBLIGHT_ENABLE
#  define REMAP_SIZE_RGBLIGHT     (5)
#else
#  define REMAP_SIZE_RGBLIGHT     (0)
#endif

// ... RGB Matrix / LED Matrix / Audio も同様

#define REMAP_OFFSET_MACRO        (REMAP_OFFSET_AUDIO + REMAP_SIZE_AUDIO)
#define REMAP_SIZE_MACRO          (REMAP_MACRO_BUFFER_SIZE)  // remap.json で固定
```

無効化されたモジュールは 0 バイト消費。マクロ枠を最大化できる。

### 3.7 `remap.json` Source Format

メタデータ生成の Single Source of Truth（SoT）として、各キーボードに `remap.json` を 1 個配置する。`keyboard.json` は `remap.json` から **ビルド時に派生生成** される（§5.6）。

#### 3.7.1 Position in Build Pipeline

```
remap.json （設計者が編集する SoT）
    │
    ▼ qmk generate-from-remap
    │
    ├── keyboard.json （QMK ビルド入力。VCS 管理外）
    └── build/remap_metadata.c （Remap メタデータ。VCS 管理外）
            │
            ▼ qmk compile
            │
            └── ファームウェア（keyboard.json + remap_metadata.c を含む）
                    │
                    ▼ USB HID
                    │
                    └── Remap webapp（§4 wire protocol で取得）
```

- 設計者は `remap.json` のみ編集する
- `keyboard.json` は触らない（生成物）
- ビルドパイプラインが両方の生成物（QMK 用 / Remap メタデータ用）を 1 ソースから出力

#### 3.7.2 ファイル名と配置

| 項目 | 値 |
|---|---|
| ファイル名 | `remap.json` |
| 配置 | `keyboards/<kb>/remap.json` |
| エンコーディング | UTF-8、JSON5 ではなく標準 JSON |

`keyboard.json` と同じディレクトリに配置することで、QMK の既存ディレクトリ規約を破らない。

#### 3.7.3 VCS Strategy

- **`remap.json` は git にコミットする**（SoT なので）
- **`keyboard.json` は git にコミットしない**（生成物）
  - `.gitignore` に `keyboards/*/keyboard.json` を追加
  - 例外：QMK upstream から merge した既存 `keyboard.json` の扱いについては §3.7.8 Migration Tool を参照

理由：両方コミットすると DRY 違反になり、片方だけ更新された場合の整合性が保証できなくなる。`remap.json` を SoT に固定し、`keyboard.json` は再生成可能なビルド成果物として扱う。

#### 3.7.4 Hybrid Structure（Option C）

`remap.json` のトップレベルは 3 つの起源を持つフィールドをハイブリッドで保持する：

```jsonc
{
  // === QMK keyboard.json 派生（トップレベルにそのまま展開） ===
  "manufacturer": "remap-keys",
  "keyboard_name": "Lunakey Pico",
  "maintainer": "yoichiro",
  "usb": { "vid": "0xFEED", "pid": "0x0000", "device_version": "0.0.1" },
  "matrix_pins": { "rows": [...], "cols": [...] },
  "diode_direction": "COL2ROW",
  "features": { "encoder": true, "rgblight": true, "extrakey": true },
  "rgblight": { "led_count": 12, "animations": {...} },
  "encoder": { "rotary": [{"pin_a": "GP10", "pin_b": "GP11"}] },
  "dynamic_keymap": { "layer_count": 4 },

  // === VIA info.json 派生（トップレベルに展開、ただし `layouts` 内構造化） ===
  "layouts": {
    "options": [
      { "type": "boolean", "label": "Bool Option" },
      { "type": "enum",    "label": "Enum Header", "choices": ["ch0", "ch1", "ch2"] }
    ],
    "keymap": [
      [{"x": 0, "y": 0}, "0,0", "0,1", "\n\n\n\n\n\n\n\n\ne0"],
      [{"y": 1}, "1,0", "1,1"]
    ]
  },

  // === Remap 独自（`remap` namespace に隔離） ===
  "remap": {
    "schema_version": 1,
    "slots": {
      "tap_dance":    8,
      "combo":        8,
      "key_override": 8
    },
    "per_key_term": { "max_entries": 8 },
    "macro":        { "count": 16, "buffer_size": 177 }
  }
}
```

**設計原則**：
- **QMK 由来フィールドはトップレベルそのまま**：QMK の既存スキーマと同形式を保ち、生成された `keyboard.json` への変換を最小ロジックで済ませる
- **VIA 由来フィールドもトップレベル `layouts` 配下**：webapp 既存パーサがそのまま読める KLE 構造
- **Remap 独自は `remap.*` で隔離**：QMK / VIA とのフィールド衝突を回避、将来の拡張地点を明確化

#### 3.7.5 `layouts.options` の構造

VIA info.json の `layout_options` メタデータ（boolean / enum）は **構造化された配列** として宣言する：

```jsonc
"layouts": {
  "options": [
    {
      "type":  "boolean",
      "label": "Split halves swap"
    },
    {
      "type":    "enum",
      "label":   "Bottom row layout",
      "choices": ["ANSI", "ISO", "JIS"]
    }
  ]
}
```

| フィールド | 型 | 説明 |
|---|---|---|
| `type` | `"boolean"` \| `"enum"` | オプションの種別 |
| `label` | string | UI 表示用ラベル |
| `choices` | string[] | enum 限定。選択肢ラベル配列 |

`layouts.keymap` 内の各キーは KLE label の slot 3（`"0,1"` 形式）でこの options 配列のインデックスと選択値を参照する。

#### 3.7.6 `layouts.keymap` のエンコーダ表現

VIA passthrough。KLE label の `\n` 区切り slot 9 に `e<n>` 形式でエンコーダ ID を埋め込む：

```jsonc
"keymap": [
  ["0,0", "0,1", "\n\n\n\n\n\n\n\n\ne0"]   // 3 番目のキーが encoder 0
]
```

| 表現形 | 意味 |
|---|---|
| `\n` × 9 + `e0` | 回転のみエンコーダ（matrix 位置なし） |
| `0,1\n` × 8 + `e0` | 押し込み付きエンコーダ（matrix 位置あり） |
| `e0` 単独（`a=3` or `a=7`） | コンパクト表記 |

CW / CCW の区別は KLE 上には現れない。webapp が wire protocol（§6）で `clockwise: true/false` フラグ付きで個別取得する。

#### 3.7.7 v1 Scope

**目標**：既存 Remap UI が提供する全機能を、Remap への事前キーボード登録なしで利用可能にする（self-describing keyboard / plug-and-play）。

**v1 でカバーする機能**（Remap UI 現行サポート）：

| ID | 機能 |
|---|---|
| A1 | Layer 切替（dynamic_keymap） |
| A2 | Keycode 編集 |
| A3 | Macro 編集 |
| A6 | Reset 操作 |
| B1 | Layout（KLE） |
| B2 | Layout Options |
| C1 | RGB Light（標準アニメーション） |
| C3 | Backlight |
| D1 | Encoder（CW/CCW keycode） |
| D2 | Layout Options（再掲、構造化） |
| D3 | KLE inline color（`c` 属性のみ） |
| E1 | Basic export/import |

**v1 範囲外**：

- Tap Dance（A4）、Combo（A5）— Remap UI 現行未対応
- RGB Matrix UI（C2）、Custom underglow / matrix（C4, C5）— Remap UI 現行未対応
- Centralized palette（lighting_extension）— Remap UI 現行未対応
- Advanced export/import（E2）— Remap UI 現行未対応

これらは spec 内の §10〜§19 で **wire protocol レベルでは予約済み**。webapp 側の対応を待つ。

#### 3.7.8 Migration Tool

既存の `keyboards/<kb>/keyboard.json` + Firestore 登録 info.json を入力に `remap.json` を生成する **個別キーボード変換ツール** を提供する：

```bash
qmk migrate-to-remap --keyboard <kb> [--info-json <path>] [--output keyboards/<kb>/remap.json]
```

**設計方針**：

- **一括変換しない**：全 1100+ キーボードを機械的に変換せず、設計者またはメンテナが個別キーボード単位で実行
- **冪等性**：同じ入力からは同じ出力（タイムスタンプ等を含めない）
- **dry-run 対応**：`--dry-run` で diff のみ表示
- **既存 `remap.json` 保護**：上書き時は確認プロンプト or `--force` 必須

キーボード設計者が自発的に Remap 対応を有効化するためのオプトインツール。一括移行スクリプトは bash ループで個別ツールを呼び出すだけで実装できるため、別途用意しない。

---

## 4. Wire Protocol Specification（正本）

### 4.1 HID Raw 32B Format

通信は WebHID API（`usagePage = 0xff60`、`usage = 0x61`）を使用し、すべて **32 バイト固定長パケット**で行う。

```
Byte  0:    module_id     (リクエスト/レスポンス共通)
Byte  1:    sub_cmd       (リクエスト) / status (レスポンス)
Byte 2-31:  payload       (最大 30 バイト)
```

**特殊**: `module_id == 0xFE` は probe handshake 用に予約。

### 4.2 Probe Handshake

ウェブアプリは接続時に最初に probe を送信し、Remap firmware か別の firmware（VIA、未対応 firmware 等）かを判別する。

#### Probe Request（ウェブ → ファーム）
```
Byte  0: 0xFE
Byte  1: 0x00
Byte  2-6: 'R', 'M', 'A', 'P', '?'   (magic = "RMAP?")
Byte  7-31: 0x00 (reserved)
```

#### Probe Response（ファーム → ウェブ）
```
Byte  0: 0xFE
Byte  1: 0x00
Byte  2-6: 'R', 'M', 'A', 'P', '!'   (magic = "RMAP!")
Byte  7: protocol_version             (uint8_t、整数連番 1, 2, 3, ...)
Byte  8: reserved                      (0x00)
Byte  9-10: capability_flags          (uint16_t LE、機能可否ビット)
Byte 11-31: reserved                   (0x00)
```

**識別ロジック**：
- Remap firmware は magic `"RMAP!"` を返す → ウェブは即座に認識し Remap protocol client を起動
- それ以外（VIA firmware は未知コマンド扱い等）→ ウェブ側 fallback ロジックで VIA protocol client を起動

**protocol_version の役割**：
- ウェブアプリが互換性判定を**通信の最初期**に行うために存在
- 整数連番（1, 2, 3, …）で運用。semver の minor/patch のような小数構造は持たない（プロトコルは互換 / 非互換の二値判断のため）
- 例：ウェブが `protocol_version=2` までしか知らないが、ファームが `3` を返した → 「未対応バージョンです」と即エラー表示

**capability_flags のビット割り当て**：
```
bit 0:  bootloader_available    - System module BOOTLOADER_JUMP が機能するか
bit 1:  matrix_state_available  - System module GET_MATRIX_STATE が機能するか（通常 1）
bit 2-15: reserved              - 将来拡張用
```

これにより Webapp は probe 応答だけで「フラッシュ更新ボタンを表示するか」「Matrix Tester を提供するか」等を即時判断できる。

### 4.3 Module Dispatch

probe 以降の通信は module_id ベースで dispatch される。

#### リクエスト
```
Byte  0: module_id     (例: 0x01 = Metadata)
Byte  1: sub_cmd       (例: 0x01 = GET_BASIC)
Byte 2-31: payload
```

#### レスポンス
```
Byte  0: module_id     (echoed)
Byte  1: status        (0x00 = OK、それ以外はエラー)
Byte 2-31: payload
```

#### Dispatcher 実装（中核）
```c
// quantum/remap/remap.c

void raw_hid_receive(uint8_t *data, uint8_t length) {
    static uint8_t resp[RAW_EPSIZE];
    memset(resp, 0, RAW_EPSIZE);

    // Probe handshake は最優先で fast-path 処理
    if (data[0] == 0xFE && data[1] == 0x00 &&
        memcmp(&data[2], "RMAP?", 5) == 0) {
        remap_probe_emit(resp);
        raw_hid_send(resp, RAW_EPSIZE);
        return;
    }

    // 通常 dispatch
    resp[0] = data[0];
    resp[1] = STATUS_UNKNOWN_MODULE;

    for (uint8_t i = 0; i < remap_module_count; i++) {
        uint8_t id = pgm_read_byte(&remap_modules[i].module_id);
        if (id == data[0]) {
            remap_handler_fn h =
                (remap_handler_fn)pgm_read_ptr(&remap_modules[i].handler);
            resp[1] = h(data[1], &data[2], length - 2, &resp[2]);
            break;
        }
    }

    raw_hid_send(resp, RAW_EPSIZE);
}
```

VIA dispatcher を併走させない単段構成のため、Remap firmware は VIA クライアントに対して全コマンドが「未対応」として応答する。これは仕様であり、Webapp 側で probe 結果に基づき VIA / Remap いずれの client を使うか判定する。

### 4.4 Module ID Allocation Map

```
0x00         (reserved)
0x01         Metadata
0x02         Keymap
0x03         Macro
0x04         (reserved)
0x05         System
0x06-0x0F    (reserved for future global modules)
0x10         Slot Table（TD/Combo/Override 共通スロット管理）
0x11         (reserved)
0x12         Combo（補完設定：COMBO_TERM）
0x13         Override（実体は slot のみ、追加設定なし）
0x14         Caps Word
0x15         One Shot
0x16         Tri Layer
0x17         Repeat Key
0x18         Auto Shift
0x19         Mouse Keys
0x1A         Per-Key Term
0x1B-0x1F    (reserved for Tier 2 拡張)
0x20         Backlight
0x21         RGB Light
0x22         RGB Matrix
0x23         Audio
0x24         LED Matrix
0x25-0xFD    (reserved for future modules)
0xFE         Probe handshake（予約、通常 dispatch 対象外）
0xFF         (reserved)
```

### 4.5 Error Codes（status バイト）

```
0x00  STATUS_OK
0x01  STATUS_UNKNOWN_MODULE
0x02  STATUS_UNKNOWN_SUB_CMD
0x03  STATUS_INVALID_PAYLOAD_LEN
0x04  STATUS_INVALID_PARAMS（範囲外 slot index 等）
0x05  STATUS_EEPROM_WRITE_FAIL
0x06  STATUS_NOT_INITIALIZED
0xFF  STATUS_INTERNAL_ERROR
```

### 4.6 Endianness

すべての多バイト整数は **little-endian**（AVR / ARM どちらも native LE のため）。`__attribute__((packed))` 構造体で直接シリアライズ可能。

### 4.7 Multi-packet Transfer Protocol（Stateless Offset-based）

HID 1 パケット 30B payload に収まらないデータ（string table、layout、keymap buffer、macro buffer 等）の転送を、**stateless offset-based** で統一的に扱う。

#### 設計思想

- ファーム側で cursor / state を持たない（AVR で割り込み・USB 切断・並行クライアントを考えると stateless が最適）
- すべての chunked GET/SET はリクエストに `offset` を明示
- レスポンスは `total_bytes`（全体サイズ）と `chunk_len`（このパケットで返した実バイト数）を返す
- EOF はサイズで判定（特別な END マーカー不要）

#### Wire Format（GET 系）

**リクエスト（32B）**
```
Byte 0    : module_id
Byte 1    : sub_cmd
Byte 2-3  : offset (uint16_t LE、要求するバイト位置)
Byte 4-31 : reserved (zero)
```

**レスポンス（32B）**
```
Byte 0    : module_id (echoed)
Byte 1    : status (0x00 OK)
Byte 2-3  : total_bytes  (uint16_t LE、転送対象の全体サイズ、毎回返す)
Byte 4-5  : offset_echoed (uint16_t LE、要求 offset の echo)
Byte 6    : chunk_len (uint8_t、本パケットのデータ実バイト数、0-26)
Byte 7-31 : data (chunk_len バイト + zero-fill)
```

#### Wire Format（SET 系、対称適用）

**リクエスト（32B）**
```
Byte 0    : module_id
Byte 1    : sub_cmd
Byte 2-3  : offset (uint16_t LE、書き込み開始位置)
Byte 4    : chunk_len (uint8_t、書き込むデータバイト数、0-25)
Byte 5-29 : data (chunk_len バイト + zero-fill)
Byte 30-31: reserved (zero)
```

**レスポンス（32B）**
```
Byte 0    : module_id
Byte 1    : status
Byte 2-31 : reserved (zero)
```

#### ホスト側 GET ループ（リファレンス疑似コード）

```python
offset = 0
buffer = bytearray()
while True:
    resp = send(module_id, sub_cmd, offset=offset)
    buffer.extend(resp.data[:resp.chunk_len])
    offset += resp.chunk_len
    if offset >= resp.total_bytes:
        break  # ← EOF はサイズで判定、特別な END マーカー不要
    if resp.chunk_len == 0:
        raise Error  # 進まない＝バグ
```

#### 採用根拠

- AVR 上で state を持たないのが鉄則：割り込み・USB 切断・並行クライアントを考えると stateless が最強
- §9.4 既存の `GET_SLOT(type, idx)` がすでに stateless offset-style なので、spec 全体で揃う
- payload 26B vs 28B の差は実用上誤差（string table が数百 B → ±数パケット差にしかならない）

#### 適用 sub_cmd 一覧

本プロトコルは以下の sub_cmd で使用される：

- `Metadata.GET_STRING_TABLE`、`GET_LAYOUTS`、`GET_LED_POSITIONS`、`GET_CUSTOM_KEYCODES`、`GET_LAYOUT_OPTIONS_META`
- `Keymap.GET_BUFFER`、`SET_BUFFER`
- `Macro.GET_BUFFER`、`SET_BUFFER`

将来 chunked transfer が必要な sub_cmd を追加する場合も、本プロトコルに従う。

---

## 5. Module: Metadata（module_id=0x01）

VIA JSON 撤廃の中核。キーボードのあらゆるメタデータを動的応答する。加えて、`layout_options` の SET 系も提供する（旧 VIA `id_set_keyboard_value(id_layout_options)` 相当）。

### 5.1 Sub-commands

```
0x01  GET_BASIC                     (基本情報 26B)
0x02  GET_STRING_TABLE              (string table 取得、§4.7 chunked)
0x03  GET_LAYOUTS                   (KLE レイアウト取得、§4.7 chunked)
0x04  GET_LED_POSITIONS             (LED 位置取得、§4.7 chunked)
0x05  GET_CUSTOM_KEYCODES           (カスタムキーコード取得、§4.7 chunked)
0x06  GET_LAYOUT_OPTIONS_STATE      (layout options 状態 4B 取得)
0x07  SET_LAYOUT_OPTIONS_STATE      (layout options 状態 4B 設定)
0x08  GET_LAYOUT_OPTIONS_META       (layout options メタデータ取得、§4.7 chunked)
```

旧 spec の `GET_xxx_BEGIN` / `GET_xxx_CONTINUE` ペアは §4.7 stateless offset-based に統合され、各機能 1 sub_cmd に削減された。旧 `GET_ENCODER_INFO` は webapp 側で利用機会が無いため廃止（encoder の数は §5.2 `num_encoders` で取得、keymap は Keymap module で取得）。

### 5.2 `remap_meta_basic_t` 構造（26B）

```c
typedef struct __attribute__((packed)) {
    /* Version 情報（先頭固定）*/
    uint8_t  protocol_version;       // 1B：probe と同値、自己整合性チェック用
    uint8_t  qmk_version_major;      // 1B：例 0
    uint8_t  qmk_version_minor;      // 1B：例 32
    uint8_t  qmk_version_patch;      // 1B：例 12

    /* Identity */
    uint16_t vendor_id;              // 2B
    uint16_t product_id;             // 2B
    uint16_t firmware_version;       // 2B（remap.json 由来、表示用）

    /* Hardware */
    uint8_t  matrix_rows;            // 1B
    uint8_t  matrix_cols;            // 1B
    uint8_t  num_layers;             // 1B
    uint8_t  num_layouts;            // 1B
    uint8_t  num_encoders;           // 1B
    uint8_t  num_custom_keycodes;    // 1B
    uint8_t  tap_dance_slot_count;   // 1B（keycode 帯動的サイズ通知用）
    uint8_t  reserved_h0;            // 1B（alignment）

    /* Features */
    uint16_t feature_flags;          // 2B
    uint16_t rgb_matrix_led_count;   // 2B
    uint16_t led_matrix_led_count;   // 2B

    uint8_t  reserved_tail[2];       // 2B（将来追加フィールド用）
} remap_meta_basic_t;
```

#### 余り 4B の扱い
HID 30B payload に対して struct 26B → 余り 4B。すべて `0x00` zero-fill する。

**プロトコル拡張ポリシー**：将来フィールド追加時、`reserved_tail` から順に消費する。最後に追加されたフィールドの位置は **protocol_version の bump** で示す。

### 5.3 String Table（UTF-8、§4.7 chunked）

キーボード名・KLE label・カスタムキーコード名等を格納する**長さ前置文字列の配列**。エンコーディングは **UTF-8**（日本語キーボードの label にも対応）。

```
[0]: keyboard_name        (例: "Lily58 Pro")
[1]: layout_name_0        (例: "LAYOUT_split_3x6_5")
[2]: layout_name_1        (...)
[N]: custom_keycode_0_name (例: "Hyper")
...
```

#### 各エントリのフォーマット
```
[length: uint8_t][bytes: UTF-8 sequence (length バイト)]
```

最大 255 バイト/エントリ。30B payload に収まらない場合、§4.7 stateless offset-based で chunked 取得。

### 5.4 Layouts / LED Positions（§4.7 chunked）

#### KLE Layout の表現

各キー位置は **Q6.2 固定小数点**（`uint8_t × 0.25u`、範囲 0-63.75u）で幾何情報をエンコード。1 構造体に幾何 + 視覚属性 + 関連付けを統合保持する：

```c
typedef struct __attribute__((packed)) {
    /* Matrix 位置（非 matrix キー = 0xFF, 0xFF）*/
    uint8_t  matrix_row;        // 1B
    uint8_t  matrix_col;        // 1B

    /* 幾何（Q6.2 固定小数点、0.25u 単位）*/
    uint8_t  x_q6_2;            // 1B
    uint8_t  y_q6_2;            // 1B
    uint8_t  w_q6_2;            // 1B
    uint8_t  h_q6_2;            // 1B

    /* 関連付け */
    uint8_t  encoder_id;        // 1B：エンコーダ番号（0xFF=非エンコーダ）
    uint8_t  layout_opt_idx;    // 1B：所属する layout option index（0xFF=該当なし）
    uint8_t  layout_opt_val;    // 1B：option 内での選択値（boolean=0/1, enum=index）
    uint8_t  flags;             // 1B：bit0=is_decal、bit1-7=reserved

    /* 視覚 */
    uint8_t  color_r;           // 1B：KLE `c` 属性 RGB（未指定時 0x00）
    uint8_t  color_g;           // 1B
    uint8_t  color_b;           // 1B

    /* 回転 */
    uint16_t rotation_q8_8;     // 2B：Q8.8 度（0.00390625° 単位、-180.0〜+179.996）
    uint8_t  reserved;          // 1B：将来拡張用
} remap_layout_key_t;           // 16B/key
```

**フィールド設計の根拠**：

| フィールド | 由来 | センチネル/初期値 |
|---|---|---|
| `matrix_row`, `matrix_col` | KLE label slot 0（`"row,col"`）or slot 3（layout option 付きキー） | 非 matrix キーは `0xFF, 0xFF` |
| `x/y/w/h_q6_2` | KLE 標準座標 | 0 始まり |
| `encoder_id` | KLE label slot 9（`e<n>` パターン）or `a=3/7` 単独 label | 非エンコーダは `0xFF` |
| `layout_opt_idx`, `layout_opt_val` | KLE label slot 3 経由の layout option 関連付け | 該当なしは idx=`0xFF` |
| `flags` | KLE `d` 属性（decal）等 | 0 |
| `color_r/g/b` | KLE `c` 属性 | 未指定時 0x000000（webapp 側でデフォルト色適用） |
| `rotation_q8_8` | KLE `r` 属性（Q8.8 度） | 0 |

**サイズ試算**：16 B/key × ~70 key（典型 60%）= ~1120 B FLASH。1 パケット 26B payload 内に 1 key 収まる（パディングあり）→ ~70 パケットで全 layout 取得（§4.7 chunked）。

#### LED Positions

RGB matrix / LED matrix が ON の場合のみ。同様の固定長構造で取得（§4.7 chunked）。`remap_led_position_t` の正式定義は Open Items 参照。

#### LED Positions

RGB matrix / LED matrix が ON の場合のみ。同様に 8B/LED 構造で取得（§4.7 chunked）。`remap_led_position_t` の正式定義は Open Items 参照。

### 5.5 Layout Options（State / Metadata）

Layout options は 2 種類のデータ層を持つ：

| 層 | 内容 | 格納場所 | sub_cmd |
|---|---|---|---|
| **Metadata** | option 定義（type, label, choices）— 静的、設計時固定 | Flash（PROGMEM、`remap.json` 由来） | `GET_LAYOUT_OPTIONS_META` (0x08) |
| **State** | 各 option の現在の選択値 — 動的、ユーザー操作で変化 | EEPROM（Remap Header §3.4） | `GET_LAYOUT_OPTIONS_STATE` (0x06) / `SET_LAYOUT_OPTIONS_STATE` (0x07) |

#### 5.5.1 `remap_layout_option_t` 構造（4B/option、PROGMEM）

```c
typedef struct __attribute__((packed)) {
    uint8_t  type;                  // 0x00=boolean, 0x01=enum
    uint8_t  label_str_idx;         // String Table の index（label 文字列）
    uint8_t  choice_count;          // boolean=0、enum=N（最大 16）
    uint8_t  choices_str_idx_base;  // String Table の連続 N 個（base, base+1, ..., base+N-1）
} remap_layout_option_t;            // 4B/option
```

**設計根拠**：
- 文字列はすべて String Table（§5.3）経由で indirect → option メタデータ自体は固定長
- `enum` の choices は String Table の **連続範囲** に配置することで `choices_str_idx_base + i` で参照可能（生成器側の責務）
- 4B × 最大 32 options = 最大 128 B FLASH

**例**（remap.json `layouts.options` → メタデータ展開）：

```jsonc
// remap.json
"options": [
  { "type": "boolean", "label": "Split halves swap" },
  { "type": "enum",    "label": "Bottom row", "choices": ["ANSI", "ISO", "JIS"] }
]
```

```c
// remap_metadata.c (auto-generated)
const char remap_str_5[] PROGMEM = "Split halves swap";
const char remap_str_6[] PROGMEM = "Bottom row";
const char remap_str_7[] PROGMEM = "ANSI";
const char remap_str_8[] PROGMEM = "ISO";
const char remap_str_9[] PROGMEM = "JIS";

const remap_layout_option_t remap_layout_options[] PROGMEM = {
    { .type = 0x00, .label_str_idx = 5, .choice_count = 0, .choices_str_idx_base = 0 },
    { .type = 0x01, .label_str_idx = 6, .choice_count = 3, .choices_str_idx_base = 7 },
};
```

#### 5.5.2 `GET_LAYOUT_OPTIONS_META` (0x08)

§4.7 chunked で全 option メタデータを取得。`total_bytes = sizeof(remap_layout_option_t) × num_options = 4 × num_options`。

webapp は §5.2 の `remap_meta_basic_t` から option 数を取得済み（→ Open Items：`num_layout_options` フィールドを `remap_meta_basic_t` に追加するか検討）して、必要 byte 数を chunked 取得。受信後は String Table（§5.3）の対応エントリを参照して label / choices 文字列を解決する。

#### 5.5.3 `GET_LAYOUT_OPTIONS_STATE` (0x06)

```
Request (32B):
  Byte 0:    0x01 (module_id)
  Byte 1:    0x06 (sub_cmd)
  Byte 2-31: 0x00

Response (32B):
  Byte 0:    0x01
  Byte 1:    status
  Byte 2-5:  layout_options_state (uint32 LE、bit field)
  Byte 6-31: 0x00
```

#### 5.5.4 `SET_LAYOUT_OPTIONS_STATE` (0x07)

```
Request (32B):
  Byte 0:    0x01
  Byte 1:    0x07
  Byte 2-5:  layout_options_state (uint32 LE)
  Byte 6-31: 0x00

Response (32B):
  Byte 0:    0x01
  Byte 1:    status (0x00=OK, 0x05=EEPROM_WRITE_FAIL)
  Byte 2-31: 0x00
```

**SET 後の挙動**：§3.5 RAM Cache + Save-on-Demand に準拠（dirty フラグ立てて遅延 commit、RAM cache は即更新で実行時挙動は即時反映）。

**EEPROM 配置**：Remap Header の `layout_options` フィールド（§3.4）に格納。VIA eeconfig 領域は使わない。

**bit field エンコーディング**：`remap_layout_option_t[i].type` に応じて：
- `boolean`：1 bit を 1 個割り当て（option N が値 v なら `state |= (v & 1) << bit_offset_for_n`）
- `enum`：`ceil(log2(choice_count))` bit を割り当て（4 choices なら 2 bit）
- bit 割り当ては option 配列の順序に従い LSB 側から累積。32 bit 上限を超える場合はビルド時エラー。

### 5.6 Build-time Generation

メタデータの大半（layout、string table、layout_options 等）は **`remap.json` から自動生成** される（§3.7 参照）。`remap.json` を SoT として、ビルド時に以下 2 つを派生生成する：

1. `keyboard.json` — QMK ビルド入力（VCS 管理外）
2. `build/remap_metadata.c` — Remap メタデータ C ソース（VCS 管理外）

#### `qmk generate-from-remap` CLI

QMK CLI の拡張サブコマンド。`remap.json` を入力に **両方の生成物** を一度に出力：

```bash
qmk generate-from-remap --keyboard <kb>
# → keyboards/<kb>/keyboard.json
# → build/remap_metadata.c
```

オプション：
- `--keyboard-json-only`：`keyboard.json` のみ生成（QMK 互換チェック用）
- `--metadata-only`：`build/remap_metadata.c` のみ生成
- `--check`：再生成せず既存生成物が最新か検証（CI 用）

`keyboard.json` 生成は §3.7.4 のトップレベル QMK 派生フィールド（`manufacturer`, `usb`, `matrix_pins`, `features`, `rgblight`, `encoder`, `dynamic_keymap` 等）と `layouts.keymap`（VIA 由来だが QMK の `layouts.<name>.layout` にもマップされる KLE 形式）をそのまま転記し、`remap` namespace と `layouts.options` は除外する。

メタデータ C ソース生成例：
```c
// build/remap_metadata.c (auto-generated)
#include "remap_metadata.h"

const remap_meta_basic_t remap_meta_basic PROGMEM = {
    .protocol_version    = 1,
    .qmk_version_major   = 0,
    .qmk_version_minor   = 32,
    .qmk_version_patch   = 12,
    .vendor_id           = 0xFEED,
    // ...
};

const char *const remap_string_table[] PROGMEM = {
    "Lily58 Pro",
    "LAYOUT_split_3x6_5",
    // ...
};

const remap_layout_key_t remap_layout_0[] PROGMEM = {
    { .matrix_row = 0, .matrix_col = 0, .x_q6_2 = 0,  .y_q6_2 = 0,  .w_q6_2 = 4, .h_q6_2 = 4 },
    // ... 70 entries
};
```

#### make 統合

`build_keyboard.mk` に組み込み、ビルド毎に自動生成。`remap.json` のタイムスタンプを依存元として両方の生成物が再生成される：

```makefile
$(KEYBOARD_PATH)/keyboard.json $(BUILD_DIR)/remap_metadata.c: $(KEYBOARD_PATH)/remap.json
	$(QMK) generate-from-remap --keyboard $(KEYBOARD)
```

PROGMEM 消費は典型 60% キーボードで約 1.1 KB FLASH。

---

## 6. Module: Keymap（module_id=0x02）

VIA `id_dynamic_keymap_*` 群の Remap 版実装。layer × row × col の単一キー編集、bulk transfer、reset、encoder keycode 編集を提供する。

### 6.1 Sub-commands

```
0x01  GET_KEYCODE              ([layer:1, row:1, col:1])               → [keycode:2 LE]
0x02  SET_KEYCODE              ([layer:1, row:1, col:1, keycode:2 LE])  → []
0x03  RESET                    ()                                       → []
0x04  GET_BUFFER               ([offset:2 LE])                          → §4.7 chunked
0x05  SET_BUFFER               ([offset:2 LE, chunk_len:1, data:≤25])   → []
0x06  GET_ENCODER_KEYCODE      ([layer:1, encoder_id:1, direction:1])   → [keycode:2 LE]
0x07  SET_ENCODER_KEYCODE      ([layer:1, encoder_id:1, direction:1, keycode:2 LE]) → []
```

`direction`: `0x00 = counterclockwise`, `0x01 = clockwise`

### 6.2 Index 方式

VIA と同じく **三次元 `(layer, row, col)`** で索引する。Keymap buffer のメモリ配置は `layer × matrix_rows × matrix_cols × 2 bytes`（layer-major）。

`GET_BUFFER` の `total_bytes` = `num_layers × matrix_rows × matrix_cols × 2`。

### 6.3 詳細仕様

#### `GET_KEYCODE` (0x01)
```
Request (32B):
  Byte 0:    0x02 (module_id)
  Byte 1:    0x01 (sub_cmd)
  Byte 2:    layer
  Byte 3:    row
  Byte 4:    col
  Byte 5-31: 0x00

Response (32B):
  Byte 0:    0x02
  Byte 1:    status (0x00=OK, 0x04=INVALID_PARAMS)
  Byte 2-3:  keycode (uint16 LE)
  Byte 4-31: 0x00
```

#### `SET_KEYCODE` (0x02)
```
Request (32B):
  Byte 0:    0x02
  Byte 1:    0x02
  Byte 2:    layer
  Byte 3:    row
  Byte 4:    col
  Byte 5-6:  keycode (uint16 LE)
  Byte 7-31: 0x00

Response (32B):
  Byte 0:    0x02
  Byte 1:    status (0x00=OK, 0x04=INVALID_PARAMS, 0x05=EEPROM_WRITE_FAIL)
  Byte 2-31: 0x00
```

#### `RESET` (0x03)
全 layer × row × col を **PROGMEM の `keymaps[][][]` 配列で初期化**（QMK の `dynamic_keymap_reset()` 相当）。encoder 用 keycode も `encoder_map[][][]` PROGMEM 値で初期化。

#### `GET_BUFFER` / `SET_BUFFER` (0x04 / 0x05)
§4.7 案 A の **stateless offset-based protocol** を流用。Keymap region 全体を bulk transfer 可能。Webapp 起動時の初期 fetch は bulk、編集後の単発書き込みは `SET_KEYCODE` という使い分けを想定。

#### `GET_ENCODER_KEYCODE` / `SET_ENCODER_KEYCODE` (0x06 / 0x07)
encoder の clockwise / counterclockwise キー設定。encoder 数は §5.2 metadata `num_encoders` で取得済み前提。Encoder データは小さいため bulk transfer は提供しない。

### 6.4 Keycode 範囲チェック方針

`SET_KEYCODE` / `SET_ENCODER_KEYCODE` / `SET_BUFFER` で受信した keycode の **範囲チェックは行わない**（webapp 側が責任を持つ）。これにより protocol が単純化され、ファーム実装サイズも削減される。

### 6.5 EEPROM 配置

Keymap region は §3.3 の `0x010` から `REMAP_LAYER_COUNT × MATRIX_ROWS × MATRIX_COLS × 2` バイト。Encoder keycode は別領域（remap.json で設定、典型 4 layer × 2 enc × 2 dir × 2B = 32B 程度）に配置。詳細レイアウト確定は Open Items 参照。

---

## 7. Module: Macro（module_id=0x03）

VIA macro 5 sub_cmd（`get_count` / `get_buffer_size` / `get_buffer` / `set_buffer` / `reset`）を **4 sub_cmd に集約**。

### 7.1 Sub-commands

```
0x01  GET_INFO       ()                                      → [count:1, buffer_size:2 LE]
0x02  GET_BUFFER     ([offset:2 LE])                         → §4.7 chunked
0x03  SET_BUFFER     ([offset:2 LE, chunk_len:1, data:≤25])  → []
0x04  RESET          ()                                       → []
```

### 7.2 詳細仕様

#### `GET_INFO` (0x01)
1 packet で macro 全体情報取得。`get_count` + `get_buffer_size` を統合。

```
Request (32B):
  Byte 0:    0x03 (module_id)
  Byte 1:    0x01 (sub_cmd)
  Byte 2-31: 0x00

Response (32B):
  Byte 0:    0x03
  Byte 1:    status
  Byte 2:    count        (uint8、サポートする macro 数、典型 16)
  Byte 3-4:  buffer_size  (uint16 LE、macro 領域のバイト数)
  Byte 5-31: 0x00
```

#### `GET_BUFFER` / `SET_BUFFER` (0x02 / 0x03)
§4.7 案 A をそのまま流用。`total_bytes` は `buffer_size` と一致。

#### `RESET` (0x04)
Macro buffer 全体を **0x00 で埋める**。

### 7.3 Macro Buffer データ構造

VIA との互換性は不要だが、**null-terminator 区切り方式**を採用：

```
[macro_0_bytes][0x00][macro_1_bytes][0x00]...[macro_N-1_bytes][0x00][unused 0x00 fill]
```

採用理由：
1. **実装シンプル**：macro N を取り出すには先頭から N 個目の `0x00` までを読むだけ
2. **可変長対応**：各 macro が異なる長さを持てる
3. **空 macro 表現**：`[0x00]` のみ = 空 macro
4. **既存 QMK の `dynamic_keymap_macro_send()` ロジックそのまま流用可能**（QMK fork の利点）

#### Macro データの内部表現
QMK の既存仕様を踏襲：
- 通常文字：そのまま 1 バイト
- 特殊シーケンス：`SS_TAP_CODE / SS_DOWN_CODE / SS_UP_CODE / SS_DELAY_CODE` 等の escape sequence
- これは QMK core 側の `dynamic_keymap_macro_send()` がそのまま解釈する

### 7.4 個別 macro の get/set

提供しない（`GET_BUFFER` / `SET_BUFFER` の bulk のみ）。Macro 編集の頻度は低いので buffer 一括書き戻しで十分、sub_cmd 削減で protocol を clean に保つ。

### 7.5 Build-time 制御

`rules.mk` に `REMAP_MACRO_ENABLE = yes` を設定したキーボードのみ Macro module が有効化される（opt-in）。EEPROM 容量制約（特に 32U4）に配慮し、必要なキーボードだけ有効化する方針。Macro buffer サイズは `remap.json` の `remap.macro.buffer_size` で固定（典型 161-196B）。

### 7.6 Macro count の通知方法

§5.2 metadata `remap_meta_basic_t` には macro count を含めない。Macro module の `GET_INFO` で取得することで、Metadata 構造体の肥大化を避け責務分離を保つ。

---

## 8. Module: System（module_id=0x05）

VIA の `id_eeprom_reset` / `id_bootloader_jump` / `id_get_keyboard_value(id_switch_matrix_state)` を統合した system 制御モジュール。

### 8.1 Sub-commands

```
0x01  GET_MATRIX_STATE   ()  → [bitmap_size:1, bitmap:N]
0x02  EEPROM_RESET       ()  → []
0x03  BOOTLOADER_JUMP    ()  → []  (送信完了後 reset_keyboard())
```

### 8.2 詳細仕様

#### `GET_MATRIX_STATE` (0x01)
キー押下状態を bit packed bitmap で返す。webapp Matrix Tester が ~60 Hz でポーリング想定。

```
Request (32B):
  Byte 0:    0x05 (module_id)
  Byte 1:    0x01 (sub_cmd)
  Byte 2-31: 0x00

Response (32B):
  Byte 0:    0x05
  Byte 1:    status
  Byte 2:    bitmap_size  (uint8、bitmap バイト数 = ceil(rows × cols / 8))
  Byte 3-?:  bitmap       (key (r,c) → bit (r × MATRIX_COLS + c))
  Byte ?-31: 0x00
```

**bitmap サイズ試算**：
- 60% (5×14 = 70 keys)：9 B
- TKL (6×17 = 102 keys)：13 B
- フルサイズ (6×21 = 126 keys)：16 B
- ergodox (14×7 = 98 keys)：13 B

→ **30B payload 内で全実用キーボード対応可能**（最大 240 keys まで）。240 keys 超は将来の chunked 対応として punt。

#### `EEPROM_RESET` (0x02)
**全 EEPROM をゼロ初期化** → Remap Header magic 書き込み → 起動時の post_init で各モジュール defaults 復元。§3.5 の「初回起動・EEPROM 整合性ポリシー」を手動トリガー。

reset 完了後の **MCU 自動 reboot は行わない**。Webapp が必要時に明示的に `BOOTLOADER_JUMP` 等で対応。

#### `BOOTLOADER_JUMP` (0x03)
DFU bootloader 起動。フラッシュ更新時に webapp から呼び出される。

実装：レスポンス送信完了後に `reset_keyboard()` を呼ぶ。送信前に jump すると webapp タイムアウト扱いになるため、順序は厳守。

無効ビルド（`BOOTLOADER = none` 等）の場合：sub_cmd 自体を未対応扱い → `STATUS_UNKNOWN_SUB_CMD` 返却。Webapp は §4.2 `capability_flags.bit0 = bootloader_available` で事前判定可能。

### 8.3 将来拡張余地

uptime / device_indication 等の VIA 機能は本仕様で「Remap 不要」と判断され削除されたが、将来 debug 機能で必要になった場合、System module の sub_cmd `0x04` 以降に追加可能。

---

## 9. Module: Slot Table（module_id=0x10）

Goal 2 の中核。Tap Dance / Combo / Key Override の動的設定を統一スロット構造で管理する。

### 9.1 `custom_slot_t` 統一構造（8B）

```c
typedef struct __attribute__((packed)) {
    uint8_t  type;       // 0x00=unused, 0x01=TD, 0x02=Combo, 0x03=Override
    uint8_t  flags;      // モジュール固有のビット（enabled 含む）
    uint16_t param0;
    uint16_t param1;
    uint16_t param2;
} custom_slot_t;         // 8B 固定
```

各モジュールは `param0/1/2` の意味を独自定義する（後述）。

### 9.2 モジュール別配分（β 配分方式）

統一プールではなく、モジュール別に**ビルド時固定の専用配列**を持つ：

```c
custom_slot_t td_slots[REMAP_TD_SLOT_COUNT];      // default 8
custom_slot_t combo_slots[REMAP_COMBO_SLOT_COUNT]; // default 8
custom_slot_t ovr_slots[REMAP_OVR_SLOT_COUNT];    // default 8
// 合計 ≤ 32（slot 数の論理上限）
```

**配分のオーバーライド**は `remap.json` で行う：

```json
{
  "remap": {
    "slots": {
      "tap_dance":    8,
      "combo":        8,
      "key_override": 8
    }
  }
}
```

合計 32 を超える設定はビルドエラー。

**β 配分の選択理由**：
- AVR scan loop が短縮される（型判別ループ不要）
- ビルド時にメモリ完全予測
- ウェブ UI も「TD タブ / Combo タブ / Override タブ」と分離するのが自然

### 9.3 keycode マッピング（動的予約）

Tap Dance のみ keymap 配置が必要なため、**TD slot 数に応じて keycode 帯が動的にサイズ変動**：

```
0x7800 ～ 0x7800 + REMAP_TD_SLOT_COUNT - 1   REMAP_USER_0..N (TD slot index 参照)
0x7820 ～ 0x783F                              (Combo Plus 将来用、予約のみ)
```

ウェブアプリは GET_BASIC レスポンスの `tap_dance_slot_count` で TD keycode 範囲を動的判定する。

**Combo / Override は scan 方式**で keymap 配置不要のため、keycode 帯を消費しない。

### 9.4 Sub-commands

```
0x01  GET_SLOT(type:1B, idx:1B)               → 8B slot data
0x02  SET_SLOT(type:1B, idx:1B, data:8B)      → 1B status
0x03  CLEAR_SLOT(type:1B, idx:1B)             → 1B status (type=0x00 にリセット)
0x04  COMMIT_TO_EEPROM()                      → 1B status (RAM cache → EEPROM)
```

- `GET_SLOT` は逐次取得（24 slot で約 500ms、起動時 1 回限定なので許容）
- バリデーションは**ファーム側で slot_index 範囲チェックのみ**。型・keycode 整合性はウェブ UI 側で事前検証する責務。

### 9.5 空 slot の扱い

ユーザーが keymap に `REMAP_USER_5` を配置したが TD slot 5 の `type == 0x00 (unused)` の場合：
- **ファーム側**: 完全に no-op（押しても何も起きない）
- **ウェブ側**: UI で「slot 未定義」を警告表示

ファームに警告 LED 等のロジックは持たせない（FLASH 節約）。

---

## 10. Module: Tap Dance Lite（slot type=0x01）

QMK Tap Dance を facade として活用する設計。

### 10.1 QMK facade 実装方針

QMK の `tap_dance_actions[]` は通常 `static const` 配列だが、Remap fork で **non-const + 外部参照可能なポインタ**に core patch する：

```c
// quantum/process_keycode/process_tap_dance.c (Remap fork で patch)
// Before: static const qk_tap_dance_action_t tap_dance_actions[] = { ... };
qk_tap_dance_action_t *tap_dance_actions = NULL;
uint8_t tap_dance_count = 0;
```

Remap モジュールは `td_slots[]` から QMK の `qk_tap_dance_action_t` を**動的構築**する：

```c
// quantum/remap/modules/tap_dance_lite/tap_dance_lite.c
static qk_tap_dance_action_t remap_td_actions[REMAP_TD_SLOT_COUNT];

void remap_tap_dance_lite_init(void) {
    for (uint8_t i = 0; i < REMAP_TD_SLOT_COUNT; i++) {
        custom_slot_t *slot = &td_slots[i];
        if (slot->type == 0x01 && (slot->flags & TD_FLAG_ENABLED)) {
            remap_td_actions[i] = (qk_tap_dance_action_t)
                ACTION_TAP_DANCE_TAP_HOLD(slot->param0, slot->param1);
            // tap=param0, hold=param1, double_tap=param2 を組み合わせる
            // （詳細は 10.4 参照）
        }
    }
    tap_dance_actions = remap_td_actions;
    tap_dance_count   = REMAP_TD_SLOT_COUNT;
}
```

### 10.2 Slot レイアウト

```c
// type = 0x01 (Tap Dance)
{
    type:    0x01,
    flags:   bit0=enabled, bit1=permissive_hold,
             bit2=hold_on_other_key_press, bit3=quick_tap_term,
             bit4-7=reserved,
    param0:  tap_kc           (1 tap で送信される keycode)
    param1:  hold_kc          (hold で送信される keycode)
    param2:  double_tap_kc    (2 taps で送信される keycode)
}
```

### 10.3 flags ビットレイアウト

QMK 標準の挙動制御に対応するビット：

| bit | 名前 | QMK 対応機能 |
|-----|------|------|
| 0 | `enabled` | slot を有効化 |
| 1 | `permissive_hold` | modifier を即発動 |
| 2 | `hold_on_other_key_press` | 他キー押下で hold 確定 |
| 3 | `quick_tap_term` | 高速連打で tap 確定 |
| 4-7 | reserved | 将来拡張用 |

これらは QMK の per-key callback（`get_permissive_hold` 等）を Remap が実装し、slot の flags を読んで応答する形で実現する。

### 10.4 制約

- **callback 関数ポインタ不可**：QMK の `ACTION_TAP_DANCE_FN` 系統は使えない（C 関数ポインタ EEPROM 保存不可）
- **2-tap + hold まで**：slot 8B で表現できる action は 3 種（tap / hold / double_tap）が物理限界
- **TAPPING_TERM はグローバル固定**：slot 別調整は持たない（必要なら Per-Key Term モジュール経由）

### 10.5 Future: Tap Dance Plus

3-tap 以上を将来サポートする場合に備えて：
- keycode 帯 `0x7840-0x785F` を**予約のみ**（実装は Tier 2 第二弾）
- 別構造体 `remap_tap_dance_extended_t`（16B 等）を別 EEPROM 領域に確保する設計余地

---

## 11. Module: Combo Lite（slot type=0x02）

QMK Combo を facade として活用。

### 11.1 QMK combo_t facade

`tap_dance_actions[]` と同様、QMK の `combo_t key_combos[]` を core patch で外部参照化し、Remap が動的構築する。

```c
// QMK fork で patch
combo_t *key_combos = NULL;
uint16_t COMBO_LEN = 0;
```

```c
// Remap 動的構築
// 注：QMK 既定の combo_t は keys を const ポインタで持つが、Remap fork では
// combo_keys 配列も非 const として扱う必要がある（実行時に書き換えるため）。
// 関連する core patch は core_patches/003 を参照（combo_t.keys の const 削除）。
static combo_t  remap_combos[REMAP_COMBO_SLOT_COUNT];
static uint16_t combo_keys[REMAP_COMBO_SLOT_COUNT][3];  // 2-key + COMBO_END

void remap_combo_lite_init(void) {
    for (uint8_t i = 0; i < REMAP_COMBO_SLOT_COUNT; i++) {
        custom_slot_t *slot = &combo_slots[i];
        if (slot->type == 0x02 && (slot->flags & COMBO_FLAG_ENABLED)) {
            combo_keys[i][0] = slot->param0;     // trigger key 0
            combo_keys[i][1] = slot->param1;     // trigger key 1
            combo_keys[i][2] = COMBO_END;
            remap_combos[i].keycode = slot->param2;  // output keycode
            remap_combos[i].keys    = combo_keys[i];
        }
    }
    key_combos = remap_combos;
    COMBO_LEN  = REMAP_COMBO_SLOT_COUNT;
}
```

### 11.2 Slot レイアウト（2-key trigger 限定）

```c
// type = 0x02 (Combo)
{
    type:    0x02,
    flags:   bit0=enabled, bit1-7=reserved,
    param0:  trigger_kc_0  (同時押し対象キー 0)
    param1:  trigger_kc_1  (同時押し対象キー 1)
    param2:  output_kc     (出力されるキーコード)
}
```

### 11.3 補完設定（module_id=0x12）

slot に収まらないグローバル設定として `COMBO_TERM` を持つ：

```c
struct combo_global_config {
    uint16_t combo_term_ms;     // default 50ms
};
```

Sub-commands:
```
0x01  GET_CONFIG()       → 2B
0x02  SET_CONFIG(2B)     → 1B status
```

EEPROM 消費 2B。

### 11.4 Future: Combo Plus

n-key trigger（3-16 keys）対応は将来の Tier 2 拡張：
- keycode 帯 `0x7820-0x783F` を予約のみ
- 別構造体（可変長 combo_ex_table）を別 EEPROM 領域に確保する設計余地

---

## 12. Module: Key Override Lite（slot type=0x03）

QMK Key Override を facade として活用。

### 12.1 QMK key_override_t facade

```c
// QMK fork で patch
const key_override_t **key_overrides = NULL;
```

```c
// Remap 動的構築
static key_override_t        remap_overrides[REMAP_OVR_SLOT_COUNT];
static const key_override_t *remap_override_ptrs[REMAP_OVR_SLOT_COUNT + 1];  // NULL 終端

void remap_key_override_lite_init(void) {
    uint8_t cnt = 0;
    for (uint8_t i = 0; i < REMAP_OVR_SLOT_COUNT; i++) {
        custom_slot_t *slot = &ovr_slots[i];
        if (slot->type == 0x03 && (slot->flags & OVR_FLAG_ENABLED)) {
            remap_overrides[cnt] = (key_override_t){
                .trigger             = slot->param0,
                .replacement         = slot->param1,
                .trigger_mods        = (uint8_t)(slot->param2 >> 8),
                .suppressed_mods     = (uint8_t)(slot->param2 & 0xFF),
                .layers              = (uint16_t)~0,   // 全レイヤー固定
                .negative_mod_mask   = 0,
                .options             = ko_options_default(),
                .custom_action       = NULL,
            };
            remap_override_ptrs[cnt] = &remap_overrides[cnt];
            cnt++;
        }
    }
    remap_override_ptrs[cnt] = NULL;
    key_overrides = remap_override_ptrs;
}
```

### 12.2 Slot レイアウト

```c
// type = 0x03 (Key Override)
{
    type:    0x03,
    flags:   bit0=enabled, bit1-7=reserved,
    param0:  trigger_kc       (反応する元の keycode)
    param1:  replacement_kc   (置き換え後の keycode)
    param2:  (trigger_mods << 8) | suppressed_mods
}
```

### 12.3 全レイヤー固定（layer mask 持たず）

QMK 本家の `key_override_t.layers`（per-layer mask）は **全レイヤー（`~0`）に固定**。slot 8B 内に layer mask を入れる余地がなく、また実用上「全レイヤーで動作」が大多数のため割り切る。

per-layer 制御が必要な場合は、QMK 本家機能（C コード直書き）を使用すること（Tier 3）。

---

## 13. Other Modules

### 13.1 Caps Word（module_id=0x14）

QMK Caps Word を facade。`caps_word_press_user` weak callback を Remap が実装。

#### 設定構造（4B）
```c
struct caps_word_config {
    uint8_t  activation_method;   // 0=off, 1=double_shift, 2=both_shifts, 3=dedicated_key
    uint16_t timeout_ms;          // 自動解除時間（default 5000ms）
    uint8_t  terminator_flags;    // 終端文字 bitmap
};
// terminator_flags:
//   bit0=space, bit1=esc, bit2=enter, bit3=tab,
//   bit4=backspace, bit5=delete, bit6=arrow_keys, bit7=reserved
```

#### Sub-commands
```
0x01  GET_CONFIG()       → 4B
0x02  SET_CONFIG(4B)     → 1B status
```

### 13.2 One Shot（module_id=0x15）

QMK One Shot Modifier / Layer をそのまま使用。設定値を `keymap_config` 等に動的反映。

#### 設定構造（4B）
```c
struct one_shot_config {
    uint16_t timeout_ms;       // 1000ms default
    uint8_t  max_chain;        // 連鎖上限（default 5）
    uint8_t  flags;            // bit0=lock_on_double_tap, bit1=chain_enable
};
```

### 13.3 Tri Layer（module_id=0x16）

QMK の `set_tri_layer_*_layer()` API 経由で動的設定。

#### 設定構造（4B）
```c
struct tri_layer_config {
    uint8_t lower_layer;
    uint8_t upper_layer;
    uint8_t adjust_layer;
    uint8_t flags;            // bit0=enabled
};
```

### 13.4 Repeat Key（module_id=0x17）

QMK Repeat Key / Alt Repeat Key を facade として活用。`REPEAT_KEY_ENABLE` を rules.mk で ON にすると QMK 標準動作が有効化される。Remap モジュールは `flags` ビットによってランタイムでの**機能 ON/OFF を上書き**できる。

#### 設定構造（1B）
```c
struct repeat_key_config {
    uint8_t flags;     // bit0=repeat_enabled, bit1=alt_repeat_enabled
};
```

#### Runtime 挙動

`flags` の各ビットが OFF（0）のときは、Remap が `process_record_remap` 内で対応 keycode（`QK_REPEAT_KEY` / `QK_ALT_REPEAT_KEY`）を**捕捉して no-op 化**する。これにより：
- `rules.mk` で `REPEAT_KEY_ENABLE = yes`（Remap モジュール ON にすれば自動で yes）
- ウェブから `flags = 0x00` で「ランタイム無効」状態を実現
- 再び `flags` を立てれば即座に有効化（再ビルド不要）

つまり「ビルド時に組み込み、ランタイムでマスク」という facade パターン。

### 13.5 Auto Shift（module_id=0x18）

QMK Auto Shift。`get_auto_shifted_key` weak callback を Remap が実装。

#### 設定構造（4B）
```c
struct auto_shift_config {
    uint16_t timeout_ms;     // 175ms default
    uint8_t  scope_flags;    // bit0=letters, bit1=numerics, bit2=special
    uint8_t  reserved;
};
```

Auto Shift Retro モードは **Tier 3** とし、本仕様の対象外。

### 13.6 Mouse Keys（module_id=0x19）

QMK Mouse Keys の **MK_3_SPEED モード限定**でサポート。`mk_*` 設定値を動的反映。

#### 設定構造（8B）
```c
struct mouse_keys_config {
    uint8_t mk_delay;
    uint8_t mk_interval;
    uint8_t mk_max_speed;
    uint8_t mk_time_to_max;
    uint8_t mk_wheel_delay;
    uint8_t mk_wheel_interval;
    uint8_t mk_wheel_max_speed;
    uint8_t mk_wheel_time_to_max;
};
```

ACCELERATED モードは **Tier 3**。

### 13.7 Per-Key Term（module_id=0x1A）

QMK の `get_tapping_term` weak callback を Remap が実装し、slot に合致する keycode のみ独自値を返す。

#### Entry 構造（4B/entry）
```c
typedef struct __attribute__((packed)) {
    uint16_t keycode;          // 対象 QMK keycode（例：LSFT_T(KC_A)）
    uint16_t tapping_term_ms;  // 個別 timer 値
} per_key_term_entry_t;

per_key_term_entry_t entries[REMAP_PER_KEY_TERM_MAX];  // default 8
```

`remap.json` で max_entries をオーバーライド可能。

#### Sub-commands
```
0x01  GET_ENTRY(idx:1B)              → 4B
0x02  SET_ENTRY(idx:1B, data:4B)     → 1B status
0x03  CLEAR_ENTRY(idx:1B)            → 1B status
0x04  GET_MAX_ENTRIES()              → 1B
```

---

## 14. Module: Backlight（module_id=0x20）

単色 LED backlight の制御。VIA `id_qmk_backlight_channel` 相当の機能を Remap protocol で再設計。

### 14.1 Sub-commands

```
0x01  GET_STATE   ()                                → [brightness:1, effect:1]
0x02  SET_STATE   ([brightness:1, effect:1])        → []
```

### 14.2 詳細仕様

#### `GET_STATE` (0x01)
```
Request (32B):
  Byte 0:    0x20 (module_id)
  Byte 1:    0x01 (sub_cmd)
  Byte 2-31: 0x00

Response (32B):
  Byte 0:    0x20
  Byte 1:    status
  Byte 2:    brightness (uint8, 0-255)
  Byte 3:    effect     (uint8, QMK backlight mode 値)
  Byte 4-31: 0x00
```

#### `SET_STATE` (0x02)
```
Request (32B):
  Byte 0:    0x20
  Byte 1:    0x02
  Byte 2:    brightness (uint8)
  Byte 3:    effect     (uint8)
  Byte 4-31: 0x00

Response (32B):
  Byte 0:    0x20
  Byte 1:    status
  Byte 2-31: 0x00
```

### 14.3 値域・実装方針

- `brightness`：0-255（QMK 内部表現と同じ）
- `effect`：QMK 内部 mode 値を**透過**（webapp が `remap.json` 由来のメタデータから有効 mode を知る）
- SET 後の挙動：§3.5 RAM Cache + Save-on-Demand 準拠（RAM cache 即更新で視覚的即時反映、EEPROM commit は遅延）
- 機能未対応キーボード（`BACKLIGHT_ENABLE = no`）：module 自体がリンクされない → webapp は `STATUS_UNKNOWN_MODULE` を受信して backlight UI を非表示にする

---

## 15. Module: RGB Light（module_id=0x21）

RGB Light（WS2812 strip / SK6812 等の under-glow LED）の制御。VIA `id_qmk_rgblight_channel` 相当。

### 15.1 Sub-commands

```
0x01  GET_STATE   ()  → [brightness:1, effect:1, effect_speed:1, hue:1, sat:1]
0x02  SET_STATE   ([brightness:1, effect:1, effect_speed:1, hue:1, sat:1]) → []
```

### 15.2 詳細仕様

#### `GET_STATE` (0x01)
```
Request (32B):
  Byte 0:    0x21
  Byte 1:    0x01
  Byte 2-31: 0x00

Response (32B):
  Byte 0:    0x21
  Byte 1:    status
  Byte 2:    brightness    (uint8, 0-255)
  Byte 3:    effect        (uint8, QMK rgblight mode 値)
  Byte 4:    effect_speed  (uint8, 0-255)
  Byte 5:    hue           (uint8, 0-255)
  Byte 6:    sat           (uint8, 0-255)
  Byte 7-31: 0x00
```

#### `SET_STATE` (0x02)
```
Request (32B):
  Byte 0:    0x21
  Byte 1:    0x02
  Byte 2:    brightness
  Byte 3:    effect
  Byte 4:    effect_speed
  Byte 5:    hue
  Byte 6:    sat
  Byte 7-31: 0x00

Response (32B):
  Byte 0:    0x21
  Byte 1:    status
  Byte 2-31: 0x00
```

### 15.3 値表現

- 色は **HSV** で提供（QMK 内部 API `rgblight_sethsv()` と一致）
- `brightness` は HSV の V (Value) と同義（`rgblight_get_val()` ≈ brightness）。webapp UI 用語「明るさ」と整合させるため `brightness` 名で公開
- SET 後の挙動：§3.5 準拠
- 機能未対応キーボード（`RGBLIGHT_ENABLE = no`）：module 不在で対応

---

## 16. Module: RGB Matrix（module_id=0x22）

RGB Matrix（per-key RGB LED）の制御。VIA `id_qmk_rgb_matrix_channel` 相当。

構造は **§15 RGB Light と完全同一**（5 値：brightness / effect / effect_speed / hue / sat）。

### 16.1 Sub-commands

```
0x01  GET_STATE   ()  → [brightness:1, effect:1, effect_speed:1, hue:1, sat:1]
0x02  SET_STATE   ([brightness:1, effect:1, effect_speed:1, hue:1, sat:1]) → []
```

### 16.2 詳細仕様

`module_id` が `0x22` に変わるのみで、payload 構造・値域・実装方針は §15 と同一。RGB Matrix と RGB Light は QMK で同時に両方有効化するキーボードがあるため別 module として独立。

機能未対応キーボード（`RGB_MATRIX_ENABLE = no`）：module 不在で対応。

---

## 17. Module: Audio（module_id=0x23）

Audio（ビープ音・clicky）の制御。VIA `id_qmk_audio_channel` 相当。

### 17.1 Sub-commands

```
0x01  GET_STATE   ()  → [audio_enable:1, clicky_enable:1]
0x02  SET_STATE   ([audio_enable:1, clicky_enable:1]) → []
```

### 17.2 詳細仕様

#### `GET_STATE` (0x01)
```
Request (32B):
  Byte 0:    0x23
  Byte 1:    0x01
  Byte 2-31: 0x00

Response (32B):
  Byte 0:    0x23
  Byte 1:    status
  Byte 2:    audio_enable  (uint8, 0=disabled, 1=enabled)
  Byte 3:    clicky_enable (uint8, 0=disabled, 1=enabled)
  Byte 4-31: 0x00
```

#### `SET_STATE` (0x02)
```
Request (32B):
  Byte 0:    0x23
  Byte 1:    0x02
  Byte 2:    audio_enable
  Byte 3:    clicky_enable
  Byte 4-31: 0x00

Response (32B):
  Byte 0:    0x23
  Byte 1:    status
  Byte 2-31: 0x00
```

### 17.3 値域・実装方針

- bool 風 uint8（0=disabled, 1=enabled）。HID payload で bool 専用型は意味がないため uint8 で統一
- SET 後の挙動：§3.5 準拠（RAM cache 即更新で聴覚的即時反映、EEPROM commit は遅延）
- 機能未対応キーボード（`AUDIO_ENABLE = no`）：module 不在で対応

---

## 18. Module: LED Matrix（module_id=0x24）

LED Matrix（単色 per-key LED）の制御。VIA `id_qmk_led_matrix_channel` 相当。

### 18.1 Sub-commands

```
0x01  GET_STATE   ()  → [brightness:1, effect:1, effect_speed:1]
0x02  SET_STATE   ([brightness:1, effect:1, effect_speed:1]) → []
```

### 18.2 詳細仕様

#### `GET_STATE` (0x01)
```
Request (32B):
  Byte 0:    0x24
  Byte 1:    0x01
  Byte 2-31: 0x00

Response (32B):
  Byte 0:    0x24
  Byte 1:    status
  Byte 2:    brightness    (uint8, 0-255)
  Byte 3:    effect        (uint8, QMK led_matrix mode 値)
  Byte 4:    effect_speed  (uint8, 0-255)
  Byte 5-31: 0x00
```

#### `SET_STATE` (0x02)
```
Request (32B):
  Byte 0:    0x24
  Byte 1:    0x02
  Byte 2:    brightness
  Byte 3:    effect
  Byte 4:    effect_speed
  Byte 5-31: 0x00

Response (32B):
  Byte 0:    0x24
  Byte 1:    status
  Byte 2-31: 0x00
```

### 18.3 値域・実装方針

- RGB Matrix から hue/sat を抜いた単色版（3 値）
- SET 後の挙動：§3.5 準拠
- 機能未対応キーボード（`LED_MATRIX_ENABLE = no`）：module 不在で対応

---

## 19. Process Record Integration

### 19.1 QMK core patch（パターン D）

Remap モジュールが**自動 dispatch**されるよう、`process_record_quantum` を改造する。

```c
// quantum/quantum.c (Remap fork で改造)
bool process_record_quantum(keyrecord_t *record) {
    uint16_t keycode = get_record_keycode(record, true);

    /* ... 既存 QMK 処理（modifier 判定等）... */

    /* ⭐ Remap モジュール dispatch を挿入 */
    if (!process_record_remap(keycode, record)) return false;

    /* ... 既存の process_record_kb 呼び出しへ続く（無改造）... */
}
```

`process_record_remap()` 内で：
1. ユーザー override 用 weak callback を呼ぶ
2. 各 Remap モジュールへ順次 dispatch
3. すべて通過したら `true` を返し、QMK 標準処理を継続

### 19.2 `process_record_remap_user` weak callback

Remap 機能を利用しつつ独自処理を介入させたいユーザー向けに、専用の weak callback を提供：

```c
// quantum/remap/remap.h
__attribute__((weak)) bool process_record_remap_user(
    uint16_t keycode, keyrecord_t *record) {
    return true;   // デフォルト no-op
}
```

ユーザーは keymap.c で任意に override：

```c
// keymap.c (ユーザー任意)
bool process_record_remap_user(uint16_t keycode, keyrecord_t *record) {
    if (record->event.pressed && keycode == KC_F13) {
        /* 独自処理 */
        return false;   // Remap dispatch をスキップ
    }
    return true;        // 通常通り Remap dispatch へ進む
}
```

### 19.3 Pipeline Diagram

```
matrix scan
  │
  ▼
process_record_quantum             [Remap fork core patch]
  │
  ├─ process_record_remap_user(weak)    [ユーザーが override 可能]
  │
  ├─ Remap モジュール群 dispatch
  │     ├ TD slot keycode → QK_TAP_DANCE 変換
  │     ├ Combo / Override（QMK が scan 方式で自動処理）
  │     ├ Caps Word weak callback
  │     ├ Auto Shift weak callback
  │     └ ...
  │
  ▼
process_record_kb                  [キーボード製作者の領域、完全無改造] ⭐
  │
  ▼
process_record_user                [keymap 製作者の領域、完全無改造] ⭐
```

⭐ キーボード製作者・keymap 製作者ともに **追加コードゼロ**で Remap 機能が動作する。

### 19.4 `core_patches/` 台帳

QMK core への変更点は `core_patches/` 配下の README に台帳化する。各 patch ファイルは：
- 対象ファイル（例: `quantum/quantum.c`）
- 変更行（diff）
- 動機（なぜこの patch が必要か）
- upstream 追従時の注意点

```
core_patches/
  README.md
  001-process-record-quantum-remap-dispatch.patch
      └ quantum/quantum.c の process_record_quantum() に Remap dispatch を挿入
  002-tap-dance-actions-non-const.patch
      └ tap_dance_actions[] を non-const + 外部参照ポインタ化
      └ tap_dance_count も併せて外部参照可能に
  003-key-combos-non-const.patch
      └ key_combos[] を non-const + 外部参照ポインタ化
      └ COMBO_LEN も併せて外部参照可能に
      └ combo_t.keys が指す配列も非 const に変更
  004-key-overrides-non-const.patch
      └ key_overrides[] を non-const + 外部参照ポインタ化
  005-process-record-remap-user-weak-callback.patch
      └ process_record_remap_user の weak callback 宣言を追加
  006-raw-hid-receive-remap-dispatch.patch
      └ raw_hid_receive を Remap dispatcher 一段化（VIA dispatcher 廃止）
  007-eeconfig-retire.patch
      └ QMK eeconfig (32B) を廃止し、Remap Header に置き換え
      └ default_layer / keymap_config / unicode_mode は Header から読む
  ...
```

QMK upstream merge 時は patch を再 apply、コンフリクト発生時は台帳と diff で判断する。

**台帳メンテ規約**：本文（§10.1、§11.1、§12.1 等）で core patch に依存するコードを示す際、対応する `core_patches/NNN-*.patch` 番号を必ずコメント参照する。本文と台帳が乖離しないよう、追加 patch ごとに本仕様書を更新する。

---

## 20. Build-time Configuration

### 20.1 `rules.mk` フィーチャーフラグ

各 Remap モジュールは個別にフィーチャーフラグで ON/OFF：

```makefile
# Remap fork が提供するフラグ
REMAP_ENABLE                   = no   # Remap 全体（必須、yes で残りが有効）
REMAP_MACRO_ENABLE             = no
REMAP_TAP_DANCE_LITE_ENABLE    = no
REMAP_COMBO_LITE_ENABLE        = no
REMAP_KEY_OVERRIDE_LITE_ENABLE = no
REMAP_CAPS_WORD_ENABLE         = no
REMAP_ONE_SHOT_ENABLE          = no
REMAP_TRI_LAYER_ENABLE         = no
REMAP_REPEAT_KEY_ENABLE        = no
REMAP_AUTO_SHIFT_ENABLE        = no
REMAP_MOUSE_KEYS_ENABLE        = no
REMAP_PER_KEY_TERM_ENABLE      = no
```

**デフォルト全 OFF（opt-in）**：キーボード製作者が必要なものだけ意識的に ON にする。EEPROM 容量の予測可能性を最優先。

なお Lighting/Audio module（Backlight / RGB Light / RGB Matrix / Audio / LED Matrix）は QMK の標準フラグ（`BACKLIGHT_ENABLE` 等）に追従し、それぞれ ON のときに自動有効化される。

### 20.2 `remap.json` の `remap` namespace

ビルド時のモジュール sizing パラメータは `remap.json` の `remap` namespace に格納する（§3.7.4 参照）。`keyboard.json` には載せない（生成物なので）。

```json
{
  "remap": {
    "schema_version": 1,
    "slots": {
      "tap_dance":    8,
      "combo":        8,
      "key_override": 8
    },
    "per_key_term": {
      "max_entries": 8
    },
    "macro": {
      "count":         16,
      "buffer_size":  177
    }
  }
}
```

階層構造を採用（フラットではなく）。理由：
- JSON Schema 検証が綺麗
- 将来モジュール追加時の構造維持しやすい
- ウェブ UI が namespace 単位でセクション構築しやすい

`data/schemas/remap.jsonschema`（新設）でこの構造を検証する。`keyboard.jsonschema` は既存のまま手を加えない。

### 20.3 QMK 機能依存の自動制御

Remap モジュール ON にしたら、対応 QMK 機能を**強制 ON**：

```makefile
# Remap fork の build_keyboard.mk が自動追加
ifeq ($(REMAP_TAP_DANCE_LITE_ENABLE), yes)
    TAP_DANCE_ENABLE = yes  # 強制 ON（警告メッセージ出力）
endif

ifeq ($(REMAP_COMBO_LITE_ENABLE), yes)
    COMBO_ENABLE = yes
endif

ifeq ($(REMAP_KEY_OVERRIDE_LITE_ENABLE), yes)
    KEY_OVERRIDE_ENABLE = yes
endif

# ... 他モジュールも同様
```

矛盾設定（例：`REMAP_TAP_DANCE_LITE_ENABLE = yes` かつ `TAP_DANCE_ENABLE = no`）の場合は**警告メッセージを出力しつつ Remap 設定優先でビルド継続**。

### 20.4 デフォルト値（opt-in）

すべてのモジュールデフォルト OFF。キーボード製作者は必要なモジュールのみ：

```makefile
# 例: Lily58 用 rules.mk
REMAP_ENABLE                = yes
REMAP_TAP_DANCE_LITE_ENABLE = yes
REMAP_COMBO_LITE_ENABLE     = yes
# 他は OFF のまま
```

### 20.5 メタデータ生成

`remap.json` を SoT としたビルド時生成フロー。詳細は §3.7 と §5.6 参照。`build_keyboard.mk` が `qmk generate-from-remap` を呼び出し、ビルド毎に `keyboard.json`（QMK ビルド入力）と `remap_metadata.c`（Remap メタデータ）を派生生成する。両者ともに VCS 管理外（`.gitignore` 対象）。

### 20.6 EEPROM 容量計算

Remap fork のビルドシステムが、有効モジュールから自動的に EEPROM 占有量を計算し、`Macro 枠が N B 残っています` 等のビルドログを出力する。容量超過時は警告 or エラー。

---

## 21. Test Strategy

### 21.1 C Unit Tests（googletest）

**facade 部分のみ集中テスト**（QMK 既存機能はテスト対象外）。

```
tests/remap_protocol/         # raw_hid parser、dispatcher
tests/remap_slot_converter/   # custom_slot_t → QMK 構造体変換
tests/remap_eeprom_io/        # EEPROM 抽象化層 + Remap Header 検証
tests/remap_metadata/         # GET_BASIC レスポンス生成
tests/remap_keymap/           # Keymap module（GET/SET_KEYCODE、bulk transfer）
tests/remap_macro/            # Macro module（buffer 操作、null-term 区切り）
tests/remap_system/           # System module（matrix bitmap、bootloader_jump）
tests/remap_lighting/         # Lighting/Audio 5 module の GET/SET_STATE
tests/remap_module_<name>/    # 各 Tier 2 モジュール個別の handler ロジック
```

実行：
```bash
make test:remap_protocol
make test:remap_slot_converter
# ...
```

### 21.2 Python Tests（pytest）

QMK CLI 拡張のテスト：

```
lib/python/qmk/tests/test_remap_metadata_gen.py  # generate-from-remap CLI
lib/python/qmk/tests/test_remap_schema.py        # remap.json schema 検証
lib/python/qmk/tests/test_remap_rules_mk.py      # rules.mk 自動制御
```

実行：`qmk pytest`

### 21.3 Mock 戦略

QMK の既存 mock 機構を流用 + 拡張：
- **raw_hid_send / raw_hid_receive**：テスト用 stub（QMK 既存）
- **EEPROM I/O**：in-memory buffer 方式（QMK 既存）
- **QMK 公開 API**（`tap_dance_actions[]` 等）：unit テストでは mock し、Remap が正しく書き込むことのみ検証

### 21.4 CI 統合

GitHub Actions の workflow を **branch 別に分離**：

```
.github/workflows/
  ci.yml              # QMK upstream（master 用、無改造で維持）
  remap_ci.yml        # Remap 用（remap-develop 用、新規）
```

`remap_ci.yml` の例：
```yaml
on:
  push:
    branches: [remap-develop]
  pull_request:
    branches: [remap-develop]

jobs:
  c_unit_tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: '3.x' }
      - run: pip install -r requirements.txt
      - run: make test:remap_protocol
      - run: make test:remap_slot_converter
      - run: make test:remap_metadata
      - run: make test:remap_keymap
      - run: make test:remap_macro
      - run: make test:remap_system
      - run: make test:remap_lighting
  python_tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: '3.x' }
      - run: pip install -r requirements.txt -r requirements-dev.txt
      - run: qmk pytest
  build_check:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        keyboard: [planck/rev6, lily58/rev1, crkbd/rev1]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: '3.x' }
      - run: pip install -r requirements.txt
      - run: make ${{ matrix.keyboard }}:default
```

`master` branch の `ci.yml` は QMK 本家のものをそのまま継承し、Remap テストを入れない（upstream merge コンフリクト最小化）。

### 21.5 Coverage 目標

明示的 % 目標は設けず、**重要パス（ホットパス）の完全網羅**を方針とする：

| ホットパス | カバレッジ目標 |
|-----------|---------------|
| プロトコル parser | 100% |
| slot 変換ロジック | 100% |
| EEPROM I/O + Header 検証 | 100% |
| §4.7 multi-packet transfer ロジック | 100% |
| エラーハンドリング | 80%+ |
| 初期化・終了処理 | 合理的範囲 |

### 21.6 Manual QA Checklist

実機テストは自動化困難なため手動 QA リストで対応。リリース時の必須確認項目を spec に明記：

```
[ ] 代表 ATmega32U4 ボード（Lily58 等）でビルド・フラッシュ成功
[ ] 代表 RP2040 ボード（KB2040 系）でビルド・フラッシュ成功
[ ] 代表 STM32 ボード（Sofle 系）でビルド・フラッシュ成功
[ ] ウェブアプリから probe 成功 → 全モジュール GET → SET → 動作確認
[ ] Keymap module：GET/SET_KEYCODE → 反映確認、bulk transfer → 全 layer 取得
[ ] Macro module：GET_INFO → SET_BUFFER → macro 再生確認
[ ] System module：BOOTLOADER_JUMP → DFU 移行確認、EEPROM_RESET → 設定リセット確認
[ ] Lighting/Audio：GET/SET_STATE → 視覚的・聴覚的反映確認
[ ] EEPROM 容量超過時のビルド警告動作確認
[ ] EEPROM 整合性ポリシー：magic 不一致時のゼロ初期化動作確認
[ ] Tap Dance Lite：tap / hold / double_tap が期待通り動作
[ ] Combo Lite：2-key 同時押し → 出力 keycode 動作確認
[ ] Key Override Lite：trigger + mods → replacement keycode 動作確認
[ ] Caps Word：activation → 入力 → 終端文字で解除
[ ] One Shot：modifier / layer 一回押し → 連鎖動作
[ ] Tri Layer：lower + upper → adjust layer 切替
[ ] Repeat Key：直前キー繰り返し動作確認
[ ] Auto Shift：長押し → shifted keycode 送信
[ ] Mouse Keys：MK_3_SPEED モード動作確認
[ ] Per-Key Term：個別 keycode の TAPPING_TERM 上書き動作確認
[ ] WSL + VSCode index.lock 競合：retry ループで回復
```

なお「VIA-only キーボードでの VIA fallback 動作」は **Remap Webapp 側の責務**として webapp spec に移管された（Remap Firmware 側 QA 対象外）。

---

## 22. Risk Register

8 つのリスクを優先度付きで列挙。

| ID | リスク | 確度 | 影響 | 優先度 | 緩和策 |
|----|--------|------|------|--------|--------|
| **R1** | Remap fork のメンテコスト（QMK upstream 追従、core patch 維持）| 高 | 中 | 🔴 高 | core patch 最小限化、`core_patches/` 台帳整備、定期 sync スケジュール（四半期） |
| **R2** | ATmega32U4 EEPROM 容量制約（全モジュール ON で 844B、マクロ枠 180B）| 中 | 高 | 🔴 高 | rules.mk opt-in 規約、ウェブ UI で「現在の EEPROM 使用量」可視化 |
| **R3** | 既存 VIA-only キーボードの扱い（1100+ keyboards）| 低 | 低 | 🟢 低 | Webapp が VIA protocol を継続サポートするため、ユーザーは何も変えずに使い続けられる。Remap firmware 化はキーボード単位でオプトイン |
| **R4** | デバッグ困難（QMK 起因 / Remap 起因 / core patch 起因の切り分け）| 中 | 中 | 🟡 中 | Remap 専用ログ機構、core patch 明示的ラベル付け |
| **R5** | 実機テスト自動化の困難 | 中 | 中 | 🟡 中 | QA リスト最小ホットパス限定、コミュニティ協力型ベータテスト |
| **R6** | ウェブアプリ機能パリティ（VIA 機能カバレッジ）| 低 | 高 | 🟡 中 | VIA 全機能 → Remap protocol 対応の表を本仕様（§6-§8、§14-§18）に維持、回帰テスト実施 |
| **R7** | プロトコル進化時の互換性 | 低 | 中 | 🟢 低 | probe で早期 version 判定、incompatible なら分岐提示 |
| **R8** | HID raw 32B 制約による拡張性 | 低 | 低 | 🟢 低 | §4.7 stateless offset-based multi-packet transfer protocol で対応済み（spec で正式定義） |

---

## 23. Open Items

実装フェーズで詰める細部：

1. **GET_BASIC reserved 領域の具体用途**：`reserved_h0` (1B)、`reserved_tail[2]` (2B)、payload 末尾の zero-fill 余り (4B) の合計 7B について、拡張時のフィールド優先順位を定義する
2. **`remap_led_position_t` 構造体の正式定義**：matrix 行列対応、x/y 座標、flags 等の中身（§5.4 で参照のみのため）
3. **`GET_STRING_BY_INDEX(idx)` sub_command の追加可否**：現状は連続読みのみ。任意エントリ参照ができないとウェブ側のリトライ・部分更新で不便
4. **core patch の差分管理ポリシー詳細**：upstream merge 時の手順スクリプト化
5. **Remap モジュール共通の `post_init` フックタイミング**：QMK の `keyboard_post_init_user` 等との順序保証
6. **エラーコード拡張時のポリシー**：将来モジュールが独自エラーを追加する場合の番号空間
7. **`STATUS_NOT_INITIALIZED` の発生条件詳細**：どのモジュール／sub_cmd で返るか、`post_init` 完了前の意味かを明記
8. **メタデータ string table の上限サイズ**：FLASH 圧迫を避ける運用上限の定義
9. **WebHID 接続切断時のリカバリ動作**：RAM cache の dirty 状態をどう扱うか
10. **Combo Plus / Tap Dance Plus の具体実装案**：Tier 2 第二弾としての設計詳細
11. **マルチ JIS / EN keymap 切替対応**：QMK の language layer と Remap layout options の関係
12. **Bluetooth / Wireless キーボード対応**：BMP 等との互換性方針
13. **EEPROM 容量超過時のフォールバック**：ビルド時警告 / エラーの具体閾値
14. **Encoder keycode の EEPROM 配置詳細**：Keymap region 内 or 別領域、サイズ計算式
15. **Remap Header の旧 QMK 互換フィールド粒度**：`steno_mode` / `handedness` 等を Header に追加するか、別領域で対応するか
16. **SET 系の chunked transfer**：現状 `Keymap.SET_BUFFER` / `Macro.SET_BUFFER` のみ。将来他 SET でも必要になった場合の対称適用ポリシー
17. **§4.7 chunk_len = 0 の扱い**：要求 offset が `total_bytes` 以上の場合は `chunk_len = 0` を返すかエラーか

---

## 24. References

- 関連 spec: `2026-04-29-remap-webapp-integration-design.md`（ウェブアプリ側）
- QMK Firmware: <https://github.com/qmk/qmk_firmware>
- VIA Firmware Protocol（参考、Remap は完全分離）: <https://www.caniusevia.com/docs/specification>
- Remap web app: <https://remap-keys.app/>
- WebHID API: <https://developer.mozilla.org/en-US/docs/Web/API/WebHID_API>

---

## Appendix A: Wire Protocol Quick Reference

### Probe
| Direction | Bytes |
|-----------|-------|
| Request   | `[0xFE, 0x00, 'R', 'M', 'A', 'P', '?', 0x00, ...]` |
| Response  | `[0xFE, 0x00, 'R', 'M', 'A', 'P', '!', protocol_version, 0x00, capability_lo, capability_hi, ...]` |

### Standard Dispatch
| Direction | Bytes |
|-----------|-------|
| Request   | `[module_id, sub_cmd, payload[0..29]]` |
| Response  | `[module_id, status, payload[0..29]]` |

### Multi-packet Transfer (§4.7、GET 系)
| Direction | Bytes |
|-----------|-------|
| Request   | `[module_id, sub_cmd, offset_lo, offset_hi, 0x00 ...]` |
| Response  | `[module_id, status, total_lo, total_hi, off_lo, off_hi, chunk_len, data[0..25]]` |

### Multi-packet Transfer (§4.7、SET 系)
| Direction | Bytes |
|-----------|-------|
| Request   | `[module_id, sub_cmd, offset_lo, offset_hi, chunk_len, data[0..24]]` |
| Response  | `[module_id, status, 0x00 ...]` |

### Status Codes
| Hex | Name | Meaning |
|-----|------|---------|
| 0x00 | OK | 成功 |
| 0x01 | UNKNOWN_MODULE | module_id 未対応 |
| 0x02 | UNKNOWN_SUB_CMD | sub_cmd 未対応 |
| 0x03 | INVALID_PAYLOAD_LEN | payload 長不正 |
| 0x04 | INVALID_PARAMS | 範囲外パラメータ |
| 0x05 | EEPROM_WRITE_FAIL | EEPROM 書き込み失敗 |
| 0x06 | NOT_INITIALIZED | モジュール未初期化 |
| 0xFF | INTERNAL_ERROR | 内部エラー |

---

## Appendix B: EEPROM Layout Diagram

ATmega32U4（1 KB EEPROM）、典型 60% キーボード、全モジュール ON：

```
Address  Size  Region
─────────────────────────────────────────────
0x000   16 B  Remap Header (magic / protocol_version / layout_options /
              default_layer / keymap_config / unicode_mode / reserved)
0x010  560 B  Keymap Region (Keymap module 管理、4 layer × 70 key × 2B)
0x240  192 B  Slot Table (TD 64 + Combo 64 + OVR 64)
0x300    4 B  Caps Word config
0x304    4 B  One Shot config
0x308    4 B  Tri Layer config
0x30C    1 B  Repeat Key flags
0x30D    4 B  Auto Shift config
0x311    8 B  Mouse Keys config
0x319   32 B  Per-Key Term table
0x339    2 B  Combo TERM
0x33B    2 B  Backlight state
0x33D    5 B  RGB Light state
0x342    5 B  RGB Matrix state
0x347    3 B  LED Matrix state
0x34A    2 B  Audio state
0x34C  180 B  Macro Region (remap.json で固定、ATmega32U4 残量)
0x3FF        (end)
```

VIA eeconfig (32B) を廃止したことで、旧 spec 試算（Macro 枠 161B）から **+19B** 余裕が生まれている（Header 縮小 +36B − Lighting/Audio 追加 17B = +19B 純増）。実際の Macro 枠は remap.json で確定する。

---

## Appendix C: Module ID & Keycode Range Reservations

> **注**：本表は §4.4 / §6 / §7 / §8 / §9.3 / §10.5 / §11.4 / §13.x / §14-§18 等で個別に定義された ID・帯域の**要約**である。正本は本文側であり、矛盾があれば本文側を優先する。仕様変更時は本文と本 Appendix の両方を同時更新すること。

### Module IDs
```
0x00         (reserved)
0x01         Metadata
0x02         Keymap
0x03         Macro
0x04         (reserved)
0x05         System
0x06-0x0F    (reserved for future global modules)
0x10         Slot Table
0x11         (reserved)
0x12         Combo (補完設定)
0x13         Override (slot のみ)
0x14         Caps Word
0x15         One Shot
0x16         Tri Layer
0x17         Repeat Key
0x18         Auto Shift
0x19         Mouse Keys
0x1A         Per-Key Term
0x1B-0x1F    (reserved for Tier 2 拡張)
0x20         Backlight
0x21         RGB Light
0x22         RGB Matrix
0x23         Audio
0x24         LED Matrix
0x25-0xFD    (reserved for future modules)
0xFE         Probe handshake (special)
0xFF         (reserved)
```

### Keycode Ranges
```
0x7800-0x781F  REMAP_USER_*             (TD 用に最大 32 個まで予約。実有効範囲は
                                        tap_dance_slot_count で通知され、
                                        0x7800 ～ 0x7800+tap_dance_slot_count-1 のみ動作)
0x7820-0x783F  Combo Plus               (Tier 2 将来用、予約のみ)
0x7840-0x785F  Tap Dance Plus           (Tier 2 将来用、予約のみ)
0x7860-0x787F  (reserved)
```

---

## Appendix D: Glossary

| 用語 | 意味 |
|------|------|
| **Remap fork** | QMK Firmware を派生させた本リポジトリ |
| **facade** | 既存実装を変えずにインターフェース層だけ提供する設計パターン |
| **Tier 1/2/3** | 機能の動的設定可否レベル分類 |
| **module_id** | Remap protocol 上の機能ブロック識別子（1B） |
| **sub_cmd** | module 内のオペレーション識別子（1B） |
| **slot** | Tap Dance / Combo / Override で共通利用される 8B 動的設定エントリ |
| **probe** | 接続時に Remap firmware か否かを判別する handshake |
| **core patch** | QMK upstream のソースに対する Remap 固有の変更 |
| **Remap Header** | EEPROM 先頭 16B に配置する Remap 固有の制御領域（VIA eeconfig の置き換え） |
| **Multi-packet Transfer** | §4.7 で定義する stateless offset-based の chunked データ転送プロトコル |
| **PROGMEM** | AVR の FLASH 領域配置指定（プログラムメモリ） |
| **Q6.2 fixed-point** | 6 整数 + 2 小数ビットの固定小数点表現（KLE 座標用） |

---

## 改訂履歴

| 日付 | 変更点 |
|------|--------|
| 2026-04-29 | 初版（Section 1-4 確定） |
| 2026-05-01 | Section 5-15 追加。Section 7 で根本的設計転換（独自実装 → QMK facade）。 |
| 2026-05-02 | VIA protocol 完全置換決定に伴う大改修。§4.7 multi-packet transfer 新設、§3.3-§3.6 EEPROM Layout 再構築（VIA eeconfig 廃止 → Remap Header 統合）、§5.1 sub_cmd 再採番（11→7 個）、§5.4 GET_ENCODER_INFO 廃止。新 module 群追加：Keymap (§6)、Macro (§7)、System (§8)、Backlight/RGB Light/RGB Matrix/Audio/LED Matrix (§14-§18)。既存 §6-§16 を §9-§24 に renumbering。R3 / R8 再評価。Open Items に encoder 配置・SET chunked policy 等を追加。 |
| 2026-05-08 | `remap.json` を SoT とするソース形式設計を §3.7 に新設。Hybrid Option C 構造（QMK passthrough + VIA passthrough + `remap` namespace）、`layouts.options` 構造化（boolean / enum）、`layouts.keymap` の encoder VIA passthrough、v1 機能スコープ（A1/A2/A3/A6/B1/B2/C1/C3/D1/D2/D3/E1）、個別キーボード移行ツールを定義。§5.6 build-time generation を `qmk generate-from-remap` に改名し `keyboard.json` + `remap_metadata.c` の同時生成フローへ更新。§20.2 を `remap.json` の `remap` namespace に再定位（`keyboard.json` 拡張から離脱）。spec 内の `keyboard.json で固定/設定` 系記述を `remap.json` 由来に統一。 |
| 2026-05-08 (2) | info.json 由来コンテンツのファームウェア保持設計を充実化。§5.4 `remap_layout_key_t` を 8B → **16B 統合構造**に拡張（color RGB / encoder_id / layout_opt 関連付け / Q8.8 rotation を per-key で保持）。§5.5 を State / Metadata の 2 層構成に再構築：`remap_layout_option_t` (4B/option, PROGMEM) を新設し、`GET_LAYOUT_OPTIONS_META` (0x08, §4.7 chunked) を追加。既存 `GET/SET_LAYOUT_OPTIONS` を `_STATE` サフィックス付きに改名して責務を明確化。bit field エンコーディング規約（boolean=1bit, enum=ceil(log2(N))bit）を明記。 |
