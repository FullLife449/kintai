# マスタ管理タブ 再構成 仕様書（確定版・Claude Code 実装引き継ぎ用）

対象: `index.html`（単一ファイル構成・GitHub Pages `fulllife449/kintai` 本番運用中）
現状: スタッフ追加/編集モーダル `#stm` 内のマスタ管理は **7タブ**。これを下記 **8タブ** へ再構成する。
作業ブランチ: `master-tab-4tabs`（※当初4タブ案だったが、権限分離の要件で8タブに確定。ブランチ名はそのまま使用）

---

## 0. 作業ルール（厳守）

- **単一HTMLファイル構成を維持**（外部JS/CSSに分割しない）
- 既存コードの構造を尊重し、**最小限の変更**で実装する。命名規則は既存（`f-xxx` id、`s.xxx` データキー）に合わせる
- **既存スタッフ情報・打刻データを絶対に削除しない**。既存データキーは温存・リネーム禁止（後方互換）
- データを変更する関数には **必ず Firebase 同期**を含める: `saveSt()` → `fbSaveSettings()` の呼び忘れ厳禁
- 会社別フィルタは **`CO`（現在選択中の会社）で統一**
- 実装後は **`node -e` で構文チェック**（`new Function(scriptText)`）を必ず行う
- 完了後、変更内容を箇条書きで要約

---

## 1. 設計の核心（最重要）

> **一般管理者には「個別性のある契約情報（賃金・雇用契約・退職条件・契約書）」を一切見せない。**
> 上級管理者（`senior`）・総合上級管理者（`super`）のみが閲覧できる。

この権限境界が**タブの並び順と一致**するように再編する。前半4タブ＝全管理者OK、後半4タブ＝上級以上のみ（🔒）。
一般管理者には🔒タブの**ボタン自体を非表示**にする（フィールド単位ロックではなく、タブ単位ロック）。

権限判定は既存 `canViewSalary()` をそのまま流用する（`super` または `senior` で `true`）。変更不要。

```js
// 既存（変更しない）
function canViewSalary(){
  if(!currentAdmin)return false;
  return currentAdmin.role==='super'||currentAdmin.role==='senior';
}
```

---

## 2. 確定タブ構成（8タブ）

全タブ共通: モーダル上部に **フリガナ・職員番号** を常時表示。

| # | data-tab | 表示名 | 閲覧権限 | 主な内容 |
|---|---|---|---|---|
| 1 | `basic` | 基本情報 | 全管理者 | 氏名・フリガナ・社員番号・**郵便番号(新)**・住所・電話・メール・生年月日・性別・保証人（緊急連絡先：氏名/フリガナ/続柄/電話） |
| 2 | `work` | 勤務（運用） | 全管理者 | 職種・資格〔**各資格に取得日/期限/免許番号(新)**〕・勤務情報(勤務タイプ+曜日別シフト+休憩+終了時刻フリー+フリー①)・所定休日・時間外・変形労働・通勤情報・有給設定・締切日・支払日・残業代計算・昇給・**所属部署(新)**・**役職(新)**・**扶養家族(新)** |
| 3 | `insurance` | 保険 | 全管理者 | 健康保険・厚生年金・雇用保険・労災保険・**基礎年金番号(新)**・**雇用保険被保険者番号(新)**・**社会保険 資格取得日(新)**・**社会保険 資格喪失日(新)** |
| 4 | `other` | その他 | 全管理者 | **健康診断 実施日(新)**（＋将来の予備項目枠） |
| 5 | `contract` | 雇用契約 🔒 | 上級以上のみ | 雇用形態・雇用期間・期間開始/終了・試用期間/月数・**契約更新条件**・就業場所・業務内容（※勤務タブから分離した個別契約条件） |
| 6 | `wage` | 賃金 🔒 | 上級以上のみ | **年俸額(新)**・**月給(新)**・**時給パターン(新)**・**手当5項目(新)**・通常賞与・臨時賞与・**退職金(金額＋規定)**・**銀行口座**・**副業・兼業**・**特約事項** |
| 7 | `retire` | 退職 🔒 | 上級以上のみ | **退職事由(新)**・解雇事由・**退職時手続きフロー(新)**・定年(**デフォルト60歳**)・退職日 |
| 8 | `document` | 契約書記載事項 🔒 | 上級以上のみ | 秘密保持・競業避止(内容＋期間)・**契約書出力ボタン(新・Q4必須)** |

