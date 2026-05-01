# Remap Firmware Extensions Design

## 文書情報

- **初版日**: 2026-04-29
- **改訂日**: 2026-05-01
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

### 1.3 Out of Scope

本仕様の対象外（Tier 3 = C コード直書きで利用）：

- Tap Dance / Combo の **callback 関数ポインタ機能**（C 関数ポインタは EEPROM 保存不可）
- **Auto Shift Retro モード**（複雑度過剰）
- **Mouse Keys ACCELERATED モード**（QMK 公式推奨外）
- **16-key 以上の Combo**（slot 8B 制約、将来 "Combo Plus" として keycode 帯予約のみ）
- **3-tap 以上の Tap Dance**（slot 8B 制約、将来 "Tap Dance Plus" として予約のみ）
- **Override の per-layer mask**（全レイヤー固定で割り切り）

---

## 2. Core Design Philosophy

### 2.1 Remap as QMK Fork

Remap Firmware は QMK Firmware の **fork**（`upstream/master` をベースに `remap-develop` ブランチで派生）。これにより QMK の core 部分（`quantum/quantum.c` 等）を**直接改造可能**であり、Remap モジュールを QMK の標準 dispatch パイプラインに組み込める。

**この設計の利点**：
- キーボード製作者・keymap 製作者が**追加コードを書かずに** Remap 機能が自動有効化
- 既存 QMK 機能の挙動を最大限尊重しつつ、動的設定層を上乗せ

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
| **Tier 1** | 既存の Remap 拡張（dynamic keymap 等）、改修最小 | dynamic_keymap、Macro |
| **Tier 2** | 本仕様で新規追加するモジュール群 | Tap Dance Lite、Combo Lite、他 |
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
    { 0x10, remap_slot_table_handler,   remap_slot_table_init },
#ifdef REMAP_CAPS_WORD_ENABLE
    { 0x14, remap_caps_word_handler,    remap_caps_word_init },
#endif
    // ... 他モジュール
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
| **B. runtime API 注入** | QMK 公開 API を呼び出して設定変更 | `set_tri_layer_lower_layer()` |
| **C. process_record** | キー入力イベントに介入 | TD slot keycode → QK_TAP_DANCE 変換 |
| **D. QMK core patch** | QMK 内部のソースを直接改造 | `process_record_quantum`、`tap_dance_actions[]` 外部参照化 |

### 3.3 EEPROM Region Layout

ATmega32U4（1 KB EEPROM）想定の典型レイアウト：

```
0x000 ─┬─────────────────────┐
       │ QMK eeconfig         │   32 B  (QMK 既存)
0x020 ─┼─────────────────────┤
       │ dynamic_keymap       │  560 B  (Tier 1 既存、4 layer × 70 key × 2B 想定)
0x250 ─┼─────────────────────┤
       │ Remap Region         │
       │  ├ meta region       │  ~20 B  (将来拡張領域含む)
       │  ├ slot table        │  192 B  (TD 64 + Combo 64 + OVR 64)
       │  ├ Caps Word config  │   4 B
       │  ├ One Shot config   │   4 B
       │  ├ Tri Layer config  │   4 B
       │  ├ Repeat Key flags  │   1 B
       │  ├ Auto Shift config │   4 B
       │  ├ Mouse Keys config │   8 B
       │  ├ Per-Key Term tbl  │  32 B  (default 8 entries × 4B)
       │  └ Combo TERM        │   2 B
0x363 ─┼─────────────────────┤
       │ Macro region         │  161 B (残り全部)
0x3FF ─┴─────────────────────┘
```

**消費試算（典型 60% キーボード、全モジュール ON、slot 配分 TD=Combo=OVR=8 既定）**：
```
QMK eeconfig:        32 B
dynamic_keymap:     560 B
Remap meta region:   20 B
Remap slot table:   192 B  (TD 64 + Combo 64 + OVR 64)
Remap modules:       59 B
─────────────────────────
合計:               863 B / 1024 B
Macro 枠残:         161 B
```

**注**：slot 配分を `keyboard.json` で変更すると slot table サイズと Macro 枠が連動して伸縮する。
たとえば slot 合計を論理上限の 32 まで拡張すると slot table は 256 B（+64 B）となり、
Macro 枠は 97 B（−64 B）に縮小される。逆に slot 配分を絞れば Macro 枠を増やせる。

