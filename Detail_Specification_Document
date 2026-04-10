# money-log-app 詳細設計書

作成日: 2026年4月10日
フェーズ: 詳細設計
前提文書: 基本設計書 (2026年4月9日)

---

## 目次

1. [前提・対応環境](#1-前提対応環境)
2. [エンティティ設計](#2-エンティティ設計)
3. [画面構成・ナビゲーション](#3-画面構成ナビゲーション)
4. [各画面の仕様](#4-各画面の仕様)
5. [サブスク自動計上ロジック](#5-サブスク自動計上ロジック)
6. [変動固定費（毎月費用）バナー・アラートロジック](#6-変動固定費毎月費用バナーアラートロジック)
7. [家計簿一覧フィルタ](#7-家計簿一覧フィルタ)
8. [SwiftData マイグレーション戦略](#8-swiftdata-マイグレーション戦略)
9. [通知設計](#9-通知設計)
10. [v1.1 以降の拡張予定](#10-v11-以降の拡張予定)

---

## 1. 前提・対応環境

| 項目 | 内容 |
|------|------|
| iOS 最低バージョン | 18.0 |
| 永続化フレームワーク | SwiftData |
| iCloud 同期 | CloudKit private database（`ModelConfiguration` で設定） |
| 外部通信 | なし（MVP 段階） |
| メインターゲット | iPhone |
| ナビゲーション方式 | タブバー形式 |

---

## 2. エンティティ設計

### 2.1 全体方針

- リレーションは全て**双方向オプショナル**
- 削除ルールは全て **`.nullify`**（カスケードなし）
- CloudKit 制約に対応するため、非リレーションフィールドは**オプショナルまたはデフォルト値付き**
- `@Attribute(.unique)` は CloudKit 同期環境では使用不可のため、使わない
- 金額の型は **`Int`**（円単位、小数なし）
- Enum の rawValue は **`String`** で統一（将来の値追加に強い）
- 最初から **`VersionedSchema`（`SchemaV1`）** を使って定義する

### 2.2 エンティティ一覧

| エンティティ | 役割 |
|-------------|------|
| Transaction | 個々の支出・収入（都度入力・サブスク自動計上・毎月費用確定後を含む） |
| Category | 支出・収入それぞれのカテゴリ（プリセット＋ユーザ追加） |
| PaymentMethod | 現金・PayPay・クレジットカードなど（プリセット＋ユーザ追加） |
| Subscription | 定期的に自動計上するサブスクの定義（マスター） |
| VariableFixedCost | 指定日に入力促しを出す毎月費用の定義（マスター） |
| IgnoredVariableFixedCost | 毎月費用の「無視」操作を記録するマーカー |

### 2.3 Transaction

| プロパティ名 | 型 | 必須/任意 | 説明 |
|---|---|---|---|
| id | UUID | 必須 | 主キー（自動生成） |
| kind | String (enum: "expense" / "income") | 必須 | 支出 or 収入 |
| date | Date | 必須 | 発生日（時刻は 00:00 に正規化） |
| title | String? | 任意 | 内容（ユーザが入力する説明） |
| amount | Int | 必須 | 金額（円単位、正の整数） |
| category | Category? | 必須（UI で強制） | カテゴリへの参照 |
| paymentMethod | PaymentMethod? | 支出のみ必須 | 支払い方法への参照（収入時は nil） |
| memo | String? | 任意 | メモ |
| isFromSubscription | Bool | 必須（デフォルト: false） | サブスク自動計上由来か |
| sourceKey | String? | 任意 | 由来キー。サブスク自動計上: `"sub-{id}-YYYYMM"`、毎月費用入力: `"vfc-{id}-YYYYMM"`。重複検知に使用 |
| groupId | UUID? | 任意 | 将来のグループ共有機能用。MVP では未使用 |

**備考:**
- `sourceKey` はサブスク自動計上と毎月費用入力の両方で使い、プレフィックスで区別する
- `kind` は `TransactionKind` enum（rawValue: String）で管理
- クレジットカード支払いは**利用日で計上**する（引き落とし日への繰り延べなし）

### 2.4 Category

| プロパティ名 | 型 | 必須/任意 | 説明 |
|---|---|---|---|
| id | UUID | 必須 | 主キー（自動生成） |
| name | String | 必須 | カテゴリ名 |
| kind | String (enum: "expense" / "income") | 必須 | 支出用 or 収入用 |
| iconName | String | 必須（デフォルト: "tag"） | SF Symbols のアイコン名 |
| displayOrder | Int | 必須（デフォルト: 0） | 表示順（将来の並べ替え用） |
| isPreset | Bool | 必須（デフォルト: false） | プリセット由来か |
| isOther | Bool | 必須（デフォルト: false） | 「その他」カテゴリか（削除不可マーカー） |

**プリセット:**

| 名前 | kind | iconName | isOther |
|---|---|---|---|
| 食費 | expense | fork.knife | false |
| 日用品 | expense | basket | false |
| 交通費 | expense | tram.fill | false |
| 娯楽 | expense | gamecontroller | false |
| サブスク | expense | arrow.clockwise | false |
| その他 | expense | ellipsis.circle | true |
| 給与 | income | yensign.circle | false |
| ボーナス | income | gift | false |
| その他 | income | ellipsis.circle | true |

**アイコン選択:**
- ユーザ追加カテゴリでは、家計簿向けに厳選した30〜40個の SF Symbols をグリッドピッカーで選択可能
- MVP ではアイコンの色はアプリのテーマカラーで統一（カテゴリ別の色選択は v1.1）

**削除ルール:**
- `isOther == true` のカテゴリは削除不可
- それ以外のプリセットカテゴリはユーザが削除可能
- 削除時、そのカテゴリを使用中の Transaction / Subscription / VariableFixedCost のカテゴリ参照を「その他」に付け替え、確認アラートを表示
- 同一 `kind` 内で名前の重複は不可

### 2.5 PaymentMethod

| プロパティ名 | 型 | 必須/任意 | 説明 |
|---|---|---|---|
| id | UUID | 必須 | 主キー（自動生成） |
| name | String | 必須 | 支払い方法名 |
| displayOrder | Int | 必須（デフォルト: 0） | 表示順 |
| isPreset | Bool | 必須（デフォルト: false） | プリセット由来か |

**プリセット:**

| 名前 |
|---|
| 現金 |
| PayPay |
| クレジットカード |

**削除ルール:**
- 全ての支払い方法が削除可能（プリセットも含む）
- ただし**最後の1件は削除不可**（支出登録時に選択肢がなくなるのを防ぐ）
- 削除時、使用中の Transaction / Subscription / VariableFixedCost の支払い方法参照を `nil`（未設定）に変更
- 一覧表示では `nil` を「未設定」と表示
- 同名の重複は不可

### 2.6 Subscription

| プロパティ名 | 型 | 必須/任意 | 説明 |
|---|---|---|---|
| id | UUID | 必須 | 主キー（自動生成） |
| name | String | 必須 | サブスク名 |
| amount | Int | 必須 | 金額（円単位） |
| chargeDay | Int | 必須 | 毎月の計上日（1〜31） |
| category | Category? | 任意 | カテゴリへの参照 |
| paymentMethod | PaymentMethod? | 任意 | 支払い方法への参照 |
| lastPostedMonth | Date? | 任意 | 最終計上年月（その月の1日 00:00 を格納） |
| pausedUntil | Date? | 任意 | 休会期限（MVP では UI なし、v1.1 で対応） |

**初期値:**
- 新規作成時の `lastPostedMonth` は**登録月の前月の1日**に設定
- これにより、当月の計上日を過ぎていれば即座に計上される

**月末日の扱い:**
- `chargeDay = 31` の場合、31日が存在しない月はその月の最終日に繰り上げて計上
- 計算: `min(chargeDay, その月の最終日)`

### 2.7 VariableFixedCost

| プロパティ名 | 型 | 必須/任意 | 説明 |
|---|---|---|---|
| id | UUID | 必須 | 主キー（自動生成） |
| name | String | 必須 | 毎月費用名 |
| reminderDay | Int | 必須 | 毎月の入力促し日（1〜31） |
| category | Category? | 任意 | カテゴリへの参照（入力時の初期値） |
| paymentMethod | PaymentMethod? | 任意 | 支払い方法への参照（入力時の初期値） |

**備考:**
- 金額は登録しない（毎月変動するため）
- 入力済みかどうかは、Transaction の `sourceKey` を検索して判定（VariableFixedCost 自体には入力済み状態を持たない）
- UI での呼称は「毎月費用」（内部コードは `VariableFixedCost` のまま）

### 2.8 IgnoredVariableFixedCost

| プロパティ名 | 型 | 必須/任意 | 説明 |
|---|---|---|---|
| id | UUID | 必須 | 主キー（自動生成） |
| variableFixedCostId | UUID | 必須 | 対象の VariableFixedCost の ID |
| yearMonth | Date | 必須 | 無視した対象月（その月の1日 00:00 を格納） |
| ignoredAt | Date | 必須 | 無視操作を行った日時 |

**備考:**
- 毎月費用の過去月アラートで「無視」を選んだ際に作成される
- 取り消し UI は MVP では作らない（v1.1 で対応予定）
- 自動掃除は MVP では不要（年数件しか増えないため）

### 2.9 エンティティ関連図

```
Transaction ──→ Category (optional, .nullify)
Transaction ──→ PaymentMethod (optional, .nullify)

Subscription ──→ Category (optional, .nullify)
Subscription ──→ PaymentMethod (optional, .nullify)

VariableFixedCost ──→ Category (optional, .nullify)
VariableFixedCost ──→ PaymentMethod (optional, .nullify)

IgnoredVariableFixedCost ──→ (VariableFixedCost を UUID で参照)
```

- Subscription から Transaction への直接参照は持たない（`sourceKey` で間接的に紐付く）
- VariableFixedCost から Transaction への直接参照も持たない（同上）
- 全リレーションは双方向オプショナル、削除ルールは `.nullify`

---

## 3. 画面構成・ナビゲーション

### 3.1 タブ構成

| タブ | アイコン | 役割 |
|------|---------|------|
| ホーム | house | 当月のサマリ表示（サブタブ切り替え） |
| 一覧 | list.bullet | 検索・フィルタによる過去レコードの横断参照 |
| [+] | plus.circle | 支出追加 / 収入追加のアクションシート |
| サブスク | arrow.clockwise | サブスクと毎月費用の管理 |
| 収入 | yensign.circle | 収入の登録・一覧 |

- [+] ボタンはタップでアクションシートを表示し、「支出追加」「収入追加」を選択
- 設定はハンバーガーメニュー経由

### 3.2 ハンバーガーメニュー構成

**ショートカット:** ホーム / 一覧 / サブスク / 収入
**管理:** 毎月費用 / カテゴリ管理 / 支払い方法管理
**データ:** エクスポート（CSV / JSON）
**設定:** アプリ設定 / アプリについて

### 3.3 ホーム画面のサブタブ

| サブタブ | 表示内容 |
|---------|---------|
| 日別（デフォルト） | リスト / カレンダーの切り替え |
| カテゴリ別 | 上半分に円グラフ、下半分にカテゴリ別リスト（当月のみ） |
| 月間 | 月の合計支出、1日平均、月ごとの推移グラフ |

---

## 4. 各画面の仕様

### 4.1 ホーム画面

**変動固定費バナー:**
- ホーム画面上部に最大2件表示 + 残件数インジケータ
- タップで支出登録モーダル（`sourceKey` 自動付与）
- 入力されたらバナーは消える

**過去月未入力アラート帯:**
- バナーの上に「⚠️ 未入力 N件」のコンパクトな帯を表示（高さ約40pt）
- タップで未入力一覧画面に遷移
- 表示条件: 過去2ヶ月（当月と合わせて計3ヶ月の追跡範囲）で未入力かつ未無視の毎月費用がある場合

**日別タブ - カレンダー表示:**
- 自前実装（`LazyVGrid` 7列）
- セル内に日付と当日の支出合計を表示
- 設定画面の「週の始まり（日曜/月曜）」と連動
- 常に6行表示（空セルで埋める）
- 前月/翌月の日付は薄く表示

**月間タブ - 推移グラフ:**
- 直近6ヶ月固定（MVP）。構造は可変対応にしておく
- 棒グラフ（`BarMark`）を使用（Swift Charts）
- データがない月はゼロ円のバーとして表示
- タップでのドリルダウンは MVP では実装しない

### 4.2 未入力一覧画面

過去月の未入力毎月費用をリスト表示する専用画面。

```
[ナビゲーションバー: 未入力一覧]

⚠️ 過去2ヶ月で N 件の未入力があります

┌─────────────────────────┐
│ {費用名}                   │
│ {YYYY年M月分}（促し日: M/DD）│
│ [入力する] [無視]            │
└─────────────────────────┘
（以下同様のカード）
```

| 操作 | 挙動 |
|------|------|
| 入力する | 支出登録モーダル（`sourceKey` 自動付与、日付は対象月の促し日が初期値） |
| 無視 | 確認アラート → `IgnoredVariableFixedCost` を作成 → 該当行が消える |

### 4.3 家計簿一覧画面

支出のみ表示。収入は収入タブで見る。

**フィルタ構成:**

| 種類 | 表示方式 | 内容 |
|------|---------|------|
| 年月フィルタ | 常時表示 | プリセット（今月/先月/直近3ヶ月/直近6ヶ月/今年/カスタム） |
| カテゴリ | 常時表示 | カテゴリ選択 |
| 詳細フィルター | 折りたたみ | 金額範囲・キーワード・支払い方法 |

- 合計カードのラベルはカテゴリ選択に応じて動的変化
- キーワード検索は `localizedStandardContains` を使用（ひらがな/カタカナ同一視）

### 4.4 サブスク/毎月費用画面

**サブスクリスト:**
- 各サブスクの名前・金額・計上日を表示
- 並べ替え機能あり

**毎月費用リスト:**
- 各毎月費用の名前・入力促し日を表示
- 未入力/入力済みバッジ（未入力=赤系、入力済=緑系+金額併記）
- 並べ替え機能あり

**毎月費用の合計表示:**
- 大きく先月実績、小さく今月経過
- 「先月 ¥X,XXX / 今月これまで ¥X,XXX（未入力N件）」

### 4.5 支出/収入登録モーダル（共通View）

同じ View を `kind` 引数で分岐。

**フィールド順序:**
1. 内容（`title`）— 任意
2. 金額（`amount`）— 必須、右横に電卓アイコンボタン
3. 日付（`date`）— 必須、`DatePicker(.graphical)` でカレンダー風
4. カテゴリ（`category`）— 必須
5. 支払方法（`paymentMethod`）— 支出のみ必須
6. メモ（`memo`）— 任意

**バリデーション:**
- 保存ボタンは必須フィールドが埋まるまで無効化
- 空の必須フィールドにエラー表示

**モーダルの閉じ方:**
- 右上の ×ボタンは不採用。下部に「閉じる」ボタンのみ（親指操作のため）

**編集モード時:**
- 削除ボタンあり（確認アラート必須）
- スワイプ削除も実装（MVP では確認アラート付き）

### 4.6 電卓モーダル

**表示方式:** `.sheet` + `.presentationDetents([.medium])`（画面半分の高さ）

**機能範囲:** 四則演算のみ（`+ - × ÷`）

| 項目 | 仕様 |
|------|------|
| 表示桁数 | 最大8桁（99,999,999 ≒ 約1億） |
| 小数 | 非対応（円単位、Int） |
| 演算順序 | 左から順に評価（簡易電卓の慣習） |
| エラー処理 | ゼロ除算は "Error" 表示、AC で復帰 |
| ボタン | 0〜9, +, -, ×, ÷, =, AC（全クリア）, C（直前クリア）, ⌫（1文字削除） |
| 反映 | 「この金額を使う」ボタンで結果を金額欄に反映 |
| キャンセル | シートを下スワイプで閉じれば反映なし |

**State 管理:**
- 「現在の表示値」「直前の演算子」「累積値」の3つを `@State` で管理
- `=` のあとに数字を押したら新しい計算を開始

### 4.7 カテゴリ管理画面

```
[ナビゲーションバー: カテゴリ管理]
[Picker: 支出 / 収入]  ← セグメント切り替え

カテゴリリスト（displayOrder 順）
  各カテゴリ: アイコン + 名前 + (プリセット)表示

  [+ カテゴリを追加]
```

| 操作 | 挙動 |
|------|------|
| タップ | 編集モーダル（名前・アイコン変更） |
| スワイプ削除 | `isOther == true` なら無効。それ以外は削除確認アラート |
| + ボタン | 追加モーダル（名前入力 + アイコン選択グリッド） |

**アイコン選択ピッカー:**
- カテゴリ追加/編集モーダル内にインライン展開
- 家計簿向けに厳選した30〜40個の SF Symbols をグリッド表示
- デフォルトアイコン: `"tag"`

**削除時のフロー:**

```
ユーザがカテゴリをスワイプ削除
    ↓
使用中チェック:
  そのカテゴリを使用している Transaction / Subscription / VariableFixedCost が存在するか?
    ↓
[存在する場合]
  アラート:
    「"{カテゴリ名}" は X 件の支出で使用中です。
     削除すると、これらのカテゴリは "その他" に変更されます。
     削除しますか?」
  [削除する] →
    該当 Transaction のカテゴリを「その他」に付け替え
    該当 Subscription のカテゴリを「その他」に付け替え
    該当 VariableFixedCost のカテゴリを「その他」に付け替え
    カテゴリを削除
  [キャンセル] → 何もしない

[存在しない場合]
  アラート: 「"{カテゴリ名}" を削除しますか?」
  [削除する] → 削除
  [キャンセル] → 何もしない
```

**バリデーション:**
- 名前が空文字・スペースのみの場合は保存不可
- 同一 `kind` 内で名前の重複不可

### 4.8 支払い方法管理画面

カテゴリ管理画面とほぼ同じ構成。

```
[ナビゲーションバー: 支払い方法管理]

支払い方法リスト（displayOrder 順）
  各支払い方法: 名前 + (プリセット)表示

  [+ 支払い方法を追加]
```

| 操作 | 挙動 |
|------|------|
| タップ | 編集モーダル（名前変更のみ） |
| スワイプ削除 | 最後の1件なら無効。それ以外は削除確認アラート |
| + ボタン | 追加モーダル（名前入力のみ） |

**削除時の挙動:**
- 使用中の Transaction / Subscription / VariableFixedCost の支払方法参照を `nil`（未設定）に変更
- 一覧表示では `nil` を「未設定」と表示

### 4.9 通知設定画面

```
[ナビゲーションバー: 通知設定]

毎月費用の入力通知
  [Toggle: ON/OFF]

通知時刻
  [時刻ピッカー]  ← Toggle が OFF なら非活性
  デフォルト: 20:00

[注意テキスト（小さく）]
  通知が届かない場合は、端末の設定 >
  通知 > money-log-app で通知を許可してください。
  [設定を開く]  ← UIApplication.openSettingsURLString
```

| 操作 | 挙動 |
|------|------|
| Toggle ON | 2段階パーミッション → 全 VariableFixedCost のローカル通知をスケジュール |
| Toggle OFF | スケジュール済みの全ローカル通知を削除 |
| 時刻変更 | 全通知を削除 → 新しい時刻で再スケジュール |

### 4.10 エクスポート画面

```
[ナビゲーションバー: データエクスポート]

エクスポート形式
  [Picker: CSV / JSON]

期間
  [Picker: 全期間 / 今年 / カスタム]
  (カスタム選択時 → 開始月・終了月ピッカー表示)

対象
  [Toggle: 支出を含む  ON]
  [Toggle: 収入を含む  ON]

[エクスポート]ボタン
```

- エクスポート後は iOS のシェアシート（`ShareLink`）を表示
- 非同期実行（`Task { }`）+ プログレスインジケータ

**CSV 仕様:**

| 項目 | 仕様 |
|------|------|
| 文字コード | UTF-8（BOM 付き: `\u{FEFF}`） |
| 改行コード | `\r\n`（Excel 互換） |
| 日付フォーマット | `yyyy-MM-dd`（ISO 8601） |
| 種別 | `支出` / `収入` |
| 金額 | カンマなしの整数 |
| フィールド内カンマ | ダブルクォートで囲む（CSV 標準） |
| ファイル名 | `money-log-export-yyyy-MM-dd.csv` |

**CSV カラム:** 日付, 種別, 内容, 金額, カテゴリ, 支払方法, メモ

**JSON 仕様:**

```json
{
  "exportedAt": "ISO 8601 形式",
  "transactions": [
    {
      "date": "yyyy-MM-dd",
      "kind": "expense | income",
      "title": "...",
      "amount": 580,
      "category": "食費",
      "paymentMethod": "PayPay",
      "memo": null,
      "sourceKey": null
    }
  ]
}
```

- JSON は将来のインポート機能のためのバックアップ用途
- `sourceKey` 等の内部フィールドも含める
- ファイル名: `money-log-export-yyyy-MM-dd.json`

### 4.11 設定画面項目

| セクション | 項目 |
|-----------|------|
| 表示 | テーマカラー、週の始まり（日曜/月曜） |
| 通知 | 毎月費用の入力通知 ON/OFF、通知時刻（→ 通知設定画面へ遷移） |
| データ | iCloud 同期状態、手動バックアップ（→ エクスポート画面へ遷移） |
| アプリについて | バージョン、プライバシーポリシー |

---

## 5. サブスク自動計上ロジック

### 5.1 アルゴリズム方針

**`lastPostedMonth`（最終計上年月）主導 + `sourceKey` による重複ガード** のハイブリッド方式。

- `lastPostedMonth` が自動計上の進行管理を担う主たる状態
- `sourceKey` は CloudKit 同期で他端末が先に計上した場合の重複検知ガード
- ユーザが自動計上された Transaction を削除しても、`lastPostedMonth` は進んだままなので再計上されない

### 5.2 実行タイミング

- アプリ起動時（`scenePhase = .active`）に実行
- CloudKit 同期完了を待つ確実な API は存在しないため、以下の2段構えで対応:
  - 起動直後に1回目の判定（ローカル DB 基準）
  - 数秒後に2回目の再判定（同期後の状態で再チェック）
- `sourceKey` による重複検知があるため、2回実行しても二重計上は起きない

### 5.3 処理フロー（疑似コード）

```
入力:
  subscriptions: 全 Subscription の配列
  today: Calendar.current.startOfDay(for: Date())
  currentMonth: today の年月を抽出した Date（その月の1日 00:00）

処理:

FOR EACH subscription IN subscriptions:

  last = subscription.lastPostedMonth

  // 初期値の設定（新規登録時は nil）
  IF last == nil:
    last = 登録月の前月の1日
    // 例: 4月に登録 → last = 2026-03-01

  // 未計上月を順次処理するループ
  WHILE last < currentMonth:

    targetMonth = Calendar.date(byAdding: .month, value: 1, to: last)
    // 例: last = 2026-03-01 → targetMonth = 2026-04-01

    targetDay = clampDay(subscription.chargeDay, targetMonth)
    // 例: chargeDay = 31, targetMonth = 2月 → targetDay = 28

    targetDate = targetMonth の targetDay 日
    // 例: 2026-04-15

    // まだ計上日に達していなければループを抜ける
    IF today < targetDate:
      BREAK

    // sourceKey の生成
    sourceKey = "sub-\(subscription.id)-\(formatYYYYMM(targetMonth))"
    // 例: "sub-ABC123-202604"

    // 重複ガード: 同じ sourceKey の Transaction が既に存在するか
    IF Transaction に同じ sourceKey が存在する:
      // 他端末が先に計上済み → lastPostedMonth だけ進めてスキップ
      last = targetMonth
      subscription.lastPostedMonth = last
      CONTINUE

    // 新規 Transaction の作成
    transaction = Transaction(
      kind: .expense,
      date: targetDate,
      title: subscription.name,
      amount: subscription.amount,
      category: subscription.category,
      paymentMethod: subscription.paymentMethod,
      memo: nil,
      isFromSubscription: true,
      sourceKey: sourceKey,
      groupId: nil
    )
    modelContext.insert(transaction)

    // 進行状態の更新
    last = targetMonth
    subscription.lastPostedMonth = last

  END WHILE

END FOR

// ループ完了後に1回だけ保存
// ループ中に毎回 save() を呼ぶと CloudKit への push が大量に走るため
try? modelContext.save()
```

### 5.4 補助関数 clampDay

月末日を超える計上日を、その月の最終日に丸める関数。

```
clampDay(指定日: Int, 対象月: Date) -> Int:
  最終日 = Calendar.current.range(of: .day, in: .month, for: 対象月)!.upperBound - 1
  RETURN min(指定日, 最終日)
```

例:
- `clampDay(31, 2026年2月)` → `28`
- `clampDay(31, 2026年4月)` → `30`
- `clampDay(15, 2026年4月)` → `15`（変更なし）

### 5.5 ケース別動作表

| # | ケース | lastPostedMonth | 今日 | 動作 | 結果 |
|---|--------|----------------|------|------|------|
| 1 | 初回起動（サブスク 0 件） | — | — | ループに入らず終了 | 何もしない |
| 2 | 4/9 に Netflix 新規登録（計上日: 5日） | 2026-03-01 | 04-09 | 4月分の計上日(4/5)は過ぎている → 計上 | last = 2026-04-01 |
| 3 | 4/9 に Netflix 新規登録（計上日: 15日） | 2026-03-01 | 04-09 | 4月分の計上日(4/15)はまだ → BREAK | last = 2026-03-01（変更なし） |
| 4 | 2ヶ月放置（計上日: 5日） | 2026-02-01 | 04-09 | 3月分(3/5)計上 → 4月分(4/5)計上 | last = 2026-04-01 |
| 5 | 計上日31、2月に突入 | 2026-01-01 | 03-01 | 対象日 = 2/28 に丸め → 計上 | last = 2026-02-01 |
| 6 | 別端末が先に4月分を計上済み | 2026-03-01 | 04-09 | sourceKey 一致 → スキップ、last だけ進める | last = 2026-04-01 |
| 7 | ユーザが4月分を削除後の再起動 | 2026-04-01 | 04-15 | last >= currentMonth → WHILE に入らず終了 | 再計上されない ✓ |
| 8 | マスター自体を削除 | — | — | FOR EACH に含まれない | 既存 Transaction は残る |

### 5.6 設計上の注意点

**「今日」の計算:**
`Calendar.current.startOfDay(for: Date())` を使い、時刻ぶんのオフバイワンを防ぐ。

**タイムゾーン:**
`Calendar.current`（端末のタイムゾーン）をそのまま使用する。家計簿アプリではユーザの体感に合わせるのが正しいため、固定タイムゾーンは不採用。

**WHILE の脱出条件:**
`last < currentMonth` であること。`<=` にすると、当月分を計上した直後に `last == currentMonth` となっても抜けられず無限ループする。

**カテゴリ/支払方法の nil ガード:**
削除ルールが `.nullify` なので、参照先のカテゴリ/支払方法が削除されると nil になる。自動計上ロジックでは `subscription.category == nil` の場合にスキップ + ログ出力のガードを入れる。

**`lastPostedMonth` の型:**
`Date?` 型で、その月の1日 00:00 を格納する。`Calendar.date(byAdding: .month, value: 1, to: ...)` で安全に1ヶ月加算できるため、年月専用型よりも Calendar API との親和性が高い。

---

## 6. 変動固定費（毎月費用）バナー・アラートロジック

### 6.1 サブスクとの違い

| | サブスク | 毎月費用 |
|---|---|---|
| 金額 | 固定 | 変動（ユーザが毎回入力） |
| 計上方法 | アプリが自動で Transaction を作成 | ユーザが促されて手動入力 |
| 状態管理 | `lastPostedMonth` で進行管理 | Transaction の `sourceKey` 検索で入力済み判定 |
| UI | 通知のみ | バナー + 通知 + アラート帯 |

### 6.2 バナーとアラートの役割分担

| | バナー | アラート帯 + 専用画面 |
|---|---|---|
| 対象 | **当月**の未入力 | **過去月**（先月・先々月）の未入力 |
| 表示位置 | ホーム画面上部（最大2件 + 残件数） | ホーム最上部のコンパクトな帯（タップで専用画面へ） |
| トーン | 促し（柔らかい） | 警告（目立つ） |
| アクション | タップで入力モーダル | 入力 or 無視 |
| 消える条件 | 入力されたら消える | 入力 or 無視されたら消える |
| 追跡期間 | 当月のみ | 過去2ヶ月（当月含め合計3ヶ月） |

### 6.3 「入力済み」の判定方式

Transaction の `sourceKey` を検索して判定する。

- `sourceKey = "vfc-{vfcId}-{YYYYMM}"` の Transaction が存在する → 入力済み
- 存在しない → 未入力

**重要な仕様:** バナー/アラート経由で入力した時のみ `sourceKey` が振られる。ユーザが促し日より前に自発的に入力した場合は `sourceKey` が振られないため、「入力済み」とは認識されない（バナーは表示される）。これはユーザが自分で促し日を設定しているため、ユーザの責任とする。

### 6.4 バナー表示ロジック（疑似コード）

```
入力:
  allVFC: 全 VariableFixedCost の配列
  today: Calendar.current.startOfDay(for: Date())
  currentYYYYMM: 当月の YYYYMM 文字列

処理:
  unfilled = []  // 未入力の毎月費用リスト

  FOR EACH vfc IN allVFC:
    targetDay = clampDay(vfc.reminderDay, 当月)
    targetDate = 当月の targetDay 日

    // まだ促し日に達していなければスキップ
    IF today < targetDate:
      CONTINUE

    sourceKey = "vfc-\(vfc.id)-\(currentYYYYMM)"
    existing = Transaction.fetch(where: sourceKey == sourceKey)

    IF existing == nil:
      unfilled.append(vfc)

  END FOR

  // バナー表示
  表示件数 = min(unfilled.count, 2)
  残件数 = unfilled.count - 表示件数
  // 表示: バナー2件 + 「他{残件数}件」インジケータ
```

### 6.5 過去月アラート判定ロジック（疑似コード）

```
入力:
  allVFC: 全 VariableFixedCost の配列
  today: Calendar.current.startOfDay(for: Date())
  currentMonth: 当月の1日

処理:
  alerts = []  // (vfc, 対象月) のタプル配列

  FOR EACH vfc IN allVFC:
    FOR offset IN 1...2:  // 先月(1)と先々月(2)
      targetMonth = Calendar.date(byAdding: .month, value: -offset, to: currentMonth)
      targetDay = clampDay(vfc.reminderDay, targetMonth)
      targetDate = targetMonth の targetDay 日

      IF today < targetDate:
        CONTINUE

      sourceKey = "vfc-\(vfc.id)-\(formatYYYYMM(targetMonth))"

      // 入力済みチェック
      existing = Transaction.fetch(where: sourceKey == sourceKey)
      IF existing != nil:
        CONTINUE

      // 無視済みチェック
      ignored = IgnoredVariableFixedCost.fetch(
        where: variableFixedCostId == vfc.id AND yearMonth == targetMonth
      )
      IF ignored != nil:
        CONTINUE

      alerts.append((vfc, targetMonth))
    END FOR
  END FOR

  // アラート帯の表示
  IF alerts.count > 0:
    帯を表示: "⚠️ 未入力 \(alerts.count)件"
  ELSE:
    帯を非表示
```

### 6.6 バナー/アラートから入力したときの自動設定

| フィールド | 初期値 |
|-----------|-------|
| 内容（title） | vfc.name |
| カテゴリ | vfc.category |
| 支払方法 | vfc.paymentMethod |
| 日付 | 当月の促し日（バナー経由）/ 対象月の促し日（アラート経由） |
| sourceKey | `"vfc-{vfcId}-{YYYYMM}"`（保存時に自動付与） |

ユーザはモーダル内で全フィールドを自由に変更可能。カテゴリを変更しても `sourceKey` は付与される（=入力済み扱い）。

### 6.7 同一 vfc のバナーとアラートの重複

同じ毎月費用が当月バナーと過去月アラートの両方に表示されることがある（例: 先月も今月も電気代が未入力）。これは仕様として許容する。現実をそのまま反映しており、ユーザの認知とズレない。

---

## 7. 家計簿一覧フィルタ

### 7.1 期間プリセット定義

全て**半開区間 `[start, end)`** で定義する。`end` の日は含まない。

| プリセット | start | end |
|-----------|-------|-----|
| 今月 | 今月の1日 | 来月の1日 |
| 先月 | 先月の1日 | 今月の1日 |
| 直近3ヶ月 | 2ヶ月前の1日 | 来月の1日 |
| 直近6ヶ月 | 5ヶ月前の1日 | 来月の1日 |
| 今年 | 今年の1月1日 | 来年の1月1日 |
| カスタム | ユーザ指定の開始月の1日 | ユーザ指定の終了月の翌月1日 |

**半開区間を使う理由:**
`<=` で `月末 23:59:59` と比較するとナノ秒精度のスキマで取りこぼしが起きる。`<` で「次の区間の開始日」と比較することでスキマをゼロにする。

**「直近3ヶ月」「直近6ヶ月」の解釈:**
日数ベース（90日等）ではなく、月境界で揃える。今月を含む過去N月分。家計簿は月単位で考えるため、月境界が直感的。

### 7.2 期間プリセットの実装

```swift
enum DateRangePreset {
    case thisMonth, lastMonth, last3Months,
         last6Months, thisYear
    case custom(start: Date, end: Date)

    // now を引数で受け取ることでテスト可能にする
    func dateRange(now: Date = Date()) -> (start: Date, end: Date) {
        let calendar = Calendar.current
        switch self {
        case .thisMonth:
            let start = calendar.date(from: calendar.dateComponents([.year, .month], from: now))!
            let end = calendar.date(byAdding: .month, value: 1, to: start)!
            return (start, end)
        // 他のケースも同様に Calendar API で計算
        case .custom(let start, let end):
            return (start, end)
        }
    }
}
```

### 7.3 フィルタ実装方針: 2段構え方式

| レイヤ | フィルタ条件 | 方式 | 理由 |
|--------|-----------|------|------|
| DB レベル | 期間 + kind（支出のみ） | `#Predicate` + `FetchDescriptor` | 大量データを事前に絞る |
| メモリレベル | カテゴリ・金額範囲・キーワード・支払方法 | Swift の `Array.filter` | #Predicate の制約を回避 |

**この方式を採用する理由:**
- `#Predicate` 内ではオプショナルのリレーション越し条件（`transaction.category?.name`）がクラッシュする場合がある
- 期間で絞れば結果は数百件規模になるため、メモリ上のフィルタで十分高速
- Swift デバッガでステップ実行可能でデバッグが容易

### 7.4 `#Predicate` の制約事項

`#Predicate` はクロージャの見た目だが、内部で DB クエリに変換されるため以下の制約がある:

- `if` / `switch` 文は使えない（条件分岐による動的組み立て不可）
- カスタム関数の呼び出し不可
- オプショナルのキーパス越し（`?.`）でクラッシュする場合あり
- `String.contains` は `localizedStandardContains` を使う（大文字小文字・ひらがな/カタカナ同一視）

### 7.5 ソート・ページング

- デフォルトソート: `SortDescriptor(\.date, order: .reverse)`（日付の新しい順）
- ページング: MVP では実装しない（SwiftUI の `List` が内部で遅延ロードするため、数千件規模では問題なし）

---

## 8. SwiftData マイグレーション戦略

### 8.1 スキーマバージョニング

最初から `VersionedSchema` を使い、`SchemaV1` として定義する。

```swift
enum SchemaV1: VersionedSchema {
    static var versionIdentifier = Schema.Version(1, 0, 0)
    static var models: [any PersistentModel.Type] {
        [Transaction.self, Category.self, PaymentMethod.self,
         Subscription.self, VariableFixedCost.self,
         IgnoredVariableFixedCost.self]
    }
}
```

将来 V2 を追加するときは `SchemaV2` を定義し、`SchemaMigrationPlan` に乗せる。

### 8.2 軽量マイグレーションの条件

以下の変更は SwiftData が自動対応する（コード変更のみで OK）:

- オプショナルフィールドの追加
- デフォルト値付き非オプショナルフィールドの追加
- フィールドの削除
- `@Attribute(originalName:)` によるリネーム

### 8.3 将来の破壊的変更を避けるための設計ルール

- 新規フィールドは原則**オプショナル**にする（または明確なデフォルト値付き）
- Enum の rawValue は **`String`** で統一（Int の連番より値の追加に強い）
- リレーションの方向は**最初に決めきる**（後からの変更は破壊的）
- プロパティ名のリネームには **`@Attribute(originalName:)`** を必ず付ける

### 8.4 CloudKit 同期時の追加制約

- すべての非リレーションフィールドはオプショナルまたはデフォルト値付き
- `@Attribute(.unique)` は使用不可（CloudKit がユニーク制約をサポートしない）
- リレーションはオプショナル必須
- `.cascade` 削除ルールは使用不可

### 8.5 マイグレーション失敗時の挙動

MVP では `ModelContainer` の初期化失敗時は**クラッシュさせる**。マイグレーション失敗はバグとして扱い、リリース前に検出する。プロダクション後はエラー画面 + 再インストール案内 UI を用意する。

### 8.6 プリセットデータ投入

**投入タイミング:** アプリ起動時（`scenePhase = .active`）

**手順:**
1. CloudKit 同期を考慮し、起動後 **3秒程度待機**
2. 各プリセットの**名前 + kind による存在チェック**を実行（`#Predicate` で検索）
3. 存在しないプリセットのみ投入

```
FOR EACH preset IN presetCategories:
  existing = Category.fetch(
    where: name == preset.name AND kind == preset.kind
  )
  IF existing.isEmpty:
    modelContext.insert(preset)
```

**UserDefaults の初回起動フラグは使わない。** CloudKit 同期で端末間の状態がズレるため。名前による存在チェックのほうが信頼できる。

完全な重複ゼロは CloudKit 環境では保証できないため、万一の重複はカテゴリ管理画面でユーザが手動削除できる逃げ道を残す。

---

## 9. 通知設計

### 9.1 権限取得: 2段階パーミッション方式

iOS の通知権限ダイアログは**1度拒否されると再表示できない**ため、その前にアプリ内の確認を挟む。

**フロー:**
1. トリガー: 毎月費用を初めて登録した時点、または設定画面の通知トグルを ON にした時点
2. アプリ内アラートを表示:「毎月費用の入力時期に通知でお知らせします。通知を有効にしますか?」
3. ユーザが「はい」→ iOS の権限ダイアログ（`requestAuthorization`）を表示
4. ユーザが「いいえ」→ 何もしない（本物のダイアログを温存）

**要求する権限:** `[.alert, .sound, .badge]`

### 9.2 ローカル通知のスケジュール

各 VariableFixedCost に対して、毎月繰り返しの通知を1件スケジュールする。

```swift
let content = UNMutableNotificationContent()
content.title = "毎月費用の入力"
content.body = "\(vfc.name) の入力時期です"
content.sound = .default

var dateComponents = DateComponents()
dateComponents.day = vfc.reminderDay
dateComponents.hour = selectedHour  // デフォルト: 20
dateComponents.minute = 0

let trigger = UNCalendarNotificationTrigger(
    dateMatching: dateComponents, repeats: true
)

let request = UNNotificationRequest(
    identifier: "vfc-\(vfc.id)",
    content: content,
    trigger: trigger
)

UNUserNotificationCenter.current().add(request)
```

### 9.3 再スケジュール方針

個別の差分更新ではなく、**「全削除 → 全再スケジュール」方式**を採用する（毎月費用は数件規模なので性能問題にならない）。

共通関数 `rescheduleAllNotifications()` を作り、以下のすべてのトリガーで呼ぶ:

| トリガー |
|---------|
| 通知設定画面で時刻を変更 |
| 毎月費用を新規追加 |
| 毎月費用の促し日を変更 |
| 毎月費用を削除 |
| 通知設定画面でトグルを ON にした |

### 9.4 制約と割り切り

| 制約 | 対応 |
|------|------|
| `reminderDay = 31` で31日のない月には通知が発火しない | 仕様として受け入れ（バナーが補完する） |
| 入力済みでもプッシュ通知が来てしまう | 通知文言で割り切り（「入力時期です」等） |
| iOS のローカル通知上限 64件/アプリ | 十分（64種類の毎月費用を登録するユーザはいない） |
| 拒否後の再誘導 | 設定アプリへの導線（`UIApplication.openSettingsURLString`）を用意 |

### 9.5 通知設定画面での権限状態同期

画面表示のたびに `UNUserNotificationCenter.current().getNotificationSettings(...)` で OS の権限状態を確認し、トグル表示と実態がズレないようにする。

---

## 10. v1.1 以降の拡張予定

以下は MVP では実装せず、将来バージョンで対応する。データモデルにはフィールドや構造の拡張ポイントを残してある。

| 機能 | 備考 |
|------|------|
| サブスク休会 | `Subscription.pausedUntil: Date?` フィールドのみ用意済み。UI とロジックは v1.1 |
| カテゴリアイコンの色選択 | `Category` に `iconColor` フィールドを追加予定 |
| カテゴリ/支払い方法の並べ替え UI | `displayOrder` フィールドは初期投入時に設定済み |
| 月間グラフの期間切り替え | Picker で 3/6/12 ヶ月切り替え |
| 未入力追跡期間のユーザ設定 | 設定画面に追跡期間の選択肢を追加 |
| 毎月費用「無視」の取り消し UI | 設定画面の毎月費用管理から無視履歴を参照・取り消し |
| 月間グラフのタップドリルダウン | バーをタップでその月の日別タブに遷移 |
| AI 予算アドバイス機能 | LLM 連携。基本設計書 8.1 参照 |
| グループ共有機能（旅行モード） | `Transaction.groupId` フィールド用意済み。基本設計書 8.2 参照 |
| iPad / Mac 対応 | `NavigationSplitView` への切り替え |

---

## 付録A: プロパティ英語名対応表

| 日本語 | 英語名 | 所属エンティティ |
|--------|--------|----------------|
| 種別（支出/収入） | kind | Transaction, Category |
| 日付 | date | Transaction |
| 内容 | title | Transaction |
| 金額 | amount | Transaction, Subscription |
| カテゴリ | category | Transaction, Subscription, VariableFixedCost |
| 支払方法 | paymentMethod | Transaction, Subscription, VariableFixedCost |
| メモ | memo | Transaction |
| サブスク由来フラグ | isFromSubscription | Transaction |
| 由来キー | sourceKey | Transaction |
| グループ ID | groupId | Transaction |
| カテゴリ名 | name | Category |
| アイコン名 | iconName | Category |
| プリセットフラグ | isPreset | Category, PaymentMethod |
| 表示順 | displayOrder | Category, PaymentMethod |
| 「その他」フラグ | isOther | Category |
| サブスク名 | name | Subscription |
| 計上日 | chargeDay | Subscription |
| 最終計上年月 | lastPostedMonth | Subscription |
| 休会期限 | pausedUntil | Subscription |
| 毎月費用名 | name | VariableFixedCost |
| 入力促し日 | reminderDay | VariableFixedCost |
| 毎月費用 ID | variableFixedCostId | IgnoredVariableFixedCost |
| 対象月 | yearMonth | IgnoredVariableFixedCost |
| 無視操作日時 | ignoredAt | IgnoredVariableFixedCost |

---

## 付録B: 基本設計からの変更・具体化事項

詳細設計の過程で基本設計書の内容を以下のように変更・具体化した。

| 項目 | 基本設計の記述 | 詳細設計での決定 | 変更理由 |
|------|-------------|----------------|---------|
| 変動固定費バナーの表示期間 | 入力されるまで無制限に表示 | 直近3ヶ月（当月+過去2ヶ月）に変更 | UX 改善: 古い未入力が無限に積み上がるのを防ぐ。過去月は専用アラート画面で対応 |
| 支出レコードの由来管理 | `isFromSubscription` フラグのみ | `sourceKey: String?` に統合 | サブスク・毎月費用共通の由来キーとして使い、重複検知にも活用 |
| カテゴリアイコン | 記載なし | SF Symbols で実装 | 円グラフやリストでの視認性向上 |
| 支払い方法の削除時挙動 | 未決事項 | `nil`（未設定）に変更 | 「現金」や「その他」への付け替えは実態と異なるため |
| 毎月費用の入力済み判定 | 記載なし | Transaction の `sourceKey` 検索で判定 | VariableFixedCost に状態を持たせると、Transaction 削除時に不整合が起きるため |
| 毎月費用の「無視」機能 | 記載なし | 新エンティティ `IgnoredVariableFixedCost` で管理 | 過去月アラートの UX 改善のため追加 |
| 過去月未入力のアラート UI | 記載なし | ホーム上部の帯 + 専用画面 | バナーとアラートの役割を分離 |
