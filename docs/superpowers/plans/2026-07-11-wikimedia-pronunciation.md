# 词迹 v1.2 Wikimedia 真人发音 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 为“词迹”增加经过审核的 Wikimedia Commons / Wiktionary 真人录音、集中审核、离线缓存和完整署名能力，同时保留同口音本地合成音频和系统语音作为降级方案。

**Architecture:** 新增独立 `src/pronunciation` 模块。131 词固定映射放在静态 JSON；用户审核结果放 IndexedDB；音频响应放 Cache Storage。`PronunciationResolver` 只选择来源，`AudioPlaybackController` 只负责播放、取消、超时和降级；未经审核的候选永不进入正式播放链路。

**Tech Stack:** React 19、TypeScript 7、Vite 8、Vitest、Testing Library、Playwright、idb、Cache Storage、MediaWiki Action API、Vite PWA/Workbox。

## Global Constraints

- 播放顺序：已缓存 Wikimedia 真人录音 → 在线 Wikimedia 真人录音 → 当前口音本地音频 → 当前口音系统语音 → 发音暂不可用。
- 英式和美式独立审核；不得用另一口音冒充。
- 当前 131 词使用固定映射；新词导入后进入“词库 → 待审核发音”。
- 未审核候选不得用于卡片、沉浸模式、词库播放或自动朗读。
- 真人录音首次成功播放后自动缓存；支持手动下载今日词包。
- 每条 approved 记录必须具备作者、许可证、Commons 文件页和 HTTPS 音频 URL。
- `speechErrorsEnabled` 默认关闭；关闭时不弹窗。
- 清理真人录音缓存不得删除词库、学习进度或审核结果。
- 保留 v1.1 的 `audioUK/audioUS` 作为兜底。
- 数据库升级和备份恢复必须保留旧学习数据。
- Wikimedia API 使用匿名 CORS `origin=*`，不引入密钥和后台。
- `extmetadata` 必须去 HTML 标签，禁止 `dangerouslySetInnerHTML`。
- 同时最多播放一条；新点击取消上一音频、系统语音和请求。
- 版本为 `1.2.0`；预览 base path 为 `/ci-ji/preview-v1-2/`。

---

### Task 1: 建立 v1.2 基线与共享测试夹具

**Files:**
- Modify: `package.json`
- Create: `tests/fixtures/pronunciation.ts`
- Create: `tests/pronunciation/baseline.test.ts`
- Modify: `RELEASE_CHECKLIST.md`

- [ ] 将版本改为 `1.2.0`，增加：

```json
{
  "discover:pronunciations": "tsx scripts/discover-wikimedia-pronunciations.ts",
  "build:pronunciations": "tsx scripts/build-wikimedia-mapping.ts",
  "validate:pronunciations": "tsx scripts/validate-wikimedia-mapping.ts"
}
```

- [ ] 创建 `makeWord`、`makeSettings`、`makeApprovedRecording`、`makeCandidate` 测试构造器。
- [ ] 写失败测试，断言默认 `pronunciationSource === 'wikimedia-first'`、`cacheWikimediaAfterPlayback === true`、`speechErrorsEnabled === false`。
- [ ] 运行：`npm test -- tests/pronunciation/baseline.test.ts`，预期先失败。
- [ ] 发布清单加入 Mate X7、离线、同口音降级、署名验证。
- [ ] 提交：`test: establish v1.2 pronunciation baseline`。

---

### Task 2: 发音领域模型、设置项和 IndexedDB v2

**Files:**
- Create: `src/pronunciation/types.ts`
- Modify: `src/domain/models.ts`
- Modify: `src/domain/constants.ts`
- Modify: `src/db/schema.ts`
- Modify: `src/db/openDb.ts`
- Modify: `src/db/repositories.ts`
- Create: `tests/db/pronunciation-migration.test.ts`

**Interfaces:**

