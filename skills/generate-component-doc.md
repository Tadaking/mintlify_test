# スキル：コンポーネントガイドライン生成

## トリガー条件

以下のいずれかに該当する場合、このスキルを使う：

- コンポーネントのガイドラインを新規作成する
- 既存のコンポーネントガイドラインに新しいプラットフォームタブを追加する

---

## インプット

| 項目 | 必須 | 説明 |
|---|---|---|
| Figma URL または node ID | ✅ | 対象コンポーネントのFigmaフレームまたはコンポーネントセット |
| 対象プラットフォーム | ✅ | `web` / `ios` / `android` / `all` |
| 出力ファイルパス | — | 省略時は `components/{component-name}.mdx` に自動決定 |

---

## Figma クエリ手順

以下の順番で実行する。

### Step 1: コンポーネント構造の取得
```
get_design_context(nodeId)
```

取得対象：
- component properties → バリアント定義として使う
- 子レイヤー名 → 構成要素（アイコン、ラベル、ボーダー等）として使う
- コンポーネントの description → 概要セクションの下書きとして使う

### Step 2: トークンの取得
```
get_variable_defs(nodeId)
```

取得対象：
- `COLOR` scope → カラートークンテーブルに使う
- `WIDTH_HEIGHT` / `GAP` / `CORNER_RADIUS` scope → サイズ・スペーシングテーブルに使う

### Step 3: `_guidelines` フレームの探索

コンポーネントと同じページ内で `_guidelines` という名前のフレームを探す。

**見つかった場合：**
```
get_design_context(_guidelines の nodeId)
```

以下のレイヤー構造に従って各セクションに振り分ける：

| レイヤーパス | 使用先 |
|---|---|
| `overview` | `## 概要` セクション |
| `related` | `## 類似要素との使い分け` セクション |
| `variants/{name}/when-to-use` | 該当バリアントの「✅ いつ使う」 |
| `variants/{name}/when-not-to-use` | 該当バリアントの「❌ いつ使わない」 |
| `combinations/do-{n}/caption` | 組み合わせパターンのDoキャプション |
| `combinations/dont-{n}/caption` | 組み合わせパターンのDon'tキャプション |
| `combinations/caution-{n}/caption` | 組み合わせパターンのCautionキャプション |

`combinations` の `frame` レイヤーはスクリーンショットを取得して `images/{component-name}-combination-{do|dont|caution}-{n}-light.png` として保存する。

MDXの `## 組み合わせパターン` セクションは `do-dont-grid` クラスを使い、以下の形式で生成する：
```md
<div class="do-dont-grid">
  <div class="do-card">
    ![](/images/{component-name}-combination-do-1-light.png)
    <div class="do-dont-label">✅ Do</div>
    <div class="do-dont-caption">{caption テキスト}</div>
  </div>
  <div class="dont-card">
    ![](/images/{component-name}-combination-dont-1-light.png)
    <div class="do-dont-label">❌ Don't</div>
    <div class="do-dont-caption">{caption テキスト}</div>
  </div>
</div>

<div class="do-dont-grid">
  <div class="caution-card">
    ![](/images/{component-name}-combination-caution-1-light.png)
    <div class="do-dont-label">⚠️ Caution</div>
    <div class="do-dont-caption">{caption テキスト}</div>
  </div>
</div>
```

do / dont / caution の数は `_guidelines` フレームの実際の連番に応じて動的に生成する。

**見つからなかった場合：**
該当セクションを以下のTODOコメントに置き換えて続行する：
```md
<!-- TODO: _guidelines フレームを追加後に更新 -->
```

### Step 4: スクリーンショットの取得

Figma Console MCP を使用する（読み取り専用のリモートMCPではない）。
```
get_screenshot(nodeId)
```

- `images/{component-name}-overview-light.png` として保存する
- MDXファイル内に `<!-- figma-frame: FILE_KEY/NODE_ID | バリアント一覧 -->` を記録する

### Step 5: Anatomy フレームの生成