### 3.4 RAM Cache + Save-on-Demand

EEPROM の書き込み回数制限（10 万回 / cell）に対応するため、Remap モジュールは：

- **RAM 上にキャッシュ**を持ち、SET 時はまず RAM のみ更新
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

#### 初回起動・EEPROM 整合性ポリシー

新規ファーム書き込み直後の EEPROM は出荷時 `0xFF` で埋まっているか、旧版データが残存している場合がある。Remap モジュールはこの状態を検知し、安全な初期状態にロールバックする。

**検知メカニズム**：
- Remap meta region の先頭 4B にマジックナンバー（例: `0x52, 0x4D, 0x50, 0x01` = ASCII `'R','M','P'` + `protocol_version`）を格納
- 起動時に EEPROM → RAM ロード前に検証
- 不一致、`0xFF` 連続、または `protocol_version` 不一致の場合、**Remap region 全体をゼロ初期化** + マジックナンバー書き込み

**ゼロ初期化の意味**：
- 全 slot の `type = 0x00` (unused) → ファーム側 no-op で安全
- 全モジュール設定値が 0 → QMK 標準デフォルト値で動作（Remap モジュールは何も override しない）
- ユーザーがウェブから設定する前の挙動は QMK 既存挙動に等価

**`protocol_version` 不一致時の扱い**：
- 異なる version で書かれたデータ構造を誤って解釈するリスクを避けるためゼロ初期化
- ウェブ UI は再接続時に「ファーム更新により設定がリセットされました」を表示する責務を持つ

このポリシーにより、初回起動・EEPROM 破損・ファーム version up のいずれも追加ロジックなく安全に処理される。

### 3.5 dynamic_keymap.c との共存

Remap Region は既存の `dynamic_keymap` 領域の**後ろ**に配置。`DYNAMIC_KEYMAP_MACRO_EEPROM_MAX_ADDR` を override してマクロ枠を Remap Region の後ろに押し込む：

```c
// keyboard 側 config.h（または Remap fork が定義）
#define DYNAMIC_KEYMAP_MACRO_EEPROM_MAX_ADDR 0x3FF
#define DYNAMIC_KEYMAP_MACRO_EEPROM_ADDR     (REMAP_REGION_END)
```

これにより、既存の dynamic_keymap 機能（VIA 互換）はそのまま動作しつつ、Remap モジュールが領域を占有できる。

### 3.6 累積オフセットマクロ（`#ifdef` 駆動）

各モジュールが ON/OFF に応じて占有サイズが変わる。これを `#ifdef` 累積マクロで表現：

```c
// quantum/remap/remap_storage_layout.h

#define REMAP_OFFSET_HEADER       (0x250)
#define REMAP_SIZE_HEADER         (20)

#define REMAP_OFFSET_SLOT_TABLE   (REMAP_OFFSET_HEADER + REMAP_SIZE_HEADER)
#define REMAP_SIZE_SLOT_TABLE \
    ((REMAP_TD_SLOT_COUNT + REMAP_COMBO_SLOT_COUNT + REMAP_OVR_SLOT_COUNT) * 8)
// 論理上限は 32 slots × 8B = 256B（keyboard.json の合計 ≤ 32 制約）。
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

// ... 以下同様に累積
```

無効化されたモジュールは 0 バイト消費。マクロ枠を最大化できる。

---

## 4. Wire Protocol Specification（正本）

### 4.1 HID Raw 32B Format

通信は WebHID API（`usagePage = 0xff60`、`usage = 0x61`）を使用し、すべて **32 バイト固定長パケット**で行う。

```
Byte  0:    module_id     (リクエスト/レスポンス共通)
Byte  1:    sub_cmd       (リクエスト) / status (レスポンス)
Byte 2-31:  payload       (最大 30 バイト)
```

**特殊**: `module_id == 0xFE` は probe handshake 用に予約（VIA との衝突回避のため）。

### 4.2 Probe Handshake

