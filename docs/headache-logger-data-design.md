# Headache Logger Data Design（頭痛記録アプリ データ設計）

## 1. Design Policy（設計方針）

- Daily Logを日付ごとの記録の起点とする
- Daily Logは日付ごとに1件のみ作成する
- 服薬設定と日ごとの服薬記録を分離する
- Daily Log作成時には、その日時点で有効な服薬設定をPreventive Medication Recordとしてコピーして保持する
- 後から服薬設定を変更しても、過去の服薬記録は変更しない
- 1つのDaily Logは複数のPain Logを持てる
- 1つのPain Logは複数のRescue Medicationを持てる
- 予防薬は服用状況のみを記録し、実際の服用時刻は記録しない
- 頓用薬は頭痛との時間関係を振り返るため、服用時刻を記録する

## 2. Terminology（用語対応表）

| 英語名 | 日本語名 | データとしての役割 |
|---|---|---|
| Daily Log | 日別記録 | 1日分の記録をまとめるデータ。予防薬服薬記録と頭痛記録を保持する |
| Medication Schedule | 服薬設定 | 予防薬の薬名、予定時刻、設定の適用期間を管理する |
| Preventive Medication Record | 予防薬服薬記録 | Daily Log作成時点の服薬設定をもとに作成し、その日の服用状況を保持する |
| Pain Log | 頭痛記録 | Daily Logに属し、発生した頭痛の開始・終了日時、強度、メモを保持する |
| Rescue Medication | 頓用薬服薬記録 | Pain Logに属し、頭痛に対して服用した頓用薬の情報を保持する |

## 3. Data Relationship（データの関係）

### データの関係

Medication Schedule（服薬設定）

↓

Daily Logを作成する際、その日時点で有効な服薬設定を読み込む。

↓

Daily Log

├─ Preventive Medication Record（予防薬の服薬記録）
│   ※ 服薬設定をコピーした記録を保持する
│
└─ Pain Log（頭痛記録）
    └─ Rescue Medication（頓用薬）

※ Preventive Medication Recordは、Medication Scheduleの内容をDaily Log作成時にコピーした記録として扱う。

### 関係

- Daily Logは日付ごとに1件作成する
- 1つのDaily Logには複数の予防薬の服薬記録を保持できる
- 1つのDaily Logには複数のPain Logを追加できる
- 1つのPain Logには複数の頓用薬を追加できる
- Daily Logには、その日時点の服薬設定をコピーした服薬記録を保持する
- 服薬設定を変更しても、過去のDaily Logに保存された服薬記録は変更しない

## 4. Data Models（データモデル）

各データモデルが保持する情報と役割を定義する。

### 4.1 Medication Schedule（服薬設定）

予防薬の服薬予定を管理する設定データ。

ユーザーが設定した薬名、予定時刻、適用期間を保持する。
Daily Log作成時には、その日時点で有効なMedication Scheduleを読み込み、
Preventive Medication RecordとしてDaily Logにコピーする。

| 項目名 | 日本語名 | 内容 | 必須 |
|---|---|---|---|
| id | ID | 服薬設定を識別する一意のID | 必須 |
| medicationName | 薬名 | 予防薬の名称 | 必須 |
| scheduledTime | 予定時刻 | 服用予定時刻 | 必須 |
| startDate | 適用開始日 | この設定を適用し始める日付 | 必須 |
| endDate | 適用終了日 | この設定の適用を終了する日付 | 任意 |

#### Rules（ルール）

- 服薬設定は最大3件まで登録できる
- `startDate`以降の日付に設定を適用する
- `endDate`が未設定の場合は、有効な設定として扱う
- `endDate`が設定されている場合は、その日まで設定を有効とする
- 過去のMedication Scheduleを直接変更せず、設定変更時は適用期間を分けて管理する
- Medication Scheduleを変更しても、すでに作成済みのDaily Logには反映しない

### 4.2 Daily Log（日別記録）

1日分の記録をまとめるデータ。

