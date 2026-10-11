# AGENTS.md — sen-yue-read-score (粵語朗誦評估 PWA)

> **Resumability doc** — 任何 program / agent 開呢個 project 讀呢個 file 就知 status。
> User: kencheng | Last updated: 2026-07-31

---

## 1. Project 概要

Single-file SPA，GitHub Pages host。學生錄音 → 4-dim 評分 → TTS side-by-side 對比。
**零 backend、純 localStorage、私隱 by design**。

| 項目 | 值 |
|------|-----|
| Path | `/Users/kencheng/workspace/vs code/education/chinese/reading-para/` |
| Entry | `index.html` (6093 → 7025 lines after Sprint 6) |
| Host | `https://ihateusingai-beep.github.io/sen-reading-para/` |
| Repo | `origin/main` 喺 GitHub |
| `__BUILD_ID` | `ui-simplify-l2 @ 2026-07-22` |
| `_SCHEMA_VERSION` | 3 |

---

## 2. Stack

- **純 HTML + vanilla JS** (冇 framework, 冇 build step)
- **localStorage keys**: `yuePassages`, `yueTypes`, `yueAdminSettings`, `yueStudents`, `yueProgress`, `yueAudioBaselines`, `yueCustomContentV1`, `yueDebug`, `yueSchemaVersion`
- **Whisper-tiny-web** ASR (CDN 載入, WebGPU/CPU fallback)
- **Web Speech API** TTS (zh-HK male/female)
- **PWA**: GitHub Pages, HTTPS-only (mic permission 要 HTTPS / localhost)
- **iOS PWA standalone** (added to home screen) 用 WebKit，跟 Safari

---

## 3. Core Loop

1. 學生揀文章 / 詞彙 → 評估頁
2. **錄音** (MediaRecorder + iOS PWA MIME ladder — c6 hotfix force mp4)
3. **ASR transcribe** (Whisper 28s avg, 或 Web Speech 即時但唔可靠)
4. **4-dim 評分**: accuracy / coverage / fluency / expression + filler
5. **Result page** 顯示對比 + click-to-replay + 錯字重讀

---

## 4. Sprint 6 已 ship (9 commits, ahead of origin)

| # | Commit | Subject | LoC |
|---|--------|---------|-----|
| 1 | `ec9479c` | feat(c1): storage + schema v3 | +184 |
| 2 | `d1ce572` | feat(c2): teacher settings UI | +278 |
| 3 | `1bf1833` | feat(c3): TTS voice pre-gen | +89 |
| 4 | `b2e555a` | feat(c4): student vocab mode + diff | +279 |
| 5 | `5cb99de` | chore(c5): audit + smoke | +26 |
| 6 | `01aa43b` | **fix(c6): iOS MIME + typo** | +21 / -10 |
| 7 | `79f9d9c` | **fix(c7): tts.miss 404 cache** | +43 / -2 |
| 8 | `6e0b09b` | **feat(c8): 12 default vocabs** | +64 / -11 |
| 9 | `4d31cc0` | **ui(c9): L2 simplify student UI** | +23 / -10 |

- **File**: 6093 → 7025 lines (+932 net)
- **Status**: 4 commits ahead of `origin/main` (c6, c7, c8, c9 仲未 push)
- 仲有 `4a7b0a5 feat(auth): school default teacher password Hcwmc` 喺 origin (user 加嘅)

---

## 5. Sprint 6 ships 嘅 feature

### Teacher 自訂詞彙 / 短文
- 老師 Settings → 「📝 自訂詞彙 / 短文」section
- 12 default vocabs injected 入 `state.customContent` (光/雨/春天/同學/學校/老師/書本/爸爸/媽媽/朋友/謝謝/你好)
- 老師可以 add/edit/delete custom items
- Default vocab 用 `isDefault: true` flag 防老師刪除

### Student 詞彙 mode (M2)
- Click vocab card → TTS reference + record + char-level diff
- 1-to-1 char alignment, color-coded highlight (綠/紅/灰/黃)
- 學生手動輸入「我讀咗咩」做 diff 對比
- ASR 對 1-6 字 vocab 唔可靠 → manual input fallback

### Filter tabs
- `[全部]` / `[預設]` / `[自訂]` / `[散文]` / `[詩詞]` / `[詞彙]`
- `showPassageSelector` merge `state.passages` + `state.customContent.items`

---

## 6. Hotfix c6/c7 — iOS PWA + tts.miss