```ts
export type PronunciationStatus = 'unreviewed' | 'approved' | 'rejected' | 'missing';
export type PronunciationSourcePreference = 'wikimedia-first' | 'local-first' | 'system-only';
export interface ApprovedPronunciation {
  accent: 'en-GB' | 'en-US';
  audioUrl: string;
  compatibleAudioUrl: string | null;
  filePageUrl: string;
  author: string;
  licenseName: string;
  licenseUrl: string | null;
  sourceFileName: string;
  durationSeconds: number | null;
  mappingVersion: number;
}
export interface PronunciationReviewRecord {
  id: string;
  wordId: string;
  normalizedKey: string;
  accent: 'en-GB' | 'en-US';
  status: PronunciationStatus;
  selected: ApprovedPronunciation | null;
  origin: 'bundled' | 'user';
  reviewedAt: string | null;
  updatedAt: string;
}
```

- [ ] 写迁移测试：从 v1 数据库升级后旧 word 仍在，且出现 `pronunciationReviews`、`pronunciationCandidates`。
- [ ] `AppSettings` 增加 `pronunciationSource`、`cacheWikimediaAfterPlayback`；默认值按 Global Constraints。
- [ ] `DB_VERSION = 2`。
- [ ] 新 store：

```ts
pronunciationReviews: keyPath 'id'; indexes 'by-word', 'by-status', 'by-normalized'
pronunciationCandidates: keyPath 'id'; indexes 'by-word', 'by-accent', 'by-fetched'
```

- [ ] repository 增加 review/candidate 的 get、put、批量写、按 word/status 查询和清理接口。
- [ ] 运行：`npm test -- tests/db/pronunciation-migration.test.ts tests/db/repositories.test.ts tests/settings/SettingsProvider.test.tsx`。
- [ ] 提交：`feat: add pronunciation domain and database stores`。

---

### Task 3: 备份 v2 与 v1 兼容恢复

**Files:**
- Modify: `src/backup/backupSchema.ts`
- Modify: `src/backup/exportBackup.ts`
- Modify: `src/backup/restoreBackup.ts`
- Create: `tests/backup/pronunciationBackup.test.ts`

- [ ] 写失败测试：v2 导出包含 `pronunciationReviews`；v1 备份恢复后该数组为空且旧数据完整。
- [ ] 定义 `CompleteBackupV2`，导出 schemaVersion 2。
- [ ] `validateBackup` 同时接受 v1/v2，并将 v1 标准化为 v2 形状。
- [ ] 新设置字段使用 Zod `.default(...)`。
- [ ] 只备份审核结果，不备份候选和 Cache Storage。
- [ ] 恢复事务加入 `pronunciationReviews`。
- [ ] 运行：`npm test -- tests/backup/restoreBackup.test.ts tests/backup/pronunciationBackup.test.ts`。
- [ ] 提交：`feat: back up pronunciation reviews with v1 compatibility`。

---

### Task 4: 固定映射格式、加载器与校验器

**Files:**
- Create: `src/pronunciation/mappingSchema.ts`
- Create: `src/pronunciation/bundledMapping.ts`
- Create: `public/data/wikimedia-pronunciations.json`
- Create: `scripts/validate-wikimedia-mapping.ts`
- Create: `tests/pronunciation/bundledMapping.test.ts`

**Required JSON shape:**

```ts
type BundledAccentRecord =
  | { status: 'missing'; accent: 'en-GB' | 'en-US' }
  | ({ status: 'approved' } & ApprovedPronunciation);
interface BundledMapping {
  schemaVersion: 1;
  mappingVersion: number;
  generatedAt: string;
  words: Record<string, { british: BundledAccentRecord; american: BundledAccentRecord }>;
}
```

- [ ] 写测试：approved 缺作者/许可证/文件页时拒绝；missing 可通过。
- [ ] Zod 限制 duration 为 `null` 或 `0.2–12` 秒。
- [ ] `loadBundledPronunciationMapping()` 从 `${BASE_URL}data/wikimedia-pronunciations.json` 单次加载。
- [ ] 初始映射覆盖 131 个 normalizedKey、262 个口音槽，未审核槽显式 `missing`。
- [ ] 校验脚本检查覆盖数、重复项、HTTPS、Commons File 页面和授权字段。
- [ ] 运行：`npm test -- tests/pronunciation/bundledMapping.test.ts && npm run validate:pronunciations`。
- [ ] 提交：`feat: add validated bundled pronunciation mapping`。

