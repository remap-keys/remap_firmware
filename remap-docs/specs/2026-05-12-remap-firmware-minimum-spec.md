# Remap Firmware Minimum Spec (v0)

## 文書情報

- **初版日**: 2026-05-12
- **ステータス**: 設計フェーズ
- **対象**: Remap Firmware（本リポジトリ）
- **ブランチ**: `remap-develop`（QMK 0.32.12 ベース）
- **関連 Spec doc**: `2026-04-29-remap-firmware-extensions-design.md`（v1 以降の包括設計）

---

## 1. はじめに

### 1.1 目的

現状の Remap UX では、ユーザーがキーボードをカスタマイズするには、**開発者が事前に** Remap (Firestore) にキーボード定義を登録しておく必要がある。

本仕様は、**この事前登録ステップを不要化**し、Remap にキーボードを接続するだけでカスタマイズ可能にすることを目的とする（プラグアンドプレイ）。

実現方法は単純で、Remap が現在 Firestore に保持しているキーボード定義情報（VIA info.json、詳細は §3）を firmware の Flash に焼き込み、HID 経由で Remap web app が取得する。これにより Firestore lookup なしで Remap が `KeyboardDefinition` を構築できる。

### 1.2 非目的（v0 で扱わないこと）

本仕様 v0 では以下を意図的にスコープ外とする：

- **Remap カタログ情報**（description, images, stores, websiteUrl 等）のファームウェア埋め込み
- **Remap 独自情報用ネームスペース**（`remap.*` namespace, `schema_version` 等）
- **レイアウトオプションの拡張メタデータ**（boolean/enum type 情報、ラベル翻訳等）
- **per-key の Remap 拡張情報**（色、回転、エンコーダ詳細等）
- **動的設定機能**（Tap-Hold, Combo, Tap Dance 等の Remap web app からの設定変更）
- **VIA info.json のフォーマット拡張**（VIA 仕様をそのまま使用）
- **マイグレーションツール**（既存 info.json 流用のため不要）

### 1.3 既存 extensions-design spec との関係

`2026-04-29-remap-firmware-extensions-design.md` は本仕様より広範な v1+ 機能を含む包括設計。本仕様で除外した項目は将来同 spec の内容を取り込んで段階的に追加する想定。本仕様は**最小の動作する仕様**を目指す。

---

## 2. 全体アーキテクチャ

### 2.1 データフロー

```
[開発時]                       [ビルド時]                       [接続時]

 keyboards/<kb>/
   remap_info.json   ──→  qmk generate-remap-info-blob
   (VIA 形式)                ↓
                        .build/<kb>_<keymap>/.../remap_info_blob.c
                                 ↓
                        [ファーム書き込み]
                                 ↓
                        [Flash 保存]  ─── HID GET_INFO_BLOB ───→  [Remap webapp]
                                                                       ↓
                                                            JSON.parse & 内部表現変換
                                                                       ↓
                                                            カスタマイズ画面表示
```

### 2.2 v0 の責任分界点

| レイヤ | 責任 |
|---|---|
| **`remap_info.json`**（開発者） | VIA 仕様に従ったキーボード定義記述 |
| **codegen ツール**（ビルド時） | info.json → C const array 生成 |
| **Firmware**（実行時） | blob を Flash に保持・HID 経由で全文提供 |
| **Webapp**（接続時） | blob 取得・パース・`KeyboardDefinition` 構築 |

**既存 VIA Protocol との関係**: 本仕様では既存の `dynamic_keymap`（VIA 互換 keymap GET/SET）には**触れない**。本仕様で追加するのは「info.json blob 取得」機能のみ。既存 keymap カスタマイズは VIA Protocol に乗ったまま動作する。

---

## 3. Source of Truth: VIA info.json

### 3.1 VIA info.json フォーマットの流用

