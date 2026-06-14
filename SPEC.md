# 朗讀評分 — 散文・詩詞朗讀評估

**Forked from**: sen-yue-read-score
**Target**: 輕度學生 (mild SEN) — 散文、詩詞朗讀評分
**Stack**: 單檔 SPA (`index.html`), inline CSS + JS, GitHub Pages static deploy

---

## 核心改動（相對原版）

| 維度 | 原有（詞彙） | 新版（散文・詩詞） |
|---|---|---|
| 學習單位 | 單字/詞彙 | 散文、詩詞篇章 |
| 學生流程 | 詞彙學習 → 即時評估 | 朗讀 → 我的進度 |
| Hub 入口 | 詞彙學習 / 即時評估 / 我的進度 | 朗讀 / 我的進度 |
| 評估模式 | 逐字朗讀 → 即時反饋 | 篇章朗讀 → 整篇評分 |
| 老師詞彙 tab | 詞彙 CRUD + category toggle | 文章 CRUD + type toggle |
| 數據檔案 | `data/words.json` (categories/words) | `data/passages.json` (types/passages) |

---

## 學生端流程

### 登入
- 同原版：emoji 頭像卡 grid，點擊確認卡登入
- 輕度學生：移除所有 PIN / 密碼

### Hub（中心頁）
- **📖 朗讀** — 揀選文章朗讀
- **⭐ 我的進度** — 查看學習記錄

### 朗讀頁（評估）
1. **揀文章**：按 散文/詩詞 分類，大卡展示（標題、作者、難度星級）
2. **預覽文章**：大字顯示全文，可點擊 TTS 示範朗讀
3. **錄音評估**：一鍵錄音（時長按文章長度：散文 30s，詩詞 15s），實時倒數
4. **評分結果**：展示朗讀分數（文字比對）、示範朗讀、重試

### 我的進度
- 最近朗讀記錄（文章、日期、分數）
- 總朗讀次數、總分、平均分

---

## 評估與計分

- **ASR 引擎**：同原版（Web Speech / Whisper 本地 / 自訂比對）
- **計分演算法**：
  - Levenshtein distance（朗讀文字 vs 文章目標文字）
  - 完整度（朗讀了多少內容）
  - 流暢度反饋
- **評級**：金（90+）/ 銀（75+）/ 銅（60+）/ 參與（<60）
- **證書**：同上（金/銀/銅/參與），confetti 動畫

---

## 老師工作台（4 tabs）

| Tab | 內容 |
|---|---|
| 📊 Dashboard | 班級總覽、朗讀次數、分數分佈、難點文章 |
| 👨‍🎓 學生 | 學生 CRUD、匯出/匯入、重設進度 |
| ✏️ 文章 | 文章 CRUD（散文/詩詞 toggle）、匯出/匯入 |
| ⚙️ 系統 | ASR 引擎切換、Whisper cache 管理 |

---

## 數據模型

### passages.json
```json
{
  "types": [
    {
      "id": "prose",
      "label": "散文",
      "description": "抒情散文、短文",
      "enabled": true
    },
    {
      "id": "poetry",
      "label": "詩詞",
      "description": "詩詞朗誦",
      "enabled": true
    }
  ],
  "passages": [
    {
      "id": "p1",
      "type": "prose",
      "title": "春天的校園",
      "author": "選自課本",
      "difficulty": 1,
      "text": "春天的校園，花兒開了，草兒綠了。同學們在操場上跑步、做運動。陽光溫暖，微風輕輕吹過。",
      "audioSeconds": 15
    },
    {
      "id": "p2",
      "type": "poetry",
      "title": "靜夜思",
      "author": "李白",
      "difficulty": 1,
      "text": "床前明月光，疑是地上霜。舉頭望明月，低頭思故鄉。",
      "audioSeconds": 10
    }
  ]
}
```

### 持久化
`localStorage` keys: `yueCategories` → `yueTypes`, `yueStudents`, `yueProgress`, `yueAdminSettings`, `yueAsrEngine`, `yueTeacherPassHash`

### 學生進度結構
```js
{
  [studentId]: {
    records: [{ passageId, passageTitle, score, date, text, feedback }],
    totalAttempts: 0,
    totalScore: 0
  }
}
```

---

## 安全

- 同原版：所有 user content 過 `_e()` HTML escape
- XSS 評分後端：Levenshtein distance + char overlap + first-match bonus

---

## 輕度學生 UX 優化

1. 移除所有 `confirm()` / `alert()` / `prompt()` → in-app modal（從原版待優化清單 #1 提前實現）
2. 更大的按鈕（min-height 70px）、更高對比度
3. 減少步驟：朗讀流程簡化為 揀文章 → 朗讀 → 結果（3 步）
4. 清晰的視覺指引：大字體、emoji icon、顏色編碼（金/銀/銅/綠）

---

## 待優化

1. 🔴 高：Whisper 本地模型需下載，考慮用 `whisper-tiny` 或 `whisper-base` 加快首次速度
2. 🟡 中：朗讀評估的計分演算法可優化（加入停頓檢測、語速分析）
3. 🟢 低：教師密碼設定改為 in-app modal
4. 🟢 低：Touch target 加大至 60px 以上