ウェブアプリは接続時に最初に probe を送信し、Remap firmware か VIA firmware かを判別する。

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
Byte  9-10: capability_flags          (uint16_t LE、将来拡張用)
Byte 11-31: reserved                   (0x00)
```

**識別ロジック**：
- VIA firmware は `module_id=0xFE` を不明コマンド扱いし、`0xFF` 応答する → Remap と区別可能
- Remap firmware は magic `"RMAP!"` を返す → ウェブは即座に認識

**protocol_version の役割**：
- ウェブアプリが互換性判定を**通信の最初期**に行うために存在
- 整数連番（1, 2, 3, …）で運用。semver の minor/patch のような小数構造は持たない（プロトコルは互換 / 非互換の二値判断のため）
- 例：ウェブが `protocol_version=2` までしか知らないが、ファームが `3` を返した → 「未対応バージョンです」と即エラー表示

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

### 4.4 Module ID Allocation Map

```
0x00         (reserved)
0x01         Metadata
0x02-0x0F    (reserved for future global modules)
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
0x1B-0xFD    (reserved for future modules)
0xFE         Probe handshake（予約、通常 dispatch 対象外）
0xFF         (reserved、VIA との衝突回避用に未使用)
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

---

## 5. Module: Metadata（module_id=0x01）

VIA JSON 撤廃の中核。キーボードのあらゆるメタデータを動的応答する。

### 5.1 Sub-commands

```
0x01  GET_BASIC                     (基本情報 26B)
0x02  GET_STRING_TABLE_BEGIN        (string table 取得開始)
0x03  GET_STRING_TABLE_CONTINUE     (続き)
0x04  GET_LAYOUTS_BEGIN             (KLE レイアウト)
0x05  GET_LAYOUTS_CONTINUE
0x06  GET_LAYOUT_OPTIONS            (レイアウトオプション)
0x07  GET_LED_POSITIONS_BEGIN       (RGB matrix 等の LED 位置)
0x08  GET_LED_POSITIONS_CONTINUE
0x09  GET_CUSTOM_KEYCODES_BEGIN     (カスタムキーコード一覧)
0x0A  GET_CUSTOM_KEYCODES_CONTINUE
0x0B  GET_ENCODER_INFO              (エンコーダ情報)
```

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
    uint16_t firmware_version;       // 2B（keyboard.json 由来、表示用）

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

### 5.3 String Table（UTF-8）

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

最大 255 バイト/エントリ。1 パケット 30B 内に収まらない場合は CONTINUE で分割。

### 5.4 Layouts / LED Positions / Encoders

#### KLE Layout の表現

各キー位置は **Q6.2 固定小数点**（`uint8_t × 0.25u`、範囲 0-63.75u）でエンコード：

```c
typedef struct __attribute__((packed)) {
    uint8_t  matrix_row;    // 1B
    uint8_t  matrix_col;    // 1B
    uint8_t  x_q6_2;        // 1B：x 座標（0.25u 単位）
    uint8_t  y_q6_2;        // 1B：y 座標
    uint8_t  w_q6_2;        // 1B：幅
    uint8_t  h_q6_2;        // 1B：高さ
    uint8_t  flags;         // 1B：bit0=is_decal、bit1-7=reserved
    uint8_t  rotation;      // 1B：将来拡張用、現状 0 固定
} remap_layout_key_t;       // 8B/key
```

8 B/key × ~70 key（典型 60%）= ~560 B → 1 パケット 30B 内に 3 key 収まる → ~24 パケットで全 layout 取得。

#### LED Positions

RGB matrix / LED matrix が ON の場合のみ。同様に 8B/LED 構造で取得。

#### Encoder Info

```c
typedef struct __attribute__((packed)) {
    uint8_t  pin_a;
    uint8_t  pin_b;
    uint8_t  resolution;
    uint8_t  flags;          // bit0=clockwise_keycode_assigned, etc.
} remap_encoder_t;           // 4B/encoder
```

### 5.5 Layout Options

VIA 互換性のため、layout options は **VIA 既存の eeconfig 領域**（バイト 32-35）を流用する。これにより VIA / Remap いずれでも同じ option 値を読み書き可能。

### 5.6 Build-time Generation

メタデータの大半（layout、string table 等）は keyboard.json から自動生成される。

#### `qmk generate-remap-metadata` CLI

QMK CLI の拡張サブコマンド。`keyboard.json` を入力に C ソースを生成：