日付、予防薬の服薬記録、メモ、頭痛記録を保持する。
予防薬服薬記録と頭痛記録は複数件保持でき、記録がない場合は空の配列として保持する。
メモは入力されていない場合がある。

| 項目名 | 日本語名 | 内容 | 必須 |
|---|---|---|---|
| id | ID | Daily Logを識別する一意のID | 必須 |
| date | 日付 | 記録対象の日付 | 必須 |
| preventiveMedicationRecords | 予防薬服薬記録 | その日の予防薬の服用状況。記録がない場合は空配列 | 必須 |
| note | メモ | その日全体について記録するメモ | 任意 |
| painLogs | 頭痛記録 | その日に発生したPain Logの一覧。記録がない場合は空配列 | 必須 |

#### Rules（ルール）

- Daily Logは日付ごとに1件のみ作成する
- 同じ日付のDaily Logを複数作成しない
- `preventiveMedicationRecords`は常に配列として保持し、記録がない場合は空配列とする
- `painLogs`は常に配列として保持し、記録がない場合は空配列とする
- `note`は入力されていない状態を許容する
- 予防薬服薬記録、メモ、Pain Logのいずれかが初めて登録された時点でDaily Logを作成する

### 4.3 Preventive Medication Record（予防薬服薬記録）

その日の予防薬の服用状況を記録するデータ。

Daily Log作成時点で有効なMedication Scheduleの内容をコピーして作成する。
元となったMedication Scheduleを識別するIDを保持するが、過去の表示にはコピーしたスナップショットデータを使用する。

| 項目名 | 日本語名 | 内容 | 必須 |
|---|---|---|---|
| id | ID | 予防薬服薬記録を識別する一意のID | 必須 |
| medicationScheduleId | 服薬設定ID | 元となったMedication ScheduleのID | 必須 |
| medicationName | 薬名 | Daily Log作成時点の薬名をコピーした値 | 必須 |
| scheduledTime | 予定時刻 | Daily Log作成時点の服用予定時刻をコピーした値 | 必須 |
| isTaken | 服用状況 | 服用済みの場合はtrue、未服用の場合はfalse | 必須 |

#### Rules（ルール）

- 1件のPreventive Medication Recordは、1回分の服薬予定を表す
- Daily Log作成時点で有効なMedication Scheduleごとに1件作成する
- `medicationScheduleId`は元の服薬設定を追跡するために使用する
- 薬名と予定時刻はMedication Scheduleからコピーし、スナップショットとして保持する
- 過去の表示ではMedication Scheduleを参照せず、保存済みの`medicationName`と`scheduledTime`を使用する
- Medication Scheduleを変更または終了しても、作成済みのPreventive Medication Recordは変更しない
- 作成時の`isTaken`は`false`とする
- 実際の服用時刻は記録しない

### 4.4 Pain Log（頭痛記録）

発生した頭痛の状態と経過を記録するデータ。

1件のPain Logは1回の頭痛を表し、開始日時、終了日時、痛みの強度、メモを保持する。
頭痛に対して頓用薬を服用した場合は、Rescue Medicationを複数件保持できる。

| 項目名 | 日本語名 | 内容 | 必須 |
|---|---|---|---|
| id | ID | Pain Logを識別する一意のID | 必須 |
| startedAt | 開始日時 | 頭痛が始まった日付と時刻 | 必須 |
| endedAt | 終了日時 | 頭痛が終了した日付と時刻。継続中の場合は未設定 | 任意 |
| intensity | 痛みの強度 | 痛みを1から5までの5段階で記録する | 必須 |
| note | メモ | 頭痛の状態や状況について記録するメモ | 任意 |
| rescueMedications | 頓用薬服薬記録 | この頭痛に対して服用したRescue Medicationの一覧。記録がない場合は空配列 | 必須 |

#### Rules（ルール）