🔒境界は **タブ4と5の間**。`canViewSalary()===false` のとき、タブ5〜8のボタンとセクションを非表示にする。

---

## 3. データキー設計

### 3-1. 温存（既存・リネーム禁止・表示位置のみ移動）
`furigana` `gender` `birthDate` `address` `phone` `email` `retireDate` `empType` `empPeriod` `empStart` `empEnd` `probation` `probationMonths` `renewal` `workplace` `duty` `emergName` `emergFurigana` `emergRel` `emergPhone` `overtime` `irregular` `wageType` `baseWage` `cutoffDay` `payDay` `overtimeCalc` `raise` `bonusRegular` `bonusTiming` `bonusCalc` `bonusExtra` `bonusExtraCond` `retireAge` `severance` `severanceRule` `dismissal` `bank` `bankBranch` `bankType` `bankNo` `bankKana` `insHealth` `insPension` `insEmp` `insWork` `nonCompete` `nonCompetePeriod` `confidentiality` `sidejob` `specialNote` `commuteFrom` `commuteVia` `commuteTo`
／既存個別: `fn` `feid` `f-furigana-basic`・職種資格chips・有給(`f-hire`/`f-pmode`/`f-wdays`)・勤務タイプ(`ft`)・shift系(`fbk`/`fft`/`ff1min`)・権限(`adminRole`/`adminPerm`)

### 3-2. 新規追加（`collectMasterFields()` / `loadMasterFields()` 両方に追加）

| タブ | キー | 型/内容 |
|---|---|---|
| basic | `postalCode` | 郵便番号（文字列） |
| work | `department` | 所属部署 |
| work | `position` | 役職 |
| work | `dependents` | 扶養家族リスト（可変長配列）`[{name, relation, birthDate}, ...]` |
| work | `licDetails` | 資格詳細。資格IDをキーに `{ [licId]: {acquiredDate, expiry, licenseNo} }`（職種資格chips拡張） |
| wage | `annualWage` | 年俸額 |
| wage | `monthlyWage` | 月給 |
| wage | `hourlyPattern` | 時給パターン |
| wage | `allowance1`〜`allowance5` | 手当 金額 |
| wage | `allowance1Period`〜`allowance5Period` | `"year"` / `"month"` |
| wage | `allowance1Name`〜`allowance5Name` | 手当名称（自由入力・プルダウン蓄積式） |
| insurance | `pensionNo` | 基礎年金番号 |
| insurance | `empInsNo` | 雇用保険被保険者番号 |
| insurance | `insAcqDate` | 社会保険 資格取得日 |
| insurance | `insLossDate` | 社会保険 資格喪失日 |
| retire | `retireReason` | 退職事由 |
| retire | `retireFlow` | 退職時手続きフロー |
| other | `checkupDate` | 健康診断 実施日 |

### 3-3. 手当名称のプルダウン（コンボボックス式）
- 名称は **固定リストを初期投入しない**。`<input list="...">` ＋ `<datalist>` で実装。
- datalist の候補は **過去に入力された手当名称の集合**から動的生成（全スタッフの `allowanceNName` を走査、または settings に `allowanceNames` マスタ配列を持たせて保存時に追記）。
- ユーザーが新しい名称を入力すると次回から候補に出る（蓄積式）。

### 3-4. 賃金の取り込み（Q3-a）
- 入力UIは新項目（年俸/月給/時給パターン＋手当）を主とする。
- 保存時、既存 `wageType`/`baseWage` へも整合する値を**内部同期**して後方互換を維持（既存の出勤・給与計算ロジックは旧キー参照のまま触らない）。