```bash
qmk generate-remap-metadata --keyboard <kb> -o build/remap_metadata.c
```

生成内容（例）：
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

`build_keyboard.mk` に組み込み、ビルド毎に自動生成：

```makefile
$(BUILD_DIR)/remap_metadata.c: $(KEYBOARD_PATH)/keyboard.json
	$(QMK) generate-remap-metadata --keyboard $(KEYBOARD) -o $@
```

PROGMEM 消費は典型 60% キーボードで約 1.1 KB FLASH。

---

## 6. Module: Slot Table（module_id=0x10）

Goal 2 の中核。Tap Dance / Combo / Key Override の動的設定を統一スロット構造で管理する。

### 6.1 `custom_slot_t` 統一構造（8B）

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

### 6.2 モジュール別配分（β 配分方式）

統一プールではなく、モジュール別に**ビルド時固定の専用配列**を持つ：

```c
custom_slot_t td_slots[REMAP_TD_SLOT_COUNT];      // default 8
custom_slot_t combo_slots[REMAP_COMBO_SLOT_COUNT]; // default 8
custom_slot_t ovr_slots[REMAP_OVR_SLOT_COUNT];    // default 8
// 合計 ≤ 32（slot 数の論理上限）
```

**配分のオーバーライド**は `keyboard.json` で行う：

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

### 6.3 keycode マッピング（動的予約）

Tap Dance のみ keymap 配置が必要なため、**TD slot 数に応じて keycode 帯が動的にサイズ変動**：

```
0x7800 ～ 0x7800 + REMAP_TD_SLOT_COUNT - 1   REMAP_USER_0..N (TD slot index 参照)
0x7820 ～ 0x783F                              (Combo Plus 将来用、予約のみ)
```

ウェブアプリは GET_BASIC レスポンスの `tap_dance_slot_count` で TD keycode 範囲を動的判定する。

**Combo / Override は scan 方式**で keymap 配置不要のため、keycode 帯を消費しない。

### 6.4 Sub-commands

```
0x01  GET_SLOT(type:1B, idx:1B)               → 8B slot data
0x02  SET_SLOT(type:1B, idx:1B, data:8B)      → 1B status
0x03  CLEAR_SLOT(type:1B, idx:1B)             → 1B status (type=0x00 にリセット)
0x04  COMMIT_TO_EEPROM()                      → 1B status (RAM cache → EEPROM)
```

- `GET_SLOT` は逐次取得（24 slot で約 500ms、起動時 1 回限定なので許容）
- バリデーションは**ファーム側で slot_index 範囲チェックのみ**。型・keycode 整合性はウェブ UI 側で事前検証する責務。

### 6.5 空 slot の扱い

ユーザーが keymap に `REMAP_USER_5` を配置したが TD slot 5 の `type == 0x00 (unused)` の場合：
- **ファーム側**: 完全に no-op（押しても何も起きない）
- **ウェブ側**: UI で「slot 未定義」を警告表示

ファームに警告 LED 等のロジックは持たせない（FLASH 節約）。

---

## 7. Module: Tap Dance Lite（slot type=0x01）

QMK Tap Dance を facade として活用する設計。