- 1件のPain Logは1回の頭痛を表す
- `startedAt`の初期値にはPain Log作成時点の現在日時を設定する
- ユーザーは`startedAt`を後から編集できる
- `endedAt`が未設定の場合は、頭痛が継続中であると扱う
- `endedAt`には`startedAt`より前の日時を設定できない
- ユーザーは`endedAt`を後から追加・編集できる
- `intensity`は1から5までの整数とする
- ユーザーは`intensity`と`note`を後から編集できる
- `rescueMedications`は常に配列として保持し、記録がない場合は空配列とする
- 1つのPain Logには複数のRescue Medicationを追加できる
- Pain Logを削除する場合は、保持しているRescue Medicationも同時に削除する
- Pain Logは`startedAt`の日付に対応するDaily Logに属する
- Pain Logが複数日にまたがる場合も、開始日のDaily Logとの所属関係は変更しない
- 終了日のDaily LogにはPain Logを追加しない
- 継続中のPain Logを表示する場合は、当日のDaily Logだけではなく、保存済みの全Pain Logから`endedAt`が未設定の記録を検索する
- 終了操作では、開始日のDaily Logに保存されているPain Logの`endedAt`を更新する

### 4.5 Rescue Medication（頓用薬服薬記録）

頭痛に対して服用した頓用薬を記録するデータ。

Rescue MedicationはPain Logに属し、薬名、服用日時、薬の種別を保持する。
1つのPain Logには複数のRescue Medicationを追加できる。

| 項目名 | 日本語名 | 内容 | 必須 |
|---|---|---|---|
| id | ID | 頓用薬服薬記録を識別する一意のID | 必須 |
| medicationName | 薬名 | 服用した頓用薬の名称 | 必須 |
| takenAt | 服用日時 | 頓用薬を服用した日付と時刻 | 必須 |
| medicationType | 種別 | 処方薬または市販薬 | 必須 |

#### Rules（ルール）

- Rescue Medicationは単独では存在せず、必ず1つのPain Logに属する
- 1件のRescue Medicationは1回分の服薬を表す
- 1つのPain Logには複数のRescue Medicationを保持できる
- `takenAt`の初期値にはRescue Medication作成時点の現在日時を設定する
- ユーザーは`medicationName`、`takenAt`、`medicationType`を後から編集できる
- `medicationType`は処方薬または市販薬のいずれかとする
- Pain Logを削除した場合、そのPain Logに属するRescue Medicationも同時に削除する

## 5. TypeScript Types（型設計）

データモデルをTypeScriptで扱うための型を定義する。

日付・日時・時刻は文字列として保持する。
配列項目は常に配列として保持し、要素がない場合は空配列とする。

```ts
type MedicationType = "prescription" | "overTheCounter";

interface MedicationSchedule {
  id: string;
  medicationName: string;
  scheduledTime: string;
  startDate: string;
  endDate?: string;
}

interface PreventiveMedicationRecord {
  id: string;
  medicationScheduleId: string;
  medicationName: string;
  scheduledTime: string;
  isTaken: boolean;
}

interface RescueMedication {
  id: string;
  medicationName: string;
  takenAt: string;
  medicationType: MedicationType;
}

interface PainLog {
  id: string;
  startedAt: string;
  endedAt?: string;
  intensity: 1 | 2 | 3 | 4 | 5;
  note?: string;
  rescueMedications: RescueMedication[];
}

interface DailyLog {
  id: string;
  date: string;
  preventiveMedicationRecords: PreventiveMedicationRecord[];
  note?: string;
  painLogs: PainLog[];
}
```

### Date and Time Format（日付・時刻の形式）

- `date`、`startDate`、`endDate`は`YYYY-MM-DD`形式の文字列とする
- `scheduledTime`は`HH:mm`形式の文字列とする
- `startedAt`、`endedAt`、`takenAt`はISO 8601形式の日時文字列とする
- 日時は日付をまたぐ記録に対応できる形式で保存する

## 6. Storage Structure（保存構造）

初期実装では、アプリのデータをLocalStorageに保存する。

Medication Schedule（服薬設定）とDaily Log（日別記録）は、それぞれ別の保存キーで管理する。

### Medication Schedules（服薬設定）

Medication Scheduleは配列として保存する。

設定を追加・変更・終了した場合も、過去の設定履歴を保持する。

