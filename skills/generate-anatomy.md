# スキル：Anatomy フレーム生成

## 役割

Figmaのテンプレートコンポーネントを複製してAnatomyフレームを生成し、
スクリーンショットをMDXのAnatomyセクションに挿入する。

`generate-component-doc.md` からサブスキルとして呼び出されるほか、単独でも使用できる。

---

## インプット

| 項目 | 必須 | 説明 |
|---|---|---|
| 対象コンポーネントの Figma URL または node ID | ✅ | AnatomyフレームのプレースホルダーにコピーするコンポーネントのノードID |
| 出力先MDXファイルパス | ✅ | Anatomyセクションの挿入先 |

---

## 実行手順

### Step 1: コンポーネント構造の取得
```
get_design_context(nodeId)
```

以下を取得する：
- **実際に存在する子レイヤーのみ**をリストアップする
- 非表示レイヤー・マスクレイヤーは除外する
- レイヤーとして存在しない要素（border、shadow等の見た目上の効果）は含めない
- 各レイヤーのtype（INSTANCE / TEXT / FRAME / VECTOR 等）を記録する

### Step 2: 要素の分類と注釈の準備

取得したレイヤーを以下のルールで分類し、テーブル用の注釈を準備する：

| type | 分類 | 注釈の書き方 |
|---|---|---|
| INSTANCE | Sub-component | `{名前} sub-component — always present` または boolean制御の場合 `hidden by default, shown via {propName} toggle` |
| TEXT | Text | テキスト内容が30文字以内なら `"{内容}" — primary label text`、それ以上は `Primary label text` |
| FRAME / GROUP（子が複数） | Container | `Layout container for {子要素の列挙}` |
| RECTANGLE / VECTOR 等 | Structural | `Background fill` / `Border/divider line` / `Decorative shape` 等、視覚的役割を説明 |

**絶対に書かない注釈：**
- `X instance`（役割説明なし）
- `Container with N children`
- `RECTANGLE` / `VECTOR`（Figmaのtype名をそのまま使う）

### Step 3: テンプレートの複製

Figma Console MCP を使い、以下のテンプレートコンポーネントを複製する：

- テンプレートURL: https://www.figma.com/design/Pvlx7A6oDLlnPF8MQYFDcq/Web-Components?node-id=5061-14376
- 複製先：対象コンポーネントと同じページの隣

複製後に以下のテキストを書き換える：
- `{component-name-anatomy-or-property}` → `{ComponentName} Anatomy`
- `{brief-component-description}` → Step 1で取得したコンポーネントのdescription（なければ空欄）
- `{section-name}` → `Elements`
- `{optional-description}` → 補足説明がある場合は内容を書き換える。不要な場合はレイヤーを削除する

### Step 4: コンポーネントの配置

グレーのプレースホルダーエリアに対象コンポーネントのインスタンスを配置する。

- defaultのバリアントを使用する
- プレースホルダーエリアの中央に配置する

### Step 5: 番号バッジと引き出し線の配置

各要素のbboxを使い、バッジを上下左右の最適な方向に振り分けて配置する。

#### 5-1: 配置方向の決定

コンポーネント全体のbboxを基準に、各要素の位置から配置方向を決める：

- 要素がコンポーネントの**左端付近** → バッジを左に配置（線は水平・左方向）
- 要素がコンポーネントの**右端付近** → バッジを右に配置（線は水平・右方向）
- 要素がコンポーネントの**上端付近** → バッジを上に配置（線は垂直・上方向）
- 要素がコンポーネントの**下端付近** → バッジを下に配置（線は垂直・下方向）
- 中央付近の要素は隣接する要素と被らない方向を選ぶ

同じ方向に複数のバッジが集中する場合は、バッジ同士が重ならないよう等間隔でずらす。

#### 5-2: バッジの配置

テンプレートの `#` バッジを要素の数だけ複製し、決定した方向にコンポーネントから **40px** 離して配置する。
バッジの番号テキストを `1`, `2`, `3`... に書き換える。

バッジの色: `#E7006C`

#### 5-3: 引き出し線の描画

`figma_execute` を使い、要素のエッジからバッジの中心まで**直線1本**を描画する。
折れ線・L字・曲線は使わない。
```javascript
// figma_execute で実行するコードのパターン
const line = figma.createLine();

// 方向に応じて始点・終点を設定
// 左方向の例：要素の左端中心 → バッジ右端中心
line.x = elementX;
line.y = elementY + elementHeight / 2;
line.resize(badgeCenterX - elementX, 0);

// スタイル
line.strokes = [{ type: 'SOLID', color: { r: 0.906, g: 0, b: 0.424 } }]; // #E7006C
line.strokeWeight = 1;
line.opacity = 0.7;
parent.appendChild(line);
```

方向別の座標ルール：
- **左**: 始点 = 要素左端中心、終点 = バッジ右端中心（水平線）
- **右**: 始点 = 要素右端中心、終点 = バッジ左端中心（水平線）
- **上**: 始点 = 要素上端中心、終点 = バッジ下端中心（垂直線）
- **下**: 始点 = 要素下端中心、終点 = バッジ上端中心（垂直線）

### Step 6: スクリーンショットの取得と保存

複製したAnatomyフレームのnode IDを確認してから実行する（元のコンポーネントやテンプレートのnode IDを使わない）。
```
get_screenshot(anatomyFrameNodeId)
```

**重要：** `get_screenshot` の戻り値をClaudeに直接渡さない。必ず以下の手順でファイルとして保存してからパスを参照する：

1. 戻り値の画像データを `images/button-anatomy-light.png` としてファイルに書き出す
2. 保存されたファイルパスをMDXで参照する
3. 画像データをそのままAPIに送ると `400 Could not process image` エラーが発生するため絶対に行わない

### Step 7: MDXへの挿入

出力先MDXファイルの `## Anatomy` セクションを以下の形式で更新する：
```md
![Button Anatomy](/images/button-anatomy-light.png)

| # | Type | Element name | Notes |
|---|---|---|---|
| 1 | Sub-component | {要素名} | {注釈} |
| 2 | Text | {要素名} | {注釈} |
```

- `<!-- figma-frame -->` コメントはnode IDの記録用に画像の直前に追記する
- Typeの値：`Sub-component` / `Text` / `Container` / `Structural`

---

## フォールバック

| 状況 | 対処 |
|---|---|
| テンプレートの複製に失敗 | `<!-- TODO: Anatomyフレームを手動で作成後に更新 -->` を挿入して続行 |
| コンポーネントの配置に失敗 | フレームだけ生成してコンポーネントのnode IDをコメントに残す |
| スクリーンショットがずれる | node IDを再確認して再取得。それでも失敗する場合は `<!-- figma-frame: 取得失敗 - node IDを手動で記入 -->` を挿入 |
| 子レイヤーが0件 | `get_design_context` を再実行し、それでも取得できない場合はTODOコメントを挿入して続行 |