ここで言う "info.json" は、**VIA が規定するキーボード定義 JSON フォーマット**を指す（`vendorId`, `productId`, KLE 形式の `layouts.keymap` 等を持つ）。QMK Firmware の `keyboard.json`（旧称 `info.json`）とは**別物**である点に注意。

Remap web app は現状、開発者が手元で作成した VIA info.json を Firestore にアップロードする運用になっており、本仕様ではその **VIA info.json をそのままファームウェアに焼き込んで再利用**する。Remap 独自フィールドの追加やフォーマット拡張は v0 では行わない。

### 3.2 ファイル配置とファイル名

VIA info.json は QMK の `keyboards/<kb>/keyboard.json`（および `info.json`）とは**別のファイル**として配置する：

| 項目 | 内容 |
|---|---|
| ファイル名 | `remap_info.json` |
| 配置先 | `keyboards/<kb>/remap_info.json` |
| フォーマット | VIA info.json 形式（既存 Remap 登録用 JSON と同一） |
| QMK ビルドへの影響 | 本ファイル単体はコンパイル対象外（codegen ツール経由でのみ参照） |

ファイル名選定理由：

- QMK の `keyboard.json` / `info.json` と名前衝突しない
- 「Remap が必要とする情報」が読み手に明示的
- 将来フォーマットを変更した場合（VIA 以外への移行）でも名前が腐らない

### 3.3 v0 で参照するフィールド

Remap web app の `KeyboardDefinition` 構築に必要なフィールドのうち、`remap_info.json`（VIA フォーマット）から取得するもの：

| VIA info.json フィールド | KeyboardDefinition 対応 | 用途 |
|---|---|---|
| `name` | `name` | キーボード名表示 |
| `vendorId` | `vendorId` | デバイス識別（hex 文字列） |
| `productId` | `productId` | デバイス識別（hex 文字列） |
| `matrix.rows` | `matrix.rows` | マトリクスサイズ |
| `matrix.cols` | `matrix.cols` | マトリクスサイズ |
| `layouts.keymap` | `layouts.keymap` | KLE レイアウト本体 |
| `layouts.labels`（任意） | `layouts.labels` | レイアウトラベル |
| `layouts.presets`（任意） | `layouts.presets` | レイアウトプリセット |
| `customKeycodes`（任意） | `customKeycodes` | カスタムキーコード定義 |
| `customFeatures`（任意） | `customFeatures` | カスタム機能 |
| `lighting`（任意） | `lighting` | ライティング設定 |

### 3.4 Remap Firestore 登録項目との突合せ（棚卸し）

現状 Remap (Firestore) の `IKeyboardDefinitionDocument`（参照: `remap/src/services/storage/Storage.ts:56-92`）の全フィールドを、本仕様での扱い別に整理する。

#### 3.4.1 VIA info.json で賄えるフィールド ✅（v0 対象）

| Firestore フィールド | VIA info.json 由来 |
|---|---|
| `name` | `name` |
| `vendorId` | `vendorId` |
| `productId` | `productId` |
| `json` | VIA info.json 全体（minified） |
| `customKeycodes`（任意） | `customKeycodes`（任意） |

#### 3.4.2 USB device descriptor で取得可能 ✅（v0 対象、ファーム側追加実装不要）

| Firestore フィールド | 取得経路 |
|---|---|
| `productName` | USB Product String（QMK が `#define PRODUCT` から自動設定） |

Web app は WebHID API の `device.productName` で取得可能。本仕様で追加の対応は不要。

#### 3.4.3 カスタマイズに不要なフィールド ⛔（v0 スコープ外）

以下は Remap カタログ・author 証跡用であり、キーボードのカスタマイズ動作には不要：