### 7.1 QMK facade 実装方針

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
            // （詳細は 7.4 参照）
        }
    }
    tap_dance_actions = remap_td_actions;
    tap_dance_count   = REMAP_TD_SLOT_COUNT;
}
```

### 7.2 Slot レイアウト

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

### 7.3 flags ビットレイアウト

QMK 標準の挙動制御に対応するビット：

| bit | 名前 | QMK 対応機能 |
|-----|------|------|
| 0 | `enabled` | slot を有効化 |
| 1 | `permissive_hold` | modifier を即発動 |
| 2 | `hold_on_other_key_press` | 他キー押下で hold 確定 |
| 3 | `quick_tap_term` | 高速連打で tap 確定 |
| 4-7 | reserved | 将来拡張用 |

これらは QMK の per-key callback（`get_permissive_hold` 等）を Remap が実装し、slot の flags を読んで応答する形で実現する。

### 7.4 制約

- **callback 関数ポインタ不可**：QMK の `ACTION_TAP_DANCE_FN` 系統は使えない（C 関数ポインタ EEPROM 保存不可）
- **2-tap + hold まで**：slot 8B で表現できる action は 3 種（tap / hold / double_tap）が物理限界
- **TAPPING_TERM はグローバル固定**：slot 別調整は持たない（必要なら Per-Key Term モジュール経由）

### 7.5 Future: Tap Dance Plus

3-tap 以上を将来サポートする場合に備えて：
- keycode 帯 `0x7840-0x785F` を**予約のみ**（実装は Tier 2 第二弾）
- 別構造体 `remap_tap_dance_extended_t`（16B 等）を別 EEPROM 領域に確保する設計余地

---

## 8. Module: Combo Lite（slot type=0x02）

QMK Combo を facade として活用。

### 8.1 QMK combo_t facade

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

### 8.2 Slot レイアウト（2-key trigger 限定）

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

### 8.3 補完設定（module_id=0x12）

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

### 8.4 Future: Combo Plus

n-key trigger（3-16 keys）対応は将来の Tier 2 拡張：
- keycode 帯 `0x7820-0x783F` を予約のみ
- 別構造体（可変長 combo_ex_table）を別 EEPROM 領域に確保する設計余地

---

## 9. Module: Key Override Lite（slot type=0x03）

QMK Key Override を facade として活用。

### 9.1 QMK key_override_t facade

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

### 9.2 Slot レイアウト

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

### 9.3 全レイヤー固定（layer mask 持たず）

QMK 本家の `key_override_t.layers`（per-layer mask）は **全レイヤー（`~0`）に固定**。slot 8B 内に layer mask を入れる余地がなく、また実用上「全レイヤーで動作」が大多数のため割り切る。

per-layer 制御が必要な場合は、QMK 本家機能（C コード直書き）を使用すること（Tier 3）。

---

## 10. Other Modules

### 10.1 Caps Word（module_id=0x14）

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

### 10.2 One Shot（module_id=0x15）

QMK One Shot Modifier / Layer をそのまま使用。設定値を `keymap_config` 等に動的反映。

#### 設定構造（4B）
```c
struct one_shot_config {
    uint16_t timeout_ms;       // 1000ms default
    uint8_t  max_chain;        // 連鎖上限（default 5）
    uint8_t  flags;            // bit0=lock_on_double_tap, bit1=chain_enable
};
```

### 10.3 Tri Layer（module_id=0x16）

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

### 10.4 Repeat Key（module_id=0x17）

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

### 10.5 Auto Shift（module_id=0x18）

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

### 10.6 Mouse Keys（module_id=0x19）

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

### 10.7 Per-Key Term（module_id=0x1A）

QMK の `get_tapping_term` weak callback を Remap が実装し、slot に合致する keycode のみ独自値を返す。

#### Entry 構造（4B/entry）
```c
typedef struct __attribute__((packed)) {
    uint16_t keycode;          // 対象 QMK keycode（例：LSFT_T(KC_A)）
    uint16_t tapping_term_ms;  // 個別 timer 値
} per_key_term_entry_t;

per_key_term_entry_t entries[REMAP_PER_KEY_TERM_MAX];  // default 8
```

`keyboard.json` で max_entries をオーバーライド可能。

#### Sub-commands
```
0x01  GET_ENTRY(idx:1B)              → 4B
0x02  SET_ENTRY(idx:1B, data:4B)     → 1B status
0x03  CLEAR_ENTRY(idx:1B)            → 1B status
0x04  GET_MAX_ENTRIES()              → 1B
```

---

## 11. Process Record Integration

### 11.1 QMK core patch（パターン D）

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

### 11.2 `process_record_remap_user` weak callback

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

### 11.3 Pipeline Diagram

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

### 11.4 `core_patches/` 台帳

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
  ...
```

QMK upstream merge 時は patch を再 apply、コンフリクト発生時は台帳と diff で判断する。

**台帳メンテ規約**：本文（§7.1、§8.1、§9.1 等）で core patch に依存するコードを示す際、対応する `core_patches/NNN-*.patch` 番号を必ずコメント参照する。本文と台帳が乖離しないよう、追加 patch ごとに本仕様書を更新する。

---

## 12. Build-time Configuration

### 12.1 `rules.mk` フィーチャーフラグ