---

## 4. デフォルト値・自動文言

- 定年 `retireAge`: **60**（プレースホルダ/初期値）
- 通常賞与 `bonusRegular`: **「12月から5月、6月から11月の成績に応じて年2回支払う」**（初期値/プレースホルダ）
- 通勤先 `commuteTo`: 既存どおり「香里園」初期値
- **契約書出力時**（タブ8）: 本文に **「65歳まで継続雇用、70歳まで努力義務」** を自動追記する

---

## 5. 実装手順（推奨）

1. HTML: `.mtb` ボタンを8つ（basic / work / insurance / other / contract / wage / retire / document）に再構成
2. HTML: モーダル上部に「フリガナ・職員番号」常時表示エリアを追加
3. HTML: 各 `.mtab-section[data-section]` を新タブ構成に合わせて配置し直す（既存項目の移動）
   - 勤務→雇用契約への分離（雇用形態/期間/試用/更新条件/就業場所/業務内容を `contract` へ）
   - その他→賃金への移動（銀行口座/副業/特約事項）
   - 退職→賃金への移動（退職金 金額＋規定）
   - 雇用→退職への移動（退職日・更新条件は退職ではなく contract へ／退職日は retire へ）
4. HTML: 新規フィールドのUI追加（郵便番号・所属部署・役職・扶養家族リスト・資格詳細・年俸/月給/時給・手当5項目コンボボックス・保険番号類・退職事由/手続きフロー・健診日・契約書出力ボタン）
5. JS: `setMTab()` を8タブ用に再実装。`canViewSalary()===false` のとき contract/wage/retire/document のボタン＋セクションを非表示
6. JS: `loadMasterFields()` / `collectMasterFields()` に §3-2 の新規キーを追加（配列系 dependents / licDetails のシリアライズ含む）
7. JS: 契約書出力関数を新規作成（タブ8のボタン → 各データを差し込み、65歳/70歳文言を自動追記）
8. JS: 定年60・賞与既定文言を初期値/プレースホルダに反映
9. `saveSt()` が `fbSaveSettings()` を呼ぶことを確認
10. `node -e` 構文チェック → `git commit` & `push` → GitHub Pages 自動デプロイ確認

---

## 6. テスト観点

- 既存スタッフを開いて全項目が正しく読み込まれる（後方互換・旧キー温存）
- 新規スタッフ追加→保存→再読込で新規キー（配列含む）が保持される
- **一般管理者でログイン → 🔒タブ（雇用契約/賃金/退職/契約書記載事項）のボタンが表示されない**
- 上級・総合上級でログイン → 8タブすべて表示される
- 手当名称が過去入力からプルダウン候補に出る
- 契約書出力に賃金等が反映され、「65歳継続雇用・70歳努力義務」が自動追記される
- Firebase に反映される（`fbSaveSettings` 発火）
- `CO`（会社フィルタ）で表示が崩れない

---

## 7. 確定した決定事項ログ

- Q1 勤務タイプ等の巨大ブロック → **勤務(運用)タブ**へ移動
- Q2 賃金ロック範囲 → 上級＋総合上級（`canViewSalary()` 現状維持）
- Q3-a 給与形態/基本給 → 新Ver（年俸/月給/時給）に取り込み、旧キーへ内部同期
- Q3-b 手当名称 → プルダウン＋新規入力（固定名称は初期投入しない）
- Q4 契約書出力 → **必須**（タブ8に出力ボタン）
- マイナンバー → **扱わない**（除外）
- 扶養家族 → 氏名・続柄・生年月日のリスト（人数可変）
- 退職金 → 退職タブに残さず**賃金タブへ集約**（金額＋規定）
- 勤務の分割 → **採用**（勤務(運用) ＋ 雇用契約）
- タブ数 → **8タブ確定**（健診「その他」は独立維持）
- ロック方式 → **タブごと非表示**