| Firestore フィールド | 用途 | v0 での扱い |
|---|---|---|
| `status` | Firestore レコードのレビュー状態 | プラグアンドプレイ経路では Firestore レコード自体が存在しないため不要 |
| `authorUid`, `authorType`, `organizationId` | 登録者識別 | 不要（事前登録しないため） |
| `githubUrl`, `githubDisplayName` | GitHub 認証情報 | 不要 |
| `firmwareCodePlace`, `qmkRepositoryFirstPullRequestUrl`, `forkedRepositoryUrl`, `forkedRepositoryEvidence`, `otherPlaceHowToGet`, `otherPlaceSourceCodeEvidence`, `otherPlacePublisherEvidence`, `contactInformation`, `organizationEvidence` | 登録時の証跡 | 不要 |
| `features` | カタログ機能タグ | カタログ表示用、カスタマイズに不要 |
| `description`, `additionalDescriptions` | カタログ説明文 | 同上 |
| `stores` | 販売店情報 | 同上 |
| `websiteUrl` | 公式サイト URL | 同上 |
| `thumbnailImageUrl`, `imageUrl`, `subImages` | カタログ画像 | 同上 |
| `firmwares` | ファームウェアファイル配信 | ユーザはすでにファーム書き込み済みのため不要 |
| `createdAt`, `updatedAt` | システム生成タイムスタンプ | 不要 |

#### 3.4.4 結論

**カスタマイズに必須のフィールドはすべて VIA info.json で賄える**（または USB descriptor で取得可能）。事前登録なしで実現可能な範囲が「カスタマイズ機能のみ」であることを意味し、カタログ機能（説明・画像・販売店情報）はプラグアンドプレイ対応せず、必要なら従来通り Firestore 経路で別途提供する形となる。これは v0 の許容トレードオフ。

---

## 4. ビルド時 codegen

### 4.1 ツール

- **名前**: `qmk generate-remap-info-blob`（仮）
- **実装位置**: `lib/python/qmk/cli/generate/remap_info_blob.py`
- **方針**: 既存 `qmk generate-*` 系コマンド（`generate-config-h`, `generate-keyboard-c` 等）と同じ pattern

### 4.2 入力

`keyboards/<kb>/remap_info.json`（§3.2 で定義した VIA info.json ファイル）

### 4.3 出力

`.build/<kb>_<keymap>/obj_<kb>_<keymap>/src/remap_info_blob.c`（既存 QMK の自動生成 C ファイル配置に従う）

生成 C ファイル概形：

```c
// Generated by qmk generate-remap-info-blob. DO NOT EDIT.
#include "remap_info_blob.h"

const char REMAP_INFO_BLOB_JSON[] PROGMEM =
    "{\"name\":\"foo\",\"vendorId\":\"0x1234\",\"productId\":\"0x5678\", ...}";

const uint32_t REMAP_INFO_BLOB_JSON_SIZE = sizeof(REMAP_INFO_BLOB_JSON) - 1;
```

### 4.4 保存形式

- **テキスト形式**: minified JSON（改行・余分なスペース削除）
- **エンコーディング**: UTF-8
- **圧縮**: なし（v0 では）

選定理由：

- 生 JSON テキストは webapp 側で `JSON.parse` 一発で消費可能
- 圧縮を入れると codegen 複雑化 + ファーム側 / webapp 側双方の依存増加
- minified だけで通常 20-40% 程度の削減効果あり

### 4.5 Make 統合

- `rules.mk` で `REMAP_INFO_BLOB_ENABLE = yes` をオプトイン
- 有効時のみ `qmk generate-remap-info-blob` を発火し、生成 C ファイルをコンパイル対象に追加
- `remap_info.json` の更新で再生成（既存 QMK の build 依存関係に追随）

無効時はコード生成・コンパイル両方ともスキップ。

---

## 5. ファームウェア側保存

### 5.1 ストレージ

| MCU 種別 | ストレージ | C 修飾子 |
|---|---|---|
| AVR | Flash ROM | `PROGMEM` |
| ARM (ChibiOS) | Flash ROM | `const`（自動配置） |

RAM（容量圧迫）・EEPROM（書き換え寿命・容量不足）は使用しない。

### 5.2 メモリレイアウト