各 Remap モジュールは個別にフィーチャーフラグで ON/OFF：

```makefile
# Remap fork が提供するフラグ
REMAP_ENABLE                   = no   # Remap 全体（必須、yes で残りが有効）
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

### 12.2 `keyboard.json` schema 拡張

`data/schemas/keyboard.jsonschema` に `remap` namespace を追加：

```json
{
  "remap": {
    "slots": {
      "tap_dance":    8,
      "combo":        8,
      "key_override": 8
    },
    "per_key_term": {
      "max_entries": 8
    }
  }
}
```

階層構造を採用（フラットではなく）。理由：
- JSON Schema 検証が綺麗
- 将来モジュール追加時の構造維持しやすい
- ウェブ UI が namespace 単位でセクション構築しやすい

### 12.3 QMK 機能依存の自動制御

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

### 12.4 デフォルト値（opt-in）

すべてのモジュールデフォルト OFF。キーボード製作者は必要なモジュールのみ：

```makefile
# 例: Lily58 用 rules.mk
REMAP_ENABLE                = yes
REMAP_TAP_DANCE_LITE_ENABLE = yes
REMAP_COMBO_LITE_ENABLE     = yes
# 他は OFF のまま
```

### 12.5 メタデータ生成

Section 5.6 参照。`build_keyboard.mk` に組み込み、ビルド毎自動生成。

### 12.6 EEPROM 容量計算

Remap fork のビルドシステムが、有効モジュールから自動的に EEPROM 占有量を計算し、`Macro 枠が N B 残っています` 等のビルドログを出力する。容量超過時は警告 or エラー。

---

## 13. Test Strategy

### 13.1 C Unit Tests（googletest）

**facade 部分のみ集中テスト**（QMK 既存機能はテスト対象外）。

```
tests/remap_protocol/         # raw_hid parser、dispatcher
tests/remap_slot_converter/   # custom_slot_t → QMK 構造体変換
tests/remap_eeprom_io/        # EEPROM 抽象化層
tests/remap_metadata/         # GET_BASIC レスポンス生成
tests/remap_module_<name>/    # 各モジュール個別の handler ロジック
```

実行：
```bash
make test:remap_protocol
make test:remap_slot_converter
# ...
```

### 13.2 Python Tests（pytest）

QMK CLI 拡張のテスト：

```
lib/python/qmk/tests/test_remap_metadata_gen.py  # generate-remap-metadata CLI
lib/python/qmk/tests/test_remap_schema.py        # keyboard.json 拡張 schema 検証
lib/python/qmk/tests/test_remap_rules_mk.py      # rules.mk 自動制御
```

実行：`qmk pytest`

### 13.3 Mock 戦略

QMK の既存 mock 機構を流用 + 拡張：
- **raw_hid_send / raw_hid_receive**：テスト用 stub（QMK 既存）
- **EEPROM I/O**：in-memory buffer 方式（QMK 既存）
- **QMK 公開 API**（`tap_dance_actions[]` 等）：unit テストでは mock し、Remap が正しく書き込むことのみ検証

### 13.4 CI 統合

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

### 13.5 Coverage 目標

明示的 % 目標は設けず、**重要パス（ホットパス）の完全網羅**を方針とする：

| ホットパス | カバレッジ目標 |
|-----------|---------------|
| プロトコル parser | 100% |
| slot 変換ロジック | 100% |
| EEPROM I/O | 100% |
| エラーハンドリング | 80%+ |
| 初期化・終了処理 | 合理的範囲 |

### 13.6 Manual QA Checklist

実機テストは自動化困難なため手動 QA リストで対応。リリース時の必須確認項目を spec に明記：

```
[ ] 代表 ATmega32U4 ボード（Lily58 等）でビルド・フラッシュ成功
[ ] 代表 RP2040 ボード（KB2040 系）でビルド・フラッシュ成功
[ ] 代表 STM32 ボード（Sofle 系）でビルド・フラッシュ成功
[ ] ウェブアプリから probe 成功 → 全モジュール GET → SET → 動作確認
[ ] EEPROM 容量超過時のビルド警告動作確認
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
[ ] VIA fallback：Remap 非対応キーボードでウェブ接続時に VIA 互換動作
[ ] WSL + VSCode index.lock 競合：retry ループで回復
```

---

## 14. Risk Register

8 つのリスクを優先度付きで列挙。

| ID | リスク | 確度 | 影響 | 優先度 | 緩和策 |
|----|--------|------|------|--------|--------|
| **R1** | Remap fork のメンテコスト（QMK upstream 追従、core patch 維持）| 高 | 中 | 🔴 高 | core patch 最小限化、`core_patches/` 台帳整備、定期 sync スケジュール（四半期） |
| **R2** | ATmega32U4 EEPROM 容量制約（全モジュール ON で 863B、マクロ枠 161B）| 中 | 高 | 🔴 高 | rules.mk opt-in 規約、ウェブ UI で「現在の EEPROM 使用量」可視化 |
| **R3** | 既存 VIA-only キーボードの移行コスト（1100+ keyboards）| 高 | 中 | 🔴 高 | 移行ガイド整備、優先度の高いキーボード（Lily58 / Corne 等）から段階対応 |
| **R4** | デバッグ困難（QMK 起因 / Remap 起因 / core patch 起因の切り分け）| 中 | 中 | 🟡 中 | Remap 専用ログ機構、core patch 明示的ラベル付け |
| **R5** | 実機テスト自動化の困難 | 中 | 中 | 🟡 中 | QA リスト最小ホットパス限定、コミュニティ協力型ベータテスト |
| **R6** | ウェブアプリ機能パリティ（VIA との同等性確認）| 低 | 高 | 🟡 中 | 機能比較表を spec doc に維持、回帰テスト実施 |
| **R7** | プロトコル進化時の互換性 | 低 | 中 | 🟢 低 | probe で早期 version 判定、incompatible なら分岐提示 |
| **R8** | HID raw 32B 制約による拡張性 | 低 | 低 | 🟢 低 | ストリーム送信機構（GET_xxx_BEGIN/CONTINUE/END）を必要時追加 |

---

## 15. Open Items

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

---

## 16. References

- 関連 spec: `2026-04-29-remap-webapp-integration-design.md`（ウェブアプリ側）
- QMK Firmware: <https://github.com/qmk/qmk_firmware>
- VIA Firmware Protocol: <https://www.caniusevia.com/docs/specification>
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
0x000   32 B  QMK eeconfig
0x020  560 B  dynamic_keymap (4 layer × 70 key × 2B)
0x250   20 B  Remap meta region
0x264  192 B  Remap slot table (TD 64 + Combo 64 + OVR 64)
0x324    4 B  Caps Word config
0x328    4 B  One Shot config
0x32C    4 B  Tri Layer config
0x330    1 B  Repeat Key flags
0x331    4 B  Auto Shift config
0x335    8 B  Mouse Keys config
0x33D   32 B  Per-Key Term table
0x35D    2 B  Combo TERM
0x35F  161 B  Macro region
0x3FF        (end)
```