### Daily Logs（日別記録）

Daily Logは日付をキーとするオブジェクトとして保存する。

日付キーは`YYYY-MM-DD`形式とし、1つの日付につき1件のDaily Logを保持する。

```ts
{
  "2026-08-01": {
    // 2026年8月1日のDaily Log
  },
  "2026-08-02": {
    // 2026年8月2日のDaily Log
  }
}
```

各Daily Logは、その日に対応するPreventive Medication Recordと、その日に開始したPain Logを保持する。

各Pain Logは、その頭痛に対して記録されたRescue Medicationを保持する。

### Storage Keys（保存キー）

| 保存キー | 保存内容 |
|---|---|
| `headacheLogger.medicationSchedules` | Medication Scheduleの一覧 |
| `headacheLogger.dailyLogs` | 日付をキーとしたDaily Logの一覧 |

### Rules（ルール）

- LocalStorageにはJSON形式へ変換して保存する
- Medication ScheduleとDaily Logは別々の保存キーで管理する
- Daily Logの日付キーには`YYYY-MM-DD`形式を使用する
- 1つの日付キーにつき、1件のDaily Logのみ保持する
- Preventive Medication RecordはDaily Log内に保存する
- Pain Logは`startedAt`の開始日に対応するDaily Log内に保存する
- 頭痛が複数日にまたがっても、Pain Logを翌日以降のDaily Logへ複製しない
- Rescue Medicationは所属するPain Log内に保存する
- データが存在しない場合は空のオブジェクトまたは空配列として扱う

## 7. Data Creation and Update Rules（データ作成・更新ルール）

### 7.1 Medication Schedule（服薬設定）

- 新しい服薬タイミングを追加する場合は、新しいMedication Scheduleを作成する
- Medication Scheduleの作成時には、`id`、`medicationName`、`scheduledTime`、`startDate`を設定する
- 継続中の設定では`endDate`を未設定とする
- 服薬設定を終了する場合は、対象のMedication Scheduleに`endDate`を設定する
- 過去の服薬設定を直接上書きせず、適用期間を分けて履歴を保持する
- Medication Scheduleを変更または終了しても、作成済みのPreventive Medication Recordは変更しない

### 7.2 Daily Log（日別記録）

- Daily Logは、予防薬服薬記録、メモ、またはPain Logが初めて登録された時点で作成する
- Daily Logは日付ごとに1件のみ作成する
- Daily Logの保存先には、対象日を`YYYY-MM-DD`形式の日付キーとして使用する
- 同じ日付キーのDaily Logがすでに存在する場合は、新規作成せず既存のDaily Logを更新する
- Daily Log作成時には、その日時点で有効なMedication Scheduleを読み込む
- 有効なMedication ScheduleごとにPreventive Medication Recordを1件作成する
- `preventiveMedicationRecords`と`painLogs`は、記録がない場合も空配列として保持する
- `note`は未入力を許容する
- アプリまたはToday画面を表示しただけではDaily Logを作成しない
- 対象日のDaily Logが存在しない場合は、有効なMedication Scheduleをもとに服薬候補を画面上へ一時的に表示する
- 画面表示のために生成した一時的なデータは、ユーザーが記録操作を行うまで保存しない
- 予防薬の服用チェック、メモの保存、Pain Logの追加のいずれかが行われた時点でDaily Logを作成する

### 7.3 Preventive Medication Record（予防薬服薬記録）

- Preventive Medication Recordは、Daily Log作成時点で有効なMedication Scheduleをもとに作成する
- 1件のPreventive Medication Recordは、1回分の服薬予定を表す
- `medicationScheduleId`には、元となったMedication ScheduleのIDを設定する
- `medicationName`と`scheduledTime`には、作成時点のMedication Scheduleの値をコピーする
- 作成時の`isTaken`は`false`とする
- ユーザーが服薬チェックを行った場合は、対象の`isTaken`を`true`に更新する
- ユーザーが服薬記録を解除した場合は、対象の`isTaken`を`false`に戻す
- Medication Scheduleの内容が後から変更されても、作成済みのスナップショットは更新しない

