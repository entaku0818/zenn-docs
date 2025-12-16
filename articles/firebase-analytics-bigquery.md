---
title: "【Firebase】Analytics + BigQuery連携でユーザー行動を分析する"
emoji: "📊"
type: "tech"
topics: ["Firebase", "BigQuery", "Analytics", "GCP", "データ分析"]
published: false
---

# この記事は？

Firebase AnalyticsのデータをBigQueryにエクスポートし、SQLで詳細な分析を行う方法を解説します。標準のFirebaseコンソールでは見られない、カスタム分析が可能になります。

## 前提条件

- Firebaseプロジェクト（Blazeプラン）
- BigQueryが有効化されたGCPプロジェクト

## Firebase Analytics → BigQueryの連携設定

### 1. Firebaseコンソールでの設定

1. Firebaseコンソール → プロジェクト設定 → 統合
2. BigQueryを選択
3. 「リンク」をクリック
4. エクスポートするデータを選択（推奨: すべて）

### 2. エクスポートされるデータ

BigQueryには以下のテーブルが作成されます：

- `events_YYYYMMDD`: 日次のイベントデータ
- `events_intraday_YYYYMMDD`: 当日のリアルタイムデータ

## React Nativeでのイベント送信

```typescript
// services/analytics.ts
import analytics from '@react-native-firebase/analytics'

// カスタムイベントの送信
export const logTalkRoomJoin = async (params: {
  talkRoomId: string
  talkRoomTitle: string
  memberCount: number
}) => {
  await analytics().logEvent('talk_room_join', {
    talk_room_id: params.talkRoomId,
    talk_room_title: params.talkRoomTitle,
    member_count: params.memberCount,
  })
}

// ユーザープロパティの設定
export const setUserProperties = async (userId: string) => {
  await analytics().setUserId(userId)
  await analytics().setUserProperty('user_type', 'premium')
}

// 画面表示のトラッキング
export const logScreenView = async (screenName: string) => {
  await analytics().logScreenView({
    screen_name: screenName,
    screen_class: screenName,
  })
}
```

## BigQueryでの分析クエリ

### 日次アクティブユーザー（DAU）

```sql
SELECT
  event_date,
  COUNT(DISTINCT user_pseudo_id) AS dau
FROM
  `project_id.analytics_XXX.events_*`
WHERE
  _TABLE_SUFFIX BETWEEN '20250101' AND '20250131'
GROUP BY
  event_date
ORDER BY
  event_date
```

### イベント別の発生回数

```sql
SELECT
  event_name,
  COUNT(*) AS event_count,
  COUNT(DISTINCT user_pseudo_id) AS unique_users
FROM
  `project_id.analytics_XXX.events_*`
WHERE
  _TABLE_SUFFIX = FORMAT_DATE('%Y%m%d', CURRENT_DATE())
GROUP BY
  event_name
ORDER BY
  event_count DESC
```

### トークルーム参加の分析

```sql
SELECT
  event_date,
  (SELECT value.string_value FROM UNNEST(event_params) WHERE key = 'talk_room_id') AS talk_room_id,
  (SELECT value.string_value FROM UNNEST(event_params) WHERE key = 'talk_room_title') AS talk_room_title,
  COUNT(*) AS join_count
FROM
  `project_id.analytics_XXX.events_*`
WHERE
  event_name = 'talk_room_join'
  AND _TABLE_SUFFIX BETWEEN '20250101' AND '20250131'
GROUP BY
  event_date, talk_room_id, talk_room_title
ORDER BY
  join_count DESC
LIMIT 100
```

### ファネル分析

```sql
WITH funnel AS (
  SELECT
    user_pseudo_id,
    MAX(CASE WHEN event_name = 'app_open' THEN 1 ELSE 0 END) AS step1_app_open,
    MAX(CASE WHEN event_name = 'view_home' THEN 1 ELSE 0 END) AS step2_view_home,
    MAX(CASE WHEN event_name = 'talk_room_join' THEN 1 ELSE 0 END) AS step3_join_room,
    MAX(CASE WHEN event_name = 'send_message' THEN 1 ELSE 0 END) AS step4_send_message
  FROM
    `project_id.analytics_XXX.events_*`
  WHERE
    _TABLE_SUFFIX = FORMAT_DATE('%Y%m%d', CURRENT_DATE())
  GROUP BY
    user_pseudo_id
)
SELECT
  COUNT(*) AS total_users,
  SUM(step1_app_open) AS app_open,
  SUM(step2_view_home) AS view_home,
  SUM(step3_join_room) AS join_room,
  SUM(step4_send_message) AS send_message
FROM
  funnel
```

### リテンション分析（日次）

```sql
WITH first_open AS (
  SELECT
    user_pseudo_id,
    MIN(event_date) AS first_date
  FROM
    `project_id.analytics_XXX.events_*`
  WHERE
    event_name = 'first_open'
  GROUP BY
    user_pseudo_id
),
daily_active AS (
  SELECT DISTINCT
    user_pseudo_id,
    event_date
  FROM
    `project_id.analytics_XXX.events_*`
)
SELECT
  DATE_DIFF(PARSE_DATE('%Y%m%d', da.event_date), PARSE_DATE('%Y%m%d', fo.first_date), DAY) AS days_since_install,
  COUNT(DISTINCT da.user_pseudo_id) AS retained_users
FROM
  daily_active da
JOIN
  first_open fo ON da.user_pseudo_id = fo.user_pseudo_id
WHERE
  DATE_DIFF(PARSE_DATE('%Y%m%d', da.event_date), PARSE_DATE('%Y%m%d', fo.first_date), DAY) BETWEEN 0 AND 30
GROUP BY
  days_since_install
ORDER BY
  days_since_install
```

## Scheduled Queryの設定

定期的にレポートを生成する場合：

```sql
-- 日次レポートテーブルに結果を保存
CREATE OR REPLACE TABLE `project_id.reports.daily_summary` AS
SELECT
  event_date,
  COUNT(DISTINCT user_pseudo_id) AS dau,
  COUNT(*) AS total_events
FROM
  `project_id.analytics_XXX.events_*`
WHERE
  _TABLE_SUFFIX = FORMAT_DATE('%Y%m%d', DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY))
GROUP BY
  event_date
```

## コスト最適化のTips

### パーティション分割テーブルの使用

```sql
-- 日付でフィルタリングしてスキャン量を削減
WHERE
  _TABLE_SUFFIX BETWEEN '20250101' AND '20250107'
```

### 必要なカラムのみ選択

```sql
-- ❌ 全カラム取得（高コスト）
SELECT * FROM `project_id.analytics_XXX.events_*`

-- ✅ 必要なカラムのみ（低コスト）
SELECT event_name, event_date, user_pseudo_id
FROM `project_id.analytics_XXX.events_*`
```

## まとめ

- FirebaseコンソールからBigQueryリンクを設定
- イベントは`event_params`配列に格納される
- `_TABLE_SUFFIX`で日付フィルタリング
- Scheduled Queryで定期レポート生成

## 参考資料

- [Firebase - BigQuery Export](https://firebase.google.com/docs/projects/bigquery-export)
- [BigQuery - Firebase スキーマ](https://support.google.com/firebase/answer/7029846)