---

### Task 5: Wikimedia 候选查询、清洗和排序

**Files:**
- Create: `src/pronunciation/wikimediaApi.ts`
- Create: `tests/fixtures/wiktionary-atmosphere-images.json`
- Create: `tests/fixtures/commons-atmosphere-imageinfo.json`
- Create: `tests/pronunciation/wikimediaApi.test.ts`

**Interfaces:**

```ts
searchWikimediaCandidates(word: WordEntry, signal?: AbortSignal): Promise<PronunciationCandidateRecord[]>;
rankCandidates(items: PronunciationCandidateRecord[], accent: 'en-GB' | 'en-US'): PronunciationCandidateRecord[];
stripMetadataHtml(value: string): string;
```

- [ ] Wiktionary 请求：`action=query&format=json&formatversion=2&origin=*&prop=images&titles=<word>&imlimit=max`。
- [ ] 对返回 File titles 在 Commons 请求 `prop=imageinfo&iiprop=url|mime|mediatype|extmetadata`。
- [ ] 兜底 Commons 搜索：`generator=search&gsrnamespace=6&gsrsearch=<word> pronunciation&gsrlimit=20`。
- [ ] `iiextmetadatafilter` 仅取 Artist、LicenseShortName、LicenseUrl、UsageTerms、Categories、Language。
- [ ] 只保留 AUDIO/audio MIME、HTTPS、完整授权字段、合理时长。
- [ ] 口音关键字：英式 `en-uk/en-gb/british/united kingdom`；美式 `en-us/american/united states`。
- [ ] 排序：Wiktionary关联 > 文件名精确匹配 > 口音明确匹配 > 授权完整 > 时长合理。
- [ ] 每口音最多五条；明确相反口音不得混入。
- [ ] 请求超时 8 秒，支持 AbortSignal；成功查询内存缓存 15 分钟。
- [ ] 运行：`npm test -- tests/pronunciation/wikimediaApi.test.ts`。
- [ ] 提交：`feat: discover and rank Wikimedia pronunciation candidates`。

---

### Task 6: 审核仓库与新词待审核队列

**Files:**
- Create: `src/pronunciation/pronunciationRepository.ts`
- Modify: `src/packs/commitImport.ts`
- Modify: `src/library/WordEditor.tsx`
- Create: `tests/pronunciation/pronunciationRepository.test.ts`
- Modify: `tests/packs/commitImport.test.ts`

**Interfaces:**

```ts
ensureReviewSlotsForWords(db, words): Promise<void>;
approveCandidate(db, candidate): Promise<PronunciationReviewRecord>;
rejectCandidate(db, candidateId): Promise<void>;
markAccentMissing(db, wordId, normalizedKey, accent): Promise<void>;
listPendingWords(db): Promise<string[]>;
```

- [ ] 写测试：每个新词创建 en-GB/en-US 两个 `unreviewed` 槽；事务失败时全部回滚。
- [ ] approved 只能接受明确 en-GB/en-US 候选，unknown 禁止确认。
- [ ] 用户 approved/rejected/missing 不得被后续导入覆盖。
- [ ] `commitPackImport` 同事务写 review slots。
- [ ] 手动新增词后创建 slots；编辑 normalizedKey 时迁移现有 review 的 normalizedKey。
- [ ] 运行：`npm test -- tests/packs/commitImport.test.ts tests/pronunciation/pronunciationRepository.test.ts`。
- [ ] 提交：`feat: create pronunciation review queue for new words`。

---

### Task 7: Cache Storage 与批量下载

**Files:**
- Create: `src/pronunciation/audioCache.ts`
- Create: `tests/pronunciation/audioCache.test.ts`

**Interfaces:**

```ts
getCachedPronunciation(record): Promise<CachedPronunciation | null>;
cachePronunciation(record, signal?): Promise<CachedPronunciation>;
downloadPronunciations(records, options): DownloadController;
getPronunciationCacheStats(): Promise<{ count: number; bytes: number }>;
clearPronunciationCache(): Promise<void>;
```

