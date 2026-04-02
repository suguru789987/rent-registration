# 単月費用設定 - 地代家賃上書き機能 仕様書

## モックファイル

- `monthly-expense-mock.html` — ブラウザで開いて動作確認可能

---

## 1. 概要

単月費用設定モーダルから、地代家賃の月別上書きを可能にする。
上書き方法は「金額直接指定」と「計算条件変更」の2パターン。

---

## 2. UI遷移フロー

```
単月費用設定モーダル
│
├── 費目プルダウンで「地代家賃」を選択
│   │
│   ├── 通常の費目行が表示（費目 / デフォルト値 / 金額 / 削除）
│   │
│   └── 「▼ 地代家賃の計算条件を確認・変更する」ボタン表示
│       │
│       └── [クリック] トグルが開く（ボタンが「▲ 計算条件を閉じる」に変化）
│           │
│           ├── 情報バー: 現在の賃料設定とデフォルト計算額を表示
│           │
│           ├── ラジオ選択
│           │   ├── [金額を直接指定] ──→ 金額入力セクション表示
│           │   └── [計算条件を変更] ──→ パラメータ変更セクション表示
│           │
│           └── [トグルボタン再クリック] トグルが閉じる（入力値は保持）
│
├── 他の費目（固定人件費、資材費等）
│   └── 従来通り金額入力のみ
│
├── [費用設定値を変更しない] → モーダルを閉じる（変更破棄）
└── [保存する] → 全費目の設定を一括保存
```

---

## 3. UI状態一覧

### 3-1. トグル閉じ（初期状態）

| 要素 | 状態 |
|---|---|
| トグルボタン | `▼ 地代家賃の計算条件を確認・変更する`（border付きボタン） |
| 金額入力欄 | 有効（デフォルト計算額が入っている） |
| 上書きエリア | 非表示 |

### 3-2. トグル開き — 金額直接指定

| 要素 | 状態 |
|---|---|
| トグルボタン | `▲ 計算条件を閉じる`（open状態の背景色） |
| 情報バー | 賃料体系・条件選択・デフォルト計算額を表示 |
| ラジオ | 「金額を直接指定」が選択状態 |
| 入力欄 | 上書き金額を入力 |
| プレビュー | 入力金額とデフォルトとの差額を表示 |

### 3-3. トグル開き — 計算条件変更

| 要素 | 状態 |
|---|---|
| ラジオ | 「計算条件を変更」が選択状態 |
| パラメータ行 | チェックOFF=無効（グレーアウト）、チェックON=入力可能 |
| プレビュー | 変更パラメータで再計算した予測額・内訳・変更中バッジを表示 |

---

## 4. コンポーネント構成

```
MonthlyExpenseModal
├── ExpenseRow (費目ごとに繰り返し)
│   ├── CategorySelect (費目プルダウン)
│   ├── DefaultValue (デフォルト値表示)
│   ├── AmountInput (金額入力)
│   └── DeleteButton
│
├── RentOverrideToggle (地代家賃の場合のみ表示)
│   └── RentOverrideArea
│       ├── RentInfoBar (現在の賃料設定情報)
│       ├── OverrideTypeRadio (金額直接指定 / 計算条件変更)
│       ├── DirectAmountSection
│       │   ├── AmountInput
│       │   └── CalcPreview
│       └── ParamsSection
│           ├── ParamRow x 5 (固定賃料/歩合率/売上閾値/売上歩合対象率/最低保証額)
│           └── CalcPreview
│
├── AddExpenseButton
└── ModalFooter (変更しない / 保存する)
```

---

## 5. データモデル

### 5-1. 保存データ構造

```typescript
interface MonthlyExpenseSetting {
  month: string;              // "2026-03"
  expenses: ExpenseItem[];
}

interface ExpenseItem {
  category: string;           // "地代家賃" | "固定人件費" | ...
  amount: number | null;      // 通常費目の金額（地代家賃以外）

  // 地代家賃専用
  rent_override?: {
    override_type: "direct" | "params";

    // direct の場合
    direct_amount?: number;

    // params の場合（nullはデフォルト値を使用）
    fixed_rent?: number | null;
    commission_rate?: number | null;
    threshold?: number | null;
    sales_rate?: number | null;
    min_guarantee?: number | null;
  };
}
```