---

## Appendix C: Module ID & Keycode Range Reservations

> **注**：本表は §4.4 / §6.3 / §7.5 / §8.4 / §10.x 等で個別に定義された ID・帯域の**要約**である。正本は本文側であり、矛盾があれば本文側を優先する。仕様変更時は本文と本 Appendix の両方を同時更新すること。

### Module IDs
```
0x00         (reserved)
0x01         Metadata
0x02-0x0F    (reserved for future global modules)
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
0x1B-0xFD    (reserved for future modules)
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
| **probe** | 接続時に Remap firmware か VIA firmware かを判別する handshake |
| **core patch** | QMK upstream のソースに対する Remap 固有の変更 |
| **eeconfig** | QMK の EEPROM 内設定領域（バイト 0-31）|
| **dynamic_keymap** | VIA 互換のランタイム keymap 変更領域 |
| **PROGMEM** | AVR の FLASH 領域配置指定（プログラムメモリ） |
| **Q6.2 fixed-point** | 6 整数 + 2 小数ビットの固定小数点表現（KLE 座標用） |

---

## 改訂履歴

| 日付 | 変更点 |
|------|--------|
| 2026-04-29 | 初版（Section 1-4 確定） |
| 2026-05-01 | Section 5-15 追加。Section 7 で根本的設計転換（独自実装 → QMK facade）。 |