- [ ] cache name 固定 `ci-ji-wikimedia-audio-v1`。
- [ ] cache key 包含 audio URL 与 `mappingVersion`。
- [ ] fetch 使用 CORS、omit credentials；非 2xx 或非 audio content-type 视为失败。
- [ ] cached response 转 Blob URL，返回 `revoke()`；播放完成/取消必须释放。
- [ ] 批量下载并发 3，支持 pause/resume/cancel；单项失败不终止队列。
- [ ] 统计优先 content-length，缺失时读取 clone。
- [ ] 清理只删除该 Cache，不触碰 IndexedDB。
- [ ] 运行：`npm test -- tests/pronunciation/audioCache.test.ts`。
- [ ] 提交：`feat: cache and download Wikimedia pronunciation audio`。

---

### Task 8: Resolver 与严格同口音播放控制器

**Files:**
- Create: `src/pronunciation/pronunciationResolver.ts`
- Create: `src/pronunciation/audioPlaybackController.ts`
- Modify: `src/speech/speechService.ts`
- Create: `tests/pronunciation/pronunciationResolver.test.ts`
- Create: `tests/pronunciation/audioPlaybackController.test.ts`
- Modify: `tests/speech/speechService.test.ts`

**Plan types:**

```ts
type PronunciationPlanStep =
  | { kind: 'wikimedia-cache'; recording: ApprovedPronunciation; timeoutMs: 1000 }
  | { kind: 'wikimedia-online'; recording: ApprovedPronunciation; timeoutMs: 6000 }
  | { kind: 'local-audio'; url: string; accent: Accent; timeoutMs: 4000 }
  | { kind: 'system'; accent: Accent; timeoutMs: 3000 };
```

- [ ] 写测试：英式计划中不得出现美式真人或本地 URL。
- [ ] 用户审核优先于内置映射；user missing/rejected 阻止内置；unreviewed 不阻止已审核内置映射。
- [ ] `wikimedia-first`、`local-first`、`system-only` 三种设置生成确定计划。
- [ ] 从 speechService 暴露 `playLocalAudio`、`playSystemSpeech`；删除 opposite-accent fallback。
- [ ] 系统语音只接受精确 en-GB/en-US voice；不使用 en-AU 冒充。
- [ ] 控制器维护单一 AbortController；新播放先 cancel。
- [ ] 超时/失败进入下一 step；cancelled 立即停止；成功立即结束链路。
- [ ] 在线真人成功后异步缓存；缓存失败不改变播放成功结果。
- [ ] 运行：`npm test -- tests/pronunciation/pronunciationResolver.test.ts tests/pronunciation/audioPlaybackController.test.ts tests/speech/speechService.test.ts`。
- [ ] 提交：`feat: resolve and play same-accent pronunciation fallbacks`。

---

### Task 9: 接入 useSpeech、按钮状态与署名

**Files:**
- Modify: `src/speech/useSpeech.ts`
- Modify: `src/speech/SpeechButton.tsx`
- Create: `src/pronunciation/AttributionPanel.tsx`
- Create: `src/pronunciation/pronunciation.css`
- Modify: `src/cards/StudyCard.tsx`
- Modify: `src/immersive/ImmersiveRow.tsx`
- Create: `tests/pronunciation/useSpeech.test.tsx`
- Create: `tests/pronunciation/AttributionPanel.test.tsx`

**Hook output:**

```ts
status: 'idle' | 'loading-human' | 'playing-human' | 'playing-fallback' | 'unavailable';
lastSource: 'wikimedia-cache' | 'wikimedia-online' | 'local-audio' | 'system' | null;
attribution: ApprovedPronunciation | null;
```

- [ ] `useSpeech` 调用 resolver/controller，保持现有 `speak(word, slow?)` 使用方式。
- [ ] 失败提示关闭时 `error=null`。
- [ ] SpeechButton 提供对应 aria 状态，不阻塞揭示卡片或评分。
- [ ] 卡片仅显示“真人发音 · Wikimedia”；点击展开作者、口音、许可证、来源页和缓存状态。
- [ ] 外链使用 `target=_blank rel=noreferrer`；不渲染元数据 HTML。
- [ ] 沉浸行的署名按钮 `stopPropagation()`，不破坏展开释义逻辑。
- [ ] 运行：`npm test -- tests/pronunciation/useSpeech.test.tsx tests/pronunciation/AttributionPanel.test.tsx tests/cards/CardStudyPage.test.tsx tests/immersive/ImmersivePage.test.tsx`。
- [ ] 提交：`feat: show human pronunciation playback and attribution`。