### 7.4 Pain Log（頭痛記録）

- Pain Logは、頭痛記録を開始した時点で作成する
- `startedAt`の初期値には、作成時点の現在日時を設定する
- Pain Logは、`startedAt`の日付に対応するDaily Logの`painLogs`に追加する
- 開始日のDaily Logが存在しない場合は、先にDaily Logを作成する
- 頭痛が複数日にまたがっても、新しいPain Logを作成しない
- 終了操作では、開始日のDaily Logに保存されているPain Logの`endedAt`を更新する
- `endedAt`が未設定の場合は継続中として扱う
- `endedAt`には`startedAt`より前の日時を設定できない
- `startedAt`を別の日付へ変更した場合は、Pain Logを変更後の開始日に対応するDaily Logへ移動する
- `intensity`、`note`、`startedAt`、`endedAt`は後から編集できる
- `rescueMedications`は、記録がない場合も空配列として保持する

### 7.5 Rescue Medication（頓用薬服薬記録）

- Rescue Medicationは、必ず1つのPain Logに追加する
- 1件のRescue Medicationは1回分の服薬を表す
- `takenAt`の初期値には、作成時点の現在日時を設定する
- `medicationName`、`takenAt`、`medicationType`は後から編集できる
- 頭痛が日付をまたいでいる場合も、Rescue Medicationは同じPain Log内に保持する
- 服用日時が開始日の翌日以降であっても、別のDaily Logへ移動しない

## 8. Deletion Rules（削除ルール）

### 8.1 Medication Schedule（服薬設定）

- 処方終了や服用中止の場合はMedication Scheduleを削除せず、`endDate`を設定して終了扱いとする
- `endDate`は、その服薬設定を最後に適用する日付とする
- `endDate`より後の日付でDaily Logを作成する場合、終了済みのMedication ScheduleからPreventive Medication Recordを作成しない
- Medication Scheduleを終了しても、過去に作成済みのPreventive Medication Recordは削除・変更しない
- 誤って作成し、一度も服薬記録の元として使用されていないMedication Scheduleのみ、物理削除を許可する

### 8.2 Daily Log（日別記録）

- Daily Logを削除する前に確認を表示する
- Daily Logを削除した場合、そのDaily Log内のPreventive Medication Record、Pain Log、Rescue Medicationもすべて同時に削除する
- Daily Logを削除しても、Medication Scheduleは削除しない
- 削除後は、対象日の日付キーを`dailyLogs`から削除する
- Daily Log内の記録を個別に削除・解除した結果、以下のすべてを満たす場合は、そのDaily Logも削除する
  - `isTaken`が`true`のPreventive Medication Recordが存在しない
  - メモが入力されていない
  - Pain Logが存在しない
- アプリまたはToday画面を表示しただけの未保存状態はDaily Logとして扱わないため、削除対象にはならない

### 8.3 Preventive Medication Record（予防薬服薬記録）

- Preventive Medication Recordは原則として個別削除しない
- 誤って服用済みにした場合は削除せず、`isTaken`を`false`に戻す
- 元となったMedication Scheduleを終了または削除しても、作成済みのPreventive Medication Recordは削除・変更しない
- Daily Logを削除した場合は、そのDaily Logに属するPreventive Medication Recordも同時に削除する

### 8.4 Pain Log（頭痛記録）

- Pain Logを削除する前に確認を表示する
- Pain Logを削除した場合、そのPain Logに属するRescue Medicationもすべて同時に削除する
- Pain Logを削除しても、所属するDaily Log内のPreventive Medication Recordやメモは削除しない
- Pain Log削除後、Daily Logに記録済みの服薬、メモ、他のPain Logが存在しない場合は、Daily Logも削除する

### 8.5 Rescue Medication（頓用薬服薬記録）

- Rescue Medicationを削除する前に確認を表示する
- Rescue Medicationを削除しても、所属するPain Logは削除しない
- Pain Logを削除した場合は、そのPain Logに属するRescue Medicationも同時に削除する