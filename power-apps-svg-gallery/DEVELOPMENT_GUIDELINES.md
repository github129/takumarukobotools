# Power Apps SVG Gallery 開発ガイドライン

このドキュメントは、power-apps-svg-galleryプロジェクトに新しいSVGサンプルを追加する際のルールとベストプラクティスをまとめたものです。

## 目次

1. [プロジェクト構造](#プロジェクト構造)
2. [新しいサンプルの追加手順](#新しいサンプルの追加手順)
3. [Power Fxコード作成のルール](#power-fxコード作成のルール)
4. [よくあるエラーと修正方法](#よくあるエラーと修正方法)
5. [コーディング規約](#コーディング規約)

---

## プロジェクト構造

```
power-apps-svg-gallery/
├── index.html          # メインファイル（SVGサンプルとPower Fxコードを含む）
└── DEVELOPMENT_GUIDELINES.md  # このファイル
```

### index.htmlの構造

index.htmlは以下の3つの主要セクションで構成されています：

1. **SVGサンプル配列** (行1-1060頃)
   - 各サンプルのid, title, category, description, svgCodeを定義

2. **getPowerAppsCode関数** (行1200-3260頃)
   - 各サンプルのIDに対応するPower Fxコードを生成

3. **UIとイベント処理** (行1030-3280頃)
   - ギャラリーの表示、検索、モーダル表示などの処理

---

## 新しいサンプルの追加手順

### 1. IDの決定

既存のサンプルの最大IDを確認し、次の番号を使用します。

```bash
grep -oP 'id: \d+' power-apps-svg-gallery/index.html | grep -oP '\d+' | sort -n | tail -1
```

### 2. SVGサンプルの追加

サンプル配列の末尾（`}` の後、`];` の前）に新しいサンプルを追加します。

```javascript
{
    id: 41,
    title: "複数折れ線グラフ",
    category: "graph",  // または ["graph", "animation"] のように配列も可
    description: "複数のデータ系列を比較できる折れ線グラフ。各系列を異なる色で表示。",
    svgCode: `<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 300 150" width="300" height="150">
        <!-- SVGコードをここに記述 -->
    </svg>`
}
```

**注意事項：**
- カンマ `,` を前のサンプルの閉じ括弧 `}` の後に忘れずに追加
- SVGコードはバッククォート `` ` `` で囲む
- viewBox, width, heightを必ず指定

### 3. Power Fxコードの追加

`getPowerAppsCode`関数内の`case`文に新しいケースを追加します。

```javascript
case 41: // 複数折れ線グラフ
    return `With(
    {
        imgWidth: Self.Width,
        imgHeight: Self.Height,
        // データ定義
    },
    // 処理ロジック
)`;
```

**追加場所：**
- 最後の`case`文の後、`default:`の前

---

## Power Fxコード作成のルール

### 1. With関数のネスト構造

Power Fxコードは基本的に以下の構造で記述します：

```powerfx
With(
    {
        // レベル1: 基本変数とデータ定義
        imgWidth: Self.Width,
        imgHeight: Self.Height,
        data: Table(...),
        color: "#667eea"
    },
    With(
        {
            // レベル2: 計算された変数（レベル1の変数を使用）
            leftMargin: imgWidth * 0.1,
            rightMargin: imgWidth * 0.067,
            topMargin: imgHeight * 0.1,
            bottomMargin: imgHeight * 0.133,
            dataMaxValue: Max(data, value)
        },
        With(
            {
                // レベル3: さらに依存する計算
                chartWidth: imgWidth - leftMargin - rightMargin,
                chartHeight: imgHeight - topMargin - bottomMargin
            },
            // SVG生成コード
        )
    )
)
```

### 2. マージンとレイアウトの標準値

- **leftMargin**: `imgWidth * 0.1` (10%)
- **rightMargin**: `imgWidth * 0.067` (6.7%)
- **topMargin**: `imgHeight * 0.1` (10%)
- **bottomMargin**: `imgHeight * 0.133` (13.3%)

### 3. グリッドラインの定義

```powerfx
gridLines: Table({level: 0}, {level: 25}, {level: 50}, {level: 75}, {level: 100})
```

グリッドラインは0-100の範囲で5段階が標準です。

### 4. データスケーリング

```powerfx
stepSize: If(dataMaxValue <= 10, 5,
    If(dataMaxValue <= 50, 10,
        If(dataMaxValue <= 100, 25, 50)))

maxValue: Round(dataMaxValue + stepSize,
    If(stepSize >= 50, -2,
        If(stepSize >= 10, -1, 0)))
```

### 5. データポイントの座標計算

#### 単一データ系列の場合：

```powerfx
x: leftMargin + (idx - 1) * (chartWidth / (CountRows(data) - 1))
y: topMargin + chartHeight - (value / maxValue * chartHeight)
```

#### 複数データ系列の場合：

```powerfx
lines: Table(
    {name: "Product A", color: "#667eea", data: Table(...)},
    {name: "Product B", color: "#f093fb", data: Table(...)}
)

dataCount: CountRows(First(lines).data)
dataMaxValue: Max(ForAll(lines, Max(data, value)), Value)
```

### 6. SVG生成パターン

#### グリッドライン：

```powerfx
Concat(
    gridLines,
    With(
        {
            gridY: topMargin + chartHeight - (level / 100 * chartHeight),
            gridValue: Round(maxValue * level / 100, 0)
        },
        "<line x1='" & leftMargin & "' y1='" & gridY & "' x2='" & (imgWidth - rightMargin) & "' y2='" & gridY & "' stroke='#e5e7eb' stroke-width='1'/>" &
        "<text x='" & (leftMargin - 5) & "' y='" & (If(level = 100, gridY + 12, gridY + 4)) & "' text-anchor='end' font-size='10' fill='#999'>" & gridValue & "</text>"
    )
)
```

#### 折れ線：

```powerfx
"<polyline points='" &
Concat(
    data,
    With(
        {
            x: leftMargin + (idx - 1) * (chartWidth / (CountRows(data) - 1)),
            y: topMargin + chartHeight - (value / maxValue * chartHeight)
        },
        x & "," & y
    ),
    " "  // スペース区切り
) &
"' fill='none' stroke='" & color & "' stroke-width='3'/>"
```

#### データポイント（円）：

```powerfx
Concat(
    data,
    With(
        {
            x: leftMargin + (idx - 1) * (chartWidth / (CountRows(data) - 1)),
            y: topMargin + chartHeight - (value / maxValue * chartHeight)
        },
        "<circle cx='" & x & "' cy='" & y & "' r='4' fill='" & color & "'/>"
    )
)
```

#### 複数系列の描画：

```powerfx
Concat(
    lines,
    "<polyline points='" &
    Concat(
        data,
        With(
            {
                x: leftMargin + (idx - 1) * (chartWidth / (dataCount - 1)),
                y: topMargin + chartHeight - (value / maxValue * chartHeight)
            },
            x & "," & y
        ),
        " "
    ) &
    "' fill='none' stroke='" & color & "' stroke-width='2.5'/>"
)
```

---

## よくあるエラーと修正方法

### 1. CountRows(data) - 1 が0になるエラー

**原因:** データポイントが1つしかない場合、除算ゼロエラーが発生します。

**修正方法:**
```powerfx
// 悪い例
chartWidth / (CountRows(data) - 1)

// 良い例
chartWidth / Max(CountRows(data) - 1, 1)
```

### 2. 複数系列の最大値取得エラー

**原因:** ForAllの結果を直接Maxに渡すと型エラーが発生します。

**修正方法:**
```powerfx
// 悪い例
Max(ForAll(lines, Max(data, value)))

// 良い例
Max(ForAll(lines, Max(data, value)), Value)
```

### 3. SVG要素の引用符エスケープエラー

**原因:** Power Fx内でSVGコードを生成する際、シングルクォートを使用する必要があります。

**修正方法:**
```powerfx
// 悪い例
"<svg xmlns=\"http://www.w3.org/2000/svg\">"

// 良い例
"<svg xmlns='http://www.w3.org/2000/svg'>"
```

### 4. データインデックスが0から始まるエラー

**原因:** Power Fxのテーブルインデックスは1から始まります。

**確認:**
```powerfx
data: Table(
    {idx: 1, value: 77},  // idxは1から開始
    {idx: 2, value: 62},
    ...
)
```

### 5. EncodeUrlの使用忘れ

**原因:** SVGコードをdata URIスキームで使用する際、EncodeUrlでエンコードする必要があります。

**修正方法:**
```powerfx
// 悪い例
"data:image/svg+xml," & "<svg>...</svg>"

// 良い例
"data:image/svg+xml," & EncodeUrl("<svg>...</svg>")
```

---

## コーディング規約

### 1. インデント

- With関数のネストは4スペースのインデント
- SVG要素の生成コードは適切に改行

### 2. 変数命名規則

- **camelCase**を使用
- 明確で説明的な名前を使用
  - 良い例: `dataMaxValue`, `chartWidth`, `leftMargin`
  - 悪い例: `dmv`, `w`, `lm`

### 3. コメント

- 各`case`文の先頭にサンプル名をコメントで記述
```javascript
case 41: // 複数折れ線グラフ
```

### 4. 色の選択

標準的な色パレット：

- **紫**: `#667eea`
- **ピンク**: `#f093fb`
- **緑**: `#4ade80`
- **青**: `#3b82f6`
- **黄**: `#facc15`
- **グレー**: `#e5e7eb` (グリッドライン用)
- **ダークグレー**: `#999` (ラベル用)

### 5. SVGのviewBox設定

- **小さいコンポーネント**: 120 x 120 (アイコン、プログレスバー)
- **カード**: 200 x 120, 250 x 100
- **グラフ**: 300 x 150, 280 x 140
- **複雑なグラフ**: 300 x 200

---

## アニメーション追加の例

SVGアニメーションを追加する場合は、マスクを使用したパターンが推奨されます：

```powerfx
"<defs><mask id='lineMask'><rect x='-" & imgWidth & "' y='0' width='" & imgWidth & "' height='" & imgHeight & "' fill='white'>" &
"<animate attributeName='x' from='-" & imgWidth & "' to='0' dur='" & animDuration & "' fill='freeze' repeatCount='" & repeatCount & "'/>" &
"</rect></mask></defs>"
```

アニメーション対象の要素に `mask='url(#lineMask)'` を追加します。

---

## テストチェックリスト

新しいサンプルを追加したら、以下を確認してください：

- [ ] ギャラリーに正しく表示される
- [ ] SVGが正しくレンダリングされる
- [ ] Power Fxコードがコピー可能
- [ ] 検索機能で見つけられる
- [ ] カテゴリーフィルタが正しく動作する
- [ ] モーダルが正しく開閉する
- [ ] レスポンシブデザインが機能する（異なるサイズでテスト）

---

## 参考リソース

- [Power Fx 公式ドキュメント](https://learn.microsoft.com/ja-jp/power-platform/power-fx/overview)
- [SVG 仕様](https://www.w3.org/TR/SVG2/)
- [Power Apps Image コントロール](https://learn.microsoft.com/ja-jp/power-apps/maker/canvas-apps/controls/control-image)

---

## 更新履歴

- **2026-01-12**: 初版作成、複数折れ線グラフサンプル追加に伴うガイドライン策定
