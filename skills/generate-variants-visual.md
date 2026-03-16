# スキル：バリアントビジュアル生成

## 役割

コンポーネントのSize・State・Shapeの各バリアントを横並びにしたFigmaフレームを生成し、
スクリーンショットをMDXのバリアントセクションに挿入する。

`generate-component-doc.md` からサブスキルとして呼び出されるほか、単独でも使用できる。

---

## インプット

| 項目 | 必須 | 説明 |
|---|---|---|
| 対象コンポーネントの Figma URL または node ID | ✅ | |
| 出力先MDXファイルパス | ✅ | |

---

## 実行手順

### Step 1: プロパティの取得
```
get_design_context(nodeId)
```

component properties から以下を取得する：

| プロパティ名 | 種別 | 用途 |
|---|---|---|
| `size` | variant | Sizeセクションの横並び |
| `state` | variant | Stateセクションの横並び（一部） |
| `focused` | boolean | Stateセクション（focused状態） |
| `isDisabled` | boolean | Stateセクション（disabled状態） |
| `full-radius` | boolean | Shapeセクション |

取得できたプロパティのみセクションを生成する。存在しないプロパティはスキップする。

---

### Step 2: Size フレームの生成（`size` プロパティがある場合）

`figma_execute` を使い、sizeの値を小さい順に横並びにしたフレームを生成する。
```javascript
// 横並びフレームの生成パターン
const frame = figma.createFrame();
frame.name = '{ComponentName}/Size';
frame.layoutMode = 'HORIZONTAL';
frame.itemSpacing = 24;
frame.paddingTop = frame.paddingBottom = 32;
frame.paddingLeft = frame.paddingRight = 32;
frame.fills = [{ type: 'SOLID', color: { r: 0.97, g: 0.97, b: 0.97 } }];
frame.cornerRadius = 12;

// sizeの値を小さい順に並べる（sm → md → lg → xl）
const sizeOrder = ['2xs', 'xs', 'sm', 'md', 'lg', 'xl', '2xl'];
const sortedSizes = availableSizes.sort((a, b) => sizeOrder.indexOf(a) - sizeOrder.indexOf(b));

for (const size of sortedSizes) {
  const instance = component.createInstance();
  instance.setProperties({ size });
  frame.appendChild(instance);
}
```

各インスタンスの下にサイズ名ラベル（`sm`, `md` 等）をテキストで追加する。

---

### Step 3: State フレームの生成（`state` または `focused` / `isDisabled` がある場合）

以下の5パターンを左から順に横並びにする：

| 表示名 | プロパティ設定 |
|---|---|
| Default | `state=default`, `focused=false`, `isDisabled=false` |
| Hover | `state=hover` |
| Pressed | `state=pressed` |
| Focused | `state=default`, `focused=true` |
| Disabled | `state=default`, `isDisabled=true` |

存在しないプロパティはスキップして生成できるパターンのみ並べる。
```javascript
const frame = figma.createFrame();
frame.name = '{ComponentName}/State';
frame.layoutMode = 'HORIZONTAL';
frame.itemSpacing = 24;
frame.paddingTop = frame.paddingBottom = 32;
frame.paddingLeft = frame.paddingRight = 32;
frame.fills = [{ type: 'SOLID', color: { r: 0.97, g: 0.97, b: 0.97 } }];
frame.cornerRadius = 12;
```

各インスタンスの下に状態名ラベルを追加する。

---

### Step 4: Shape フレームの生成（`full-radius` プロパティがある場合）

`full-radius=false`（角丸）と `full-radius=true`（フルラディウス）を横並びにする。

各インスタンスの下にラベルを追加する：
- `full-radius=false` → `Default`
- `full-radius=true` → `Full radius`

---

### Step 5: スクリーンショットの取得と保存

生成した各フレームのnode IDを確認してからスクリーンショットを取得する。
```
get_screenshot(frameNodeId)
```

保存先：
- `images/{component-name}-size-light.png`
- `images/{component-name}-state-light.png`
- `images/{component-name}-shape-light.png`

**重要：** `get_screenshot` の戻り値を直接使わず、必ずファイルとして保存してからパスを参照する。

---

### Step 6: MDXへの挿入

出力先MDXの `## バリアント` セクション内、sizeテーブルの直後に以下の順で挿入する：
```md
## バリアント

### Size

![Button Size](/images/button-size-light.png)

| サイズ | 用途 |
|---|---|
| sm | （説明） |
| md | （説明） |
| lg | （説明） |
| xl | （説明） |

### State

![Button State](/images/button-state-light.png)

| ステート | 説明 |
|---|---|
| Default | 通常状態 |
| Hover | カーソルが乗っている状態 |
| Pressed | クリック・タップ中の状態 |
| Focused | キーボードフォーカス状態 |
| Disabled | 操作不可状態。`opacity: 0.4` を適用 |

### Shape

![Button Shape](/images/button-shape-light.png)

| 形状 | 説明 |
|---|---|
| Default | 標準の角丸 |
| Full radius | ピル型。角丸を最大にした形状 |

### その他のプロパティ

| プロパティ | 説明 |
|---|---|
| hasLeftIcon / hasRightIcon | アイコンの表示・非表示を切り替える |
| isPending | 処理中状態。ローディングインジケーターを表示する |
```

`figma-frame` コメントは各画像の直前に追記する。

---

## フォールバック

| 状況 | 対処 |
|---|---|
| プロパティが存在しない | そのセクションをスキップする |
| フレーム生成に失敗 | `<!-- TODO: {セクション名}フレームを手動で作成後に更新 -->` を挿入して続行 |
| スクリーンショット取得に失敗 | `<!-- figma-frame: 取得失敗 - node IDを手動で記入 -->` を挿入して続行 |