シンプルさを優先し、v0 では **JSON 本体 + 長さ定数**の最小構成：

```c
// remap_info_blob.h
#pragma once
#include <stdint.h>
#include "progmem.h"

extern const char REMAP_INFO_BLOB_JSON[] PROGMEM;
extern const uint32_t REMAP_INFO_BLOB_JSON_SIZE;
```

ヘッダ構造体は導入せず、JSON テキストそのものと、その長さ定数のみを公開する。整合性チェック（CRC 等）は v0 では行わない（Flash 破損は通常起こらないため）。

### 5.3 Flash サイズ見積もり

| キーボードタイプ | 典型サイズ（minified） |
|---|---|
| 小型（40 キー前後・単一レイアウト） | ~1 KB |
| 中型（60-80 キー・複数レイアウト） | ~2-4 KB |
| 大型（フルサイズ・複数バリエーション） | ~6-8 KB |

MCU 別 Flash 余裕度のガイドライン：

- `atmega32u4`（32 KB Flash、QMK で typ. 18-22 KB 使用）: 残り **8-12 KB** 程度。中型までは問題なし
- ARM（Cortex-M0/M4 系、128 KB+）: 通常問題にならない

### 5.4 機能フラグ

| フラグ | 場所 | デフォルト |
|---|---|---|
| `REMAP_INFO_BLOB_ENABLE` | `rules.mk` | `no` |

有効化方法：

```makefile
# keyboards/<kb>/keymaps/<keymap>/rules.mk
REMAP_INFO_BLOB_ENABLE = yes
```

VIA 既存機能（`VIA_ENABLE`）とは独立して有効化可能（両立可能）。

---

## 6. HID プロトコル

### 6.1 コマンド ID

- VIA Protocol 既存コマンド（0x00-0x0E 周辺）と衝突しない ID を使用
- **本仕様の提案**: `0xFE`
- VIAL 等の他拡張との実際の衝突確認は実装時に行う

### 6.2 サブコマンド

| sub_cmd | 名前 | 機能 |
|---|---|---|
| 0x01 | `GET_INFO_BLOB_META` | blob 全体サイズ・チャンク数を返す |
| 0x02 | `GET_INFO_BLOB_CHUNK` | 指定 `chunk_idx` の中身を返す |

### 6.3 パケットフォーマット (32B)

HID Raw レポート長は 32 バイト固定（`RAW_EPSIZE`）。

#### GET_INFO_BLOB_META

**リクエスト**:

```
offset  size  field
[0]      1B    cmd_id (0xFE)
[1]      1B    sub_cmd (0x01)
[2-31]  30B    未使用 (0 埋め)
```

**レスポンス**:

```
offset  size  field
[0]      1B    cmd_id (0xFE)
[1]      1B    sub_cmd (0x01)
[2-5]    4B    total_size       (uint32_le, blob 全体バイト数)
[6-7]    2B    chunk_count      (uint16_le)
[8]      1B    chunk_size       (uint8, 通常 27)
[9-31]  23B    予約 (0 埋め)
```

#### GET_INFO_BLOB_CHUNK

**リクエスト**:

```
offset  size  field
[0]      1B    cmd_id (0xFE)
[1]      1B    sub_cmd (0x02)
[2-3]    2B    chunk_idx (uint16_le)
[4-31]  28B    未使用 (0 埋め)
```

**レスポンス**:

```
offset  size  field
[0]      1B    cmd_id (0xFE)
[1]      1B    sub_cmd (0x02)
[2-3]    2B    chunk_idx (uint16_le, echo)
[4]      1B    bytes_in_this_chunk (1-27)
[5-31]  27B    payload (JSON bytes、末尾は 0 埋め)
```

### 6.4 チャンク転送シーケンス