---

### Task 10: “待审核发音”页面

**Files:**
- Create: `src/pronunciation/PronunciationReviewPage.tsx`
- Create: `src/pronunciation/PronunciationReviewCard.tsx`
- Modify: `src/library/LibraryPage.tsx`
- Modify: `src/app/routes.tsx`
- Create: `tests/pronunciation/PronunciationReviewPage.test.tsx`

- [ ] 词库显示去重后的待审核单词数，并链接 `/library/pronunciation-review`。
- [ ] 筛选：今日词包、全部待审核、仅缺英式、仅缺美式、已标记缺失。
- [ ] 候选为空或超过 15 分钟时查询；网络失败保留 unreviewed，不自动 missing。
- [ ] 候选试听不进入正式 resolver，不自动缓存；切换候选立即停止上一条。
- [ ] unknown 候选“确认使用”禁用。
- [ ] 操作：确认、换一条、拒绝、标记缺失、跳过、确认并下一词、双口音确认。
- [ ] “双口音确认”只有两侧均选中明确匹配候选时启用。
- [ ] 运行：`npm test -- tests/pronunciation/PronunciationReviewPage.test.tsx tests/library/LibraryPage.test.tsx`。
- [ ] 提交：`feat: add centralized pronunciation review workflow`。

---

### Task 11: 今日词包下载、缓存管理与设置页

**Files:**
- Create: `src/pronunciation/DownloadPackAudioButton.tsx`
- Modify: `src/home/HomePage.tsx`
- Modify: `src/settings/SettingsPage.tsx`
- Modify: `src/settings/SettingsProvider.tsx`
- Create: `tests/pronunciation/DownloadPackAudioButton.test.tsx`

- [ ] 下载预览显示可下载数、缺失数、当前口音和可获得时的预计空间。
- [ ] 支持当前口音或双口音临时下载，不修改全局设置。
- [ ] 首页仅在今日词包存在时显示“下载今日真人发音”。
- [ ] 设置增加真人优先/本地优先/仅系统、首次播放后缓存、失败提示。
- [ ] 诊断区显示实际来源、缓存条数/空间、本地备用状态和精确系统语音数量；删除硬编码“131/131”。
- [ ] “清除真人录音缓存”确认后只清 Cache Storage，并刷新统计。
- [ ] 设置页版本显示 `1.2.0`；试听 atmosphere 走 resolver。
- [ ] 运行：`npm test -- tests/pronunciation/DownloadPackAudioButton.test.tsx tests/settings/SettingsProvider.test.tsx`。
- [ ] 提交：`feat: download and manage human pronunciation cache`。

---

### Task 12: 131 词发现、人工审核与映射构建

**Files:**
- Create: `scripts/discover-wikimedia-pronunciations.ts`
- Create: `scripts/build-wikimedia-mapping.ts`
- Create: `docs/pronunciation/wikimedia-review.csv`
- Create: `docs/pronunciation/README.md`
- Modify: `public/data/wikimedia-pronunciations.json`
- Create: `tests/pronunciation/mappingBuild.test.ts`

**CSV header:**

```csv
word,normalizedKey,accent,status,audioUrl,compatibleAudioUrl,filePageUrl,author,licenseName,licenseUrl,sourceFileName,durationSeconds,reviewedBy,reviewedAt,notes
```

- [ ] 发现脚本读取当前 131 词包，逐词查询，每请求间隔 250ms；失败记录后继续并最终非零退出。
- [ ] Node 请求 User-Agent：`CiJiPronunciationAudit/1.2 (https://github.com/lsad-max/ci-ji)`。
- [ ] 每个词必须具备 en-GB/en-US 两行审核；状态仅 approved/missing。
- [ ] 人工试听并排除例句、多人、音乐、明显噪声、错误词形、地区不明和授权不完整录音。
- [ ] build script 对 normalizedKey 排序、稳定生成 JSON，mappingVersion 自动加 1。
- [ ] approved 必须字段完整；missing 只保留 status/accent。
- [ ] 至少抽样复核 20 条，含 atmosphere、climate、drought、earthquake、ecosystem。
- [ ] 运行：