### c6: iOS MIME ladder
- **問題**: iOS Safari 17+ PWA standalone `isTypeSupported('audio/webm')` return true，但 MediaRecorder 產 corrupted bytes，Whisper transcribe garbage (e.g. 「Ойained今天的 Lazarởng」)
- **Fix**: 加 `_isIOSPWA()` UA sniff (iPad/iPhone/iPod + iPadOS 13+ MacIntel+maxTouchPoints)；iOS 直接 skip webm candidate
- **Files**: `pickMimeType()` line ~1140

### c7: tts.miss 404 cache
- **問題**: 每次 render re-fetch TTS mp3，404 唔 cache，連環 retry (6 個 tts.miss in 60s)
- **Fix**: `_ttsMissedForId` + `_markTtsMiss(id, ttlMs)` module-scope state
  - Custom passage miss → 24h cache (expected miss, TTS file 唔存在 by design)
  - Default passage miss → 30s cache (retry)
- **Files**: `_loadTtsForPassage` line ~3200, `_clearTtsCache` line ~3284

---

## 7. c9 L2 Student UI Simplify

Hide 學生 result page noise:
- 🐛 DEBUG 浮動 panel — `__debugRenderPanel` guard `state.mode === 'student'`
- Color legend (綠=啱 紅=錯 灰=漏咗 黃=多咗) — removed
- Play mode toggle (chain/mix) — `if (false && replayable)` skip
- 「💡 新功能」tips — localStorage permanent dismiss

**Bonus**: `exitTeacherMode` 自動 disable debug + remove panel (防 teacher 開 debug 後漏俾學生睇)

---

## 8. Architectural decisions (重要!)

### isDefault flag pattern
Default vocab 入 `state.customContent` 而唔係 `state.passages`：
- Reuse `_getCustomItemById` + `startVocabPractice` without refactor
- `_updateCustomItem` / `_deleteCustomItem` guard `isDefault` (return `CANNOT_MODIFY_DEFAULT` / `CANNOT_DELETE_DEFAULT`)
- `_renderCustomContentList` filter `!it.isDefault`
- `showPassageSelector` merge: `source: it.isDefault ? 'default' : 'custom'`

### Schema versioning
- `_SCHEMA_VERSION = 3` 喺 line 545
- v2→v3 no-op (新 key `yueCustomContentV1`)
- Future migration: 寫喺 `_migrateIfNeeded` (line 515)

### iOS MIME ladder
```js
const isIOS = _isIOSPWA();
const candidates = isIOS
  ? ['audio/mp4;codecs=mp4a.40.2', 'audio/mp4', '']
  : ['audio/webm;codecs=opus', 'audio/webm', 'audio/mp4;codecs=mp4a.40.2', 'audio/mp4', ''];
```

### TTS voice availability (NOT byte capture)
- **iOS Safari PWA 唔可以 capture `speechSynthesis` output** (OS audio path 唔係 Web Audio graph)
- 改用 silent test speak 5s timeout 試 voice availability → mark `ttsStatus: 'ready' | 'failed'`
- 學生 click 時即時 `speechSynthesis.speak()` (latency ~300-500ms, acceptable)

### 5-bug-family check pre-ship (per memory rule 13)
- F1: default-state desync
- F2: reset-on-render
- F3: enable-condition off-by-concept
- F4: silent data loss (TTS pre-gen fail handling)
- F5: dead code via name collision

---

## 9. Known Issues / Gotchas