```
Webapp                           Firmware
  │                                  │
  │── GET_INFO_BLOB_META ──────────→ │
  │←── total=2048, chunks=76 ────────│
  │                                  │
  │── GET_INFO_BLOB_CHUNK(0) ──────→ │
  │←── bytes=27, payload[27] ────────│
  │                                  │
  │── GET_INFO_BLOB_CHUNK(1) ──────→ │
  │←── bytes=27, payload[27] ────────│
  │                                  │
  │           ... (繰り返し) ...       │
  │                                  │
  │── GET_INFO_BLOB_CHUNK(75) ─────→ │
  │←── bytes=20, payload[20] ────────│  (最終チャンク=端数)
  │                                  │
  │  webapp 側でバッファ結合            │
  │  → UTF-8 デコード → JSON.parse    │
```

エラー処理は v0 では最小限：

- 不正な `chunk_idx`（範囲外）→ ファーム側はレスポンスを返さない（タイムアウト扱い）
- Webapp 側で N 回リトライ → 最終的に既存 Firestore 経路にフォールバック

---

## 7. Webapp 統合

### 7.1 Remap ファームウェア検出

接続時に webapp は以下フローで Remap ファームかどうか判定する：

1. WebHID で device open 後、`GET_INFO_BLOB_META` を送信
2. 一定時間内に正常レスポンスあり → **Remap ファームと認識**、blob 取得フローへ進む
3. タイムアウト or 不正レスポンス → 既存 Firestore lookup 経路にフォールバック

### 7.2 info.json 取得・パース

1. `GET_INFO_BLOB_META` で `total_size`, `chunk_count`, `chunk_size` を取得
2. `chunk_count` 回 `GET_INFO_BLOB_CHUNK(i)` を発行し、`bytes_in_this_chunk` 分だけバッファに連結
3. UTF-8 文字列としてデコード
4. `JSON.parse` で JS object 化

### 7.3 既存 KeyboardDefinition 形式への変換

パースした VIA info.json を、Remap webapp 内部の `KeyboardDefinition` 型に変換するアダプタを実装：

- 既存の Firestore 経路（`KeyboardDefinitionSchema`）で組み立てる `KeyboardDefinition` と同じ型に揃える
- 詳細なフィールドマッピングは webapp 側 spec で確定（本仕様の対象外）

### 7.4 既存 Firestore 経路との関係

段階的移行を想定：

- **Remap ファーム搭載キーボード**: HID 経由 VIA info.json blob → `KeyboardDefinition` 構築
- **VIA / 既存登録キーボード**: 従来通り Firestore lookup
- 接続時の判定でどちらの経路を使うか自動分岐

両経路で得た `KeyboardDefinition` は同一型のため、後段のカスタマイズ UI は経路を意識しない。

---

## 8. スコープ外（v1 以降に持ち越し）

以下は本仕様 v0 では扱わず、`2026-04-29-remap-firmware-extensions-design.md` 等で別途設計：

- `remap.*` namespace（`schema_version`, 将来の Remap 独自拡張ポイント）
- レイアウトオプション（boolean/enum）の拡張メタデータ
- per-key 拡張情報（色、回転、ラベル、エンコーダ詳細等）
- マイグレーションツール（`qmk migrate-to-remap` 等）
- 動的設定機能（Tap-Hold / Combo / Tap Dance / Auto Shift / Caps Word 等の web app からの GUI 設定）
- VIA Protocol からの完全分離（Remap protocol への一本化）
- HID プロトコル拡張（per-key tapping term, macro 用領域, etc.）
- バイナリエンコーディング・圧縮対応
- カタログ系メタデータ（description, images, stores 等）のファームウェア埋め込み

---

## 9. リビジョン履歴

| 日付 | 著者 | 変更内容 |
|---|---|---|
| 2026-05-12 | 洋一郎 / Claude | 初版作成（v0 最小スペック） |
| 2026-05-12 | 洋一郎 / Claude | info.json が VIA フォーマットであることを明示、`remap_info.json` 配置を §3.2 に新設、フィールド表を VIA 仕様に修正、codegen ツール名を `qmk generate-remap-info-blob` に変更 |