```bash
npm run discover:pronunciations
npm run build:pronunciations
npm run validate:pronunciations
npm test -- tests/pronunciation/mappingBuild.test.ts tests/pronunciation/bundledMapping.test.ts
```

- [ ] 提交：`data: add reviewed Wikimedia pronunciation mapping`。

---

### Task 13: PWA、版本和 v1.2 预览部署

**Files:**
- Modify: `vite.config.ts`
- Modify: `.github/workflows/deploy-pages.yml`
- Create: `tests/pwa/pronunciationCachePolicy.test.ts`
- Modify: `tests/app/basePath.test.ts`

- [ ] `includeAssets` 加入 `data/wikimedia-pronunciations.json`。
- [ ] Workbox 保留本地 mp3 兜底，不增加远程 ogg/oga/webm；远程真人音频只由 `audioCache.ts` 管理。
- [ ] cacheId 由 base path 派生，避免正式站和 preview 相互污染。
- [ ] 联合部署 checkout `feature/v1.2-wikimedia-pronunciation`，输出到 `site/preview-v1-2`。
- [ ] workflow 验证映射文件存在且 `Object.keys(words).length === 131`。
- [ ] 不再把“262 个 MP3”作为 v1.2 成功条件；仍验证本地 fallback 构建资源。
- [ ] 运行：

```bash
npm run typecheck
npm test
CI_JI_BASE_PATH=/ci-ji/preview-v1-2/ npm run build
```

- [ ] 提交：`ci: prepare v1.2 Wikimedia pronunciation preview`。

---

### Task 14: E2E、离线和 Mate X7 实机验收

**Files:**
- Create: `e2e/pronunciation.spec.ts`
- Modify: `e2e/helpers.ts`
- Modify: `RELEASE_CHECKLIST.md`
- Create: `docs/pronunciation/mate-x7-verification.md`

- [ ] Playwright 拦截 Wiktionary、Commons 和 upload.wikimedia.org，返回固定 API fixtures 与可播放短 WAV。
- [ ] 场景：approved 真人播放和署名；网络失败转同口音本地；相反口音不请求；缓存后离线播放；新词审核；清缓存不丢掌握度。
- [ ] 运行：

```bash
npm run typecheck
npm test
npm run build
npm run test:e2e
npm run validate:pronunciations
```

- [ ] 部署并验证：`https://lsad-max.github.io/ci-ji/preview-v1-2/`。
- [ ] Mate X7 记录设备、HarmonyOS 6、浏览器完整版本、日期、40 位提交 SHA，以及英式/美式真人、断网缓存、同口音降级、连续点击、署名、升级保留结果。
- [ ] 若应用无声：先直接打开 approved 的 upload.wikimedia.org 原始 URL。原始 URL 无声则更换候选；原始 URL 有声则保存 Console、Network、Cache Storage 证据并继续修复，禁止宣布完成。
- [ ] 仅实机关键项全部通过后提交：`test: verify v1.2 pronunciation on web and Mate X7`。

---

## Final Verification Gate

```bash
npm ci
npm run typecheck
npm test
npm run validate:pack
npm run validate:pronunciations
CI_JI_BASE_PATH=/ci-ji/preview-v1-2/ npm run build
npm run test:e2e
```

发布前必须确认：

1. 映射覆盖 131 个 normalizedKey 和 262 个口音槽。
2. 每条 approved 具有完整授权元数据。
3. missing 只进入同口音降级，不借用另一口音。
4. v1 数据库升级、v1 备份恢复、v2 备份往返通过。
5. 审核、批量下载、缓存清理和离线播放通过。
6. GitHub Pages v1.2 预览部署为绿色。
7. Mate X7 实机关键项全部通过。

达到以上门槛后才能创建合并到 `main` 的 Pull Request。