| # | Issue | Workaround |
|---|-------|------------|
| 1 | iOS Safari PWA 唔可以 capture `speechSynthesis` audio bytes | Voice availability check only |
| 2 | Vocab 1-6 字 ASR 唔可靠 (Whisper 對 1 字「光」誤判) | Manual text input for diff |
| 3 | Whisper inference ~28s 喺 iOS PWA (WebAssembly single-thread, 無 GPU) | Acceptable for passage; vocab use TTS-only |
| 4 | localStorage 5MB quota (vocab + TTS audio cache) | Quota guard 4MB warn / 4.75MB reject |
| 5 | GitHub Pages HTTPS required (file:// / http:// localhost 唔 work for mic) | Deploy via gh-pages only |
| 6 | Sprint 5 race 4 (auto-replay on first-click) — verified race-free by commit `4693b57` | 唔好 refactor `_playTtsAutoReplay` heuristic |
| 7 | `Module name, 'undefined'` prewarm error (first `init.prewarm.whisper.start`) | Second transcribe retry auto-fixes; cosmetic log only |

---

## 10. File Structure (key sections)

| Section | Line | Description |
|---------|------|-------------|
| `__BUILD_ID` | 27 | Bump 每次 commit (date-based) |
| `_SCHEMA_VERSION` | 545 | 3 (v2→v3 no-op) |
| `_migrateIfNeeded` | 515-531 | Schema version migration |
| Custom content storage | 540-702 | `_CUSTOM_CONTENT_*` helpers |
| `_isIOSPWA` | 1132 | iOS UA sniff |
| `pickMimeType` | 1140 | iOS-aware MIME ladder |
| `__debugRenderPanel` | 1462+ | L2 guard `state.mode === 'student'` |
| `showPassageSelector` | 1809+ | Default + custom merge + filter |
| `startVocabPractice` | 1956+ | Vocab mode entry (M2) |
| `_vocab*` functions | 2022-2160 | Vocab mode recording + diff |
| `_loadTtsForPassage` | 3190+ | TTS fetch + miss cache |
| `_clearTtsCache` | 3280+ | Cache reset (includes miss) |
| `_renderDiffHighlight` | 4240+ | Sprint 4 mode toggle (L2 disabled) |
| `_DEFAULT_PASSAGES` | 4709 | 5 prose + 5 poetry |
| `_DEFAULT_VOCABS` | 4724 | 12 Cantonese vocabs |
| `_DEFAULT_TYPES` | 4710 | prose / poetry / vocab |
| `init()` | 4712+ | Load + inject default vocabs |
| `exitTeacherMode` | 5014+ | L2 disable debug + remove panel |
| SMOKE TEST | 587+ | 22 cases inline |
| `App` public surface | ~6600+ | `window.App` namespace |

---

## 11. How to continue (Resume protocol)

### Step 1: 確認 status
```bash
cd "/Users/kencheng/workspace/vs code/education/chinese/reading-para"
git log --oneline -10      # 應該見 9 commits + 1 auth commit (4a7b0a5)
git status                  # working tree: 4 uncommitted commits (c6, c7, c8, c9) + 呢個 AGENTS.md
grep "__BUILD_ID = " index.html | head -1
grep "_SCHEMA_VERSION = " index.html | head -1
```

### Step 2: Push + deploy
```bash
git add index.html AGENTS.md
git commit -m "docs: add AGENTS.md resume doc + ship 4 uncommitted commits"
git push origin main
# 然後喺 iPad Safari → https://ihateusingai-beep.github.io/sen-reading-para/
# 跑 22 個 smoke test (line 587+)
```

### Step 3: 任何 fix/feature 流程
1. **Pre-flight recon** (per memory rule 6) — `git status` + `git log` + file mtime + drift table
2. **Plan + spec** if >100 LoC (per memory rule 5)
3. **5-bug-family check** (per memory rule 13) pre-ship
4. **Bump `__BUILD_ID`** (date-based: `feature-name @ YYYY-MM-DD`)
5. **Commit + push** to `main`

### Step 4: Audit gates (per memory rule 5)
- >500 LoC → 1.5x budget
- >100 LoC per step → 10min mini-audit
- Multi-version spec → 5-bug-family pre-ship

---

## 12. Scope Lock (重要!)

**Out of scope** (per user 2026-06-19 鎖):
- ❌ Roadmap D1-D7 dimensions (phoneme diff / tone contour / prosody / D7 clarity)
- ❌ AudioWorklet rewrite
- ❌ Backend tools (MFA / Praat / Azure) — 破壞本地私隱
- ❌ Gamification (cert 用咗 mix N 次)
- ❌ Emotion TTS, volume slider, L/R pan

**Default posture**: **fixes only**
- ✅ Bug fixes
- ✅ Voice swap
- ✅ Default mode 改
- ✅ 老師 add custom passage 嘅 TTS fallback
- ❌ 唔 propose 新 features without **explicit** user ask

**Apply when**: 任何 session 開頭想 propose 新 sprint / Roadmap 規劃新方向 / Drift 去「加多樣嘢啦」嘅念頭 — STOP，問 user 確認先 proceed。

---

## 13. Memory Rules That Apply

| Rule | Description | Apply when |
|------|-------------|------------|
| §1 | Secrets 唔入 chat; 落 `~/.app/.env` + chmod 600 | commit 前 check |
| §5 | Senior audit gates (1.5x budget + mini-audit) | >500 LoC change |
| §6 | Pre-flight recon 30s | 任何 session 開頭 |
| §7 | Routing-layer change cadence | 2 routing changes in 60h → pause |
| §8 | Multi-agent reviewer gate | 唔適用 (single agent / single file) |
| §9 | Token Plan quota | 留意 batch blocked |
| §11 | Single-file HTML `</script>` trap | inline JS 入 string 唔好 |
| §12 | Memory hygiene | Project → `AGENTS.md` (此 file) |
| §13 | 5-bug-family check | Multi-version spec ship 前 |
| §14 | CI gates (coverage + a11y) | v2 才考慮 |

---

## 14. Open questions (defer to v2)

- **Vocab mode 4-dim 評分**: 1-6 字 score 唔準 — defer
- **Multi-device sync**: 要 backend → 破壞私隱 lock — out of scope
- **Custom vocab TTS audio cache**: iOS PWA 唔支援 capture — manual voice check only
- **Export/import custom vocab JSON**: v2 spec
- **ASR auto-fill for vocab manual input**: Web Speech / Whisper 對 1-6 字唔可靠 — defer

---

## 15. Speed Lookup (cheat sheet)

| 任務 | Quick action |
|------|--------------|
| Bump build | Edit `__BUILD_ID` (line 27) |
| Bump schema | Edit `_SCHEMA_VERSION` (line 545) + add migration case in `_migrateIfNeeded` |
| Add new type | 加 `_DEFAULT_TYPES` object + showPassageSelector filter handle |
| Add default passage | 加 `_DEFAULT_PASSAGES` array |
| Add default vocab | 加 `_DEFAULT_VOCABS` array |
| Hide teacher feature in student | `state.mode === 'student'` guard |
| Add iOS-aware MIME | 用 `_isIOSPWA()` + `pickMimeType()` |
| Cache TTS miss | 用 `_markTtsMiss(id, ttlMs)` |
| Bump smoke test | Edit `MANUAL SMOKE TEST CHECKLIST` (line 587+) |
| Force iOS mp4 | `pickMimeType()` 自動 detect；唔使 caller 知 |

---

## 16. Test Data Fixtures

### 12 default vocabs (`_DEFAULT_VOCABS`)
| ID | 字 | | ID | 字 |
|----|----|---|----|----|
| vocab-001 | 光 | | vocab-007 | 書本 |
| vocab-002 | 雨 | | vocab-008 | 爸爸 |
| vocab-003 | 春天 | | vocab-009 | 媽媽 |
| vocab-004 | 同學 | | vocab-010 | 朋友 |
| vocab-005 | 學校 | | vocab-011 | 謝謝 |
| vocab-006 | 老師 | | vocab-012 | 你好 |

### 5 default prose (`_DEFAULT_PASSAGES`)
春天的校園 / 我的家人 / 快樂的一天 / 下雨天 / 可愛的小狗

### 5 default poetry (`_DEFAULT_PASSAGES`)
靜夜思 / 詠鵝 / 春曉 / 憫農 / 登鸛雀樓

### 老師 add custom vocab for testing
- 光 (1 字)
- 雨點 (2 字)
- 春天來了 (4 字) — 短文 mode test
- 你好嗎 (3 字)
- 香港人 (3 字)
- 廣東話發音 (5 字)

### Smoke test (22 cases 喺 line 587+)
1-4: Add vocab valid/empty/too long/duplicate
5: Add passage 10-50 字
6: Edit vocab
7-8: Delete + clear all
9: TTS pre-gen fail
10-12: Filter [全部] [預設] [自訂]
13-15: Vocab mode (record + diff)
16: Custom passage 4-dim
17: Voice not available
18: Edit-while-pregen
19: Reload persistence
20-21: Quota 80%/95%
22: Privacy 0 external request

---

## 17. Resume 入口 (TL;DR)

> 你 (agent) 喺另一個 program 開呢個 project，第一件事：

```bash
# 1. 讀呢個 file
cat AGENTS.md

# 2. 確認 status
cd "/Users/kencheng/workspace/vs code/education/chinese/reading-para"
git log --oneline -10
git status

# 3. Push 4 uncommitted commits
git push origin main

# 4. iPad PWA live verify 22 cases
# → https://ihateusingai-beep.github.io/sen-reading-para/

# 5. 等 user 報 issue / 答問題
```

如果 user 開新 sprint：
1. Read `AGENTS.md` §12 (Scope Lock) 先
2. Pre-flight recon (per memory rule 6)
3. Spec if >100 LoC
4. Ship 5 commits per spec
5. 5-bug-family pre-ship check
6. Bump `__BUILD_ID`
7. Commit + push
