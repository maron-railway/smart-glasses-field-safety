# hud_renderer: 0x4E Text Rendering Design (EVEN Gx / Dual BLE)

## 目的
`hud_renderer` は **現場で読めるテキスト** を、EVEN系スマートグラスの **Text Show（0x4E）** で確実に表示させるためのレンダリング層。

責務は「文章 → 行分割 → ページング → パケット化 → 0x4Eフレーム列の生成」まで。
送信（BLE I/O）やACK待ち/再送は `ble_core` の責務とする。

---

## 前提（表示・運用制約）
- ディスプレイは横幅制約が強い（EvenDemoAppの例では `max width = 488 px`）
- 1画面あたりの表示行数は有限（例: 5行）
- 現場用途のため、以下を重視する  
  - **短文・要点先頭**
  - **手袋/暗所でも読みやすい**
  - **ページ送りが少ない**
  - **誤解しない（数値/単位/否定が潰れない）**

---

## プロトコル要点（0x4E / Text Show）
### フレーム構造（Text Show）
Command: `0x4E`

フィールド（EvenDemoApp記載に準拠）:
- `seq` (0~255): 送信シーケンス
- `total_package_num` (1~255): 全パケット数
- `current_package_num` (0~255): 現在パケット番号
- `newscreen`: 画面/モード指定  
  - Lower 4 bits: `0x01` = Display new content  
  - Upper 4 bits: `0x70` = Text Show  
  - よって Text Show の新規表示は **`0x71`**
- `new_char_pos0`, `new_char_pos1`: 新規文字位置（現状は 0 固定運用でよい。将来拡張）
- `current_page_num` (0~255)
- `max_page_num` (1~255)
- `data`: 表示データ本体

※ `data` の文字コードや終端の扱いは、まず EvenDemoApp の実装（送信側）に合わせる。
（UTF-8 / ASCII 想定。まずは英数+記号で安全運用 → 日本語は実機確認後に拡張）

---

## レンダリングパイプライン

```
RawText
  ↓ normalize (trim / unify newline / sanitize)
  ↓ tokenize (words / punctuation / numbers)
  ↓ line_break (width=488px, font_size=21)
  ↓ page_pack (lines_per_screen=5)
  ↓ packetize (fit BLE payload constraints)
  ↓ build_0x4E_frames (seq, totals, page info)
```

---

## 1) Normalize（整形）
入力 `RawText` を次のルールで整形する。

### ルール
- 改行の正規化: `\r\n` → `\n`
- 連続空行は最大1つ
- 先頭/末尾の空白削除
- タブはスペース1つへ
- 危険: “否定”や“数値+単位”が分離されないように保護
  - 例: `NG`, `禁止`, `不可`, `0V`, `750V`, `kV`, `A`, `mA`, `mm` などはなるべく同一行内に収める

---

## 2) Line Break（行分割）
### 入力
- `width_px`（例: 488）
- `font_size`（例: 21）
- `text`

### 出力
- `Vec<String>` lines

### 実装方針（段階的）
**Phase 1（確実に動く）**
- 等幅近似で分割（文字数ベース）
  - `max_cols = floor(width_px / (font_size * k))`
  - 係数 `k` は実測で調整（仮: 0.55〜0.65）
- 分割単位:
  - 空白があれば単語境界優先
  - 空白がない（日本語など）場合は文字境界
- 句読点/記号（`, . : ; ) ]`）は行頭に来ないよう調整

**Phase 2（品質UP）**
- 文字種ごとの幅テーブル（ASCII/数字/大文字/小文字/全角）で概算
- 数値+単位はセット扱い（例: `750V`）

**Phase 3（理想）**
- 実機フォントメトリクスに基づく幅計算（可能なら）

---

## 3) Paging（ページング）
### パラメータ
- `lines_per_screen`（例: 5）

### 出力
- `Vec<Page>`  
  `Page { page_index, max_pages, lines: Vec<String> }`

### ルール
- 1ページは最大 `lines_per_screen`
- 最終ページは不足行があってもOK（詰めない）
- ページ先頭に **状況ラベル** を入れる運用も検討
  - 例: `【注意】`, `【手順】`, `【異常】`
  - ただし表示行が減るので、重要時のみ

---

## 4) Packetize（BLEパケット化）
EvenDemoApp例では「1画面のテキスト」をさらに複数パケットに分割して送っている。

### パラメータ（暫定）
- `ble_payload_max`（ble_coreが管理するMTUに依存）
- 例運用（EvenDemoApp）:
  - 5行/画面
  - 3行 + 2行 の2パケットに分けることがある

### 方針
- `hud_renderer` は **“ページ→複数パケット”** の分割ロジックを持つ
- 各パケットは `0x4E` の `data` に載る範囲で行を詰める
- 画面単位送信: `ble_core` が「左→ACK→右」の順序で送る

---

## 5) 0x4E Frame Builder（フレーム生成）
### 入力
- `pages: Vec<Page>`
- `seq_start: u8`（ble_coreから供給してもいい）

### 出力
- `Vec<TxFrame>`（送信すべき0x4Eの列）

#### newscreen の扱い
- Text Showは原則 `0x71`（new content + text show）
- 同一ページ内の複数パケットも `0x71` 固定で開始してOK（まずは簡単運用）
- 将来: 継続パケットは lower bits を変える可能性があるが、現状は仕様未確定なので触らない

#### page 番号
- `current_page_num`: 0-based or 1-based は実装で統一（まずは 0-based 推奨）
- `max_page_num`: 総ページ数（1以上）

---

## データフォーマット（data）
未確定部分があるため、**EvenDemoApp互換を最優先**にする。

推奨（暫定）:
- 行区切り: `\n`
- ページ内パケットに含めるデータは「行単位」で詰める
- 末尾に余計なNULLを入れない（必要なら ble_core側で調整）

---

## API案（Rust）
```rust
pub struct RenderConfig {
    pub width_px: u16,          // 488
    pub font_size: u8,          // 21
    pub lines_per_screen: u8,   // 5
    pub ble_payload_max: usize, // e.g. 180.. depending on MTU and header
}

pub struct TextJob<'a> {
    pub text: &'a str,
    pub title: Option<&'a str>,     // optional label
    pub priority: u8,               // for future: warning banners
}

pub struct Page {
    pub page_index: u8,
    pub max_pages: u8,
    pub lines: Vec<String>,
}

pub struct TxFrame {
    pub cmd: u8,            // 0x4E
    pub payload: Vec<u8>,   // fully built
    pub page_index: u8,
    pub packet_index: u8,
    pub packet_total: u8,
}

pub fn render_text(job: TextJob, cfg: &RenderConfig) -> Vec<TxFrame>;
```

---

## テスト方針
### ユニットテスト
- 行分割（英数/記号/長単語）
- 数値+単位の分離防止（例: `750V`, `1.2kV`, `5mA`）
- ページング境界（ちょうど5行/6行/10行）
- payload_max を超えないこと

### ゴールデンテスト（互換）
- EvenDemoAppと同入力で「生成フレームの構造」が一致するか（可能なら）

---

## 既知の未確定ポイント（要実機確認）
- `data` の文字コード（UTF-8で日本語が素直に出るか）
- `new_char
Placeholder