`skills/generate-anatomy.md` をサブスキルとして呼び出す。

- Figma Console MCP の書き込み機能を使って対象コンポーネントの隣にAnatomyフレームを生成する
- 生成後に `get_screenshot` で書き出し、`images/{component-name}-anatomy-light.png` として保存する
- MDXファイル内に `<!-- figma-frame: FILE_KEY/NODE_ID | Anatomy -->` を記録する

> **注:** `skills/generate-anatomy.md` が未作成の場合はこのステップをスキップし、
> `<!-- TODO: generate-anatomy スキル作成後に実行 -->` をMDXに挿入して続行する。

### Step 6: バリアントビジュアルの生成

`skills/generate-variants-visual.md` をサブスキルとして呼び出す。

- Figma Console MCP の書き込み機能を使って Size・State・Shape フレームを生成する
- 生成後に各フレームのスクリーンショットを取得し `images/` に保存する
- MDXの `## バリアント` セクションに挿入する

> **注:** `skills/generate-variants-visual.md` が未作成の場合はこのステップをスキップし、
> `<!-- TODO: generate-variants-visual スキル作成後に実行 -->` をMDXに挿入して続行する。

---

## セクションマッピング

CLAUDE.md の `#### Overview タブの必須セクション` に従い、以下のルールで変換する。

| Figmaから取れるもの | MDXのセクション | なかった場合 |
|---|---|---|
| component description | `## 概要` | Claude が構造から推測して下書き + TODOコメント |
| component properties | `## バリアント` の各バリアント名 | そのまま列挙してTODOコメント |
| `_guidelines` > `when-to-use` | バリアントの「✅ いつ使う」 | TODOコメント |
| `_guidelines` > `when-not-to-use` | バリアントの「❌ いつ使わない」 | TODOコメント |
| `_guidelines` > `related` | `## 類似要素との使い分け` | セクションごとスキップ |
| `_guidelines` > `do` / `dont` | `<!-- illustration-needed: -->` | TODOコメント |
| `COLOR` scope の variable_defs | カラートークンテーブル | プレースホルダー行を生成 |
| `WIDTH_HEIGHT` / `GAP` scope | サイズ・スペーシングテーブル | プレースホルダー行を生成 |

### 画像プレースホルダーの判断ルール

- バリアントが2種類以上 → `<!-- figma-frame: ... | バリアント一覧 -->`
- サイズ展開あり → `<!-- figma-frame: ... | サイズ比較 -->`
- `_guidelines` に `do` / `dont` のテキストあり → `<!-- illustration-needed: [内容] -->`
- `_guidelines` がない → `<!-- illustration-needed: Do/Don't（_guidelines 追加後に更新） -->`

---

## 出力ルール

1. ファイルを `components/{component-name}.mdx` に作成する
2. CLAUDE.md の `#### コンポーネントガイドライン` テンプレートに従う
3. `docs.json` の `navigation` に以下の要領で追加する：
   - `"Foundation"` グループにはコンポーネントを追加しない
   - `"Components"` グループがなければ新規作成して追加する
4. プラットフォームと生成タブの対応は CLAUDE.md の以下のテーブルに従う：

   | 渡されたFigmaファイル | 生成・更新するタブ |
   |---|---|
   | Web Components ファイル | Overview（初回のみ）+ Web |
   | iOS Components ファイル | iOS |
   | Android Components ファイル | Android |

---

## フォールバック

| 状況 | 対処 |
|---|---|
| `_guidelines` フレームが存在しない | TODOコメントを挿入して続行。ドキュメント冒頭に `<!-- _guidelines フレーム未作成：追加後に再実行 -->` を追記 |
| トークンが取得できない | カラー・サイズテーブルをプレースホルダー行で生成。セル内に `（要確認）` と記載 |
| component description がない | コンポーネント名と構造から Claude が概要を推測して下書きし、TODOコメントを添える |
| スクリーンショット取得に失敗 | `<!-- figma-frame: 取得失敗 - node IDを手動で記入 -->` を挿入して続行 |