### 5-2. デフォルト値の参照元

| 項目 | 参照元 |
|---|---|
| 賃料体系 | 店舗情報 > 賃料設定 |
| 条件選択 | 店舗情報 > 賃料設定 |
| 固定賃料 | 店舗情報 > 賃料設定 |
| 歩合率 | 店舗情報 > 賃料設定 |
| 売上閾値 | 店舗情報 > 賃料設定 |
| 売上歩合対象率 | 店舗情報 > 賃料設定 |
| 最低保証額 | 店舗情報 > 賃料設定 |
| 月次売上予算 | 月次予算設定 |

---

## 6. 計算ロジック（優先度）

```
月別地代家賃 =
  1. rent_override.override_type === "direct"
     → rent_override.direct_amount

  2. rent_override.override_type === "params"
     → 変更パラメータ + デフォルト値で再計算
     → 計算式は賃料体系・条件選択に従う

  3. rent_override が存在しない
     → 年間デフォルト設定で計算（従来通り）
```

### 計算式（変動賃料の場合）

```
// パラメータ解決（上書き値 ?? デフォルト値）
fixed_rent    = override.fixed_rent    ?? default.fixed_rent
comm_rate     = override.commission_rate ?? default.commission_rate
threshold     = override.threshold      ?? default.threshold     ?? 0
sales_rate    = override.sales_rate     ?? default.sales_rate    ?? 100
min_guarantee = override.min_guarantee  ?? default.min_guarantee ?? 0

// 計算
target_sales = sales * sales_rate / 100
commission   = MAX(target_sales - threshold, 0) * comm_rate / 100

// 条件選択で分岐
if (合算) subtotal = fixed_rent + commission
if (最大値選択) subtotal = MAX(fixed_rent, commission)

// 最低保証
rent = MAX(subtotal, min_guarantee)
```

---

## 7. UI操作 → データ変換マッピング

| UI操作 | データへの反映 |
|---|---|
| トグルを開かずに保存 | rent_override = undefined（上書きなし） |
| 金額直接指定で保存 | override_type = "direct", direct_amount = 入力値 |
| 計算条件変更で保存 | override_type = "params", チェックONの項目のみ値をセット |
| チェックOFFの項目 | null（デフォルト値にフォールバック） |

---

## 8. バリデーション

| 項目 | ルール |
|---|---|
| 金額直接指定 | 0以上の数値必須 |
| 固定賃料 | 0以上の数値 |
| 歩合率 | 0〜100の数値 |
| 売上閾値 | 0以上の数値 |
| 売上歩合対象率 | 0〜100の数値 |
| 最低保証額 | 0以上の数値 |
| チェックONで空欄 | エラー（値を入力するか、チェックを外してください） |

---

## 9. 表示条件

| 条件 | 表示 |
|---|---|
| 費目が「地代家賃」 | トグルボタンを表示 |
| 費目が「地代家賃」以外 | トグルボタン非表示（従来の金額入力のみ） |
| 店舗の賃料設定が「固定賃料」 | トグル内のパラメータは固定賃料のみ |
| 店舗の賃料設定が「変動賃料」 | トグル内のパラメータ5項目を表示 |
| 店舗の賃料設定が「段階別歩合」 | トグル内は固定賃料・売上歩合対象率・最低保証額のみ（段階設定は単月では変更不可） |

---

## 10. プレビュー計算

| モード | プレビュー内容 |
|---|---|
| 金額直接指定 | 入力金額、デフォルトとの差額 |
| 計算条件変更 | 月次売上予算ベースの予測計算結果、内訳、変更中のパラメータをバッジ表示 |

プレビューはリアルタイムで更新（入力値変更時に即座に再計算）。
