# 阅接力（YueJieLei）项目备忘

## 依赖
- `base64-arraybuffer`：前端上传封面图到 Supabase Storage 时，把 FileReader 读出的 base64 转成 ArrayBuffer（沙箱内 Blob/File 直传会 400）。

## 架构
- 云服务：Meoo Cloud（Supabase），表 members / books / loans / reminders / chat_messages + profiles（登录用户档案）。RLS 为「登录即可读写」：业务表策略 TO authenticated USING(true)（migration 20261022_035450 去掉了 authorized 白名单门槛；chat_messages 按 thread_key=auth.uid() 隔离；profiles own-only）。profiles 表仍保留 authorized/is_admin 列但前端不再使用。
- 认证：纯短信验证码一键登录（无密码）。sendLoginCode = signInWithOtp({phone, shouldCreateUser:true, data:{username}}) 自动区分新老用户；verifyLoginCode = verifyOtp(type:'sms')，新用户成功后在 nickname 步骤用 updateMyProfile 写昵称。login.tsx 三步状态机 phone→code(60s 重发倒计时)→nickname。扫码登录=通行码核验（YJL-XIAOZHAN-2026）模拟，非真二维码扫描。管理员口令通道与 /admin 授权页已于 2026-10-22 按用户要求整体移除（admin-tools Edge Function 已删除）。
- Edge Function `ai-assistant`（Deno TS，部署用 `-j false` 免 JWT）：四模式 mode=chat（对话借书 + AnySearch 联网检索）/ mode=enrich（AI 补全书目草稿）/ mode=recommend（AI 生成好书推荐美文，返回 {text}）/ mode=vision（qwen3-vl-plus 看封面图流式识别书名/作者/年级/分类，done 帧下发 draft）。内部调 Meoo LLM 兼容模式接口，AK 从环境变量读取；AnySearch MCP 走 JSON-RPC over Streamable HTTP。vision 约束：imageUrl 支持 http(s) URL 与 data:image base64；带 auth_key 的 CDN 查询串会被模型侧拒收（函数自动剥掉 ?）；公网 URL 先做 8s GET 预检，不可达时返回 422 `IMAGE_UNREACHABLE:`，前端 recognizeBookFromCover 自动降级压缩后本地图 base64 直传重试。
- LLM 结构化协议：回复末尾输出 `[[CARDS]]{"bookIds":[...]}[[/CARDS]]` 与 `[[ACTION]]{"type":"borrow","bookId":"..."}[[/ACTION]]`，函数流式转发时过滤标记，done 帧下发 text/cards/action，前端 AssistantDrawer 消费。
- 前端状态：TanStack Query 管服务端数据（books/loans refetchInterval=REFRESH_MS 1 秒轮询实现"实时刷新"），`src/stores/shelf.ts` 用 useSyncExternalStore 管全局状态（profile: Profile|null / authChecked / books / loans / assistantOpen / borrowRequest）。
- 借阅人昵称链路：createLoan 后同步 updateBook({status:'borrowed', borrowed_by_nickname})；归还/取消时置 null。书架 ShelfPage 木质层板图形（styles.css .bookcase/.shelf-board/.shelf-plank）陈列封面（object-contain 防变形），借出压暗显示"已被 X 借走"。
- 语音输入：`src/components/VoiceInput.tsx` VoiceInputButton 基于 Web Speech API（zh-CN），已挂到登录页、上传页、书架搜索、助手输入、借阅联系方式等全部文本输入位。
- 路由：TanStack Router 文件路由，`_layout` 布局壳（AppShell 内部渲染 Outlet，路由 component 直接写 AppShell 不传 children）；/login 为独立无壳路由。
- 还书提醒：ReminderWatcher 前端扫描 daysLeft ∈ {45,30,15,7,3} 档位，reminders 表 UNIQUE(book_id,bucket) 幂等去重，站内 toast 提醒。

## 踩坑记录
- ✅ 「手机端上架不了」根因（2026-10-22 修复）：storage.objects 原策略仅 TO anon 且 INSERT WITH CHECK 为 NULL，authenticated 会话上传封面被 RLS 直接拒绝 → migration 20261022_041738 重建 auth_insert_book_covers / public_select_book_covers（TO authenticated, anon + bucket_id 字面量）。uploadCover 现链路：compressImage(canvas ≤1MB)→FileReader base64→decode() ArrayBuffer→upload；失败按 duplicate/policy 分类提示。
- ✅ vision 冒烟实测：g.cdn.meoo.host 裸地址 403、带 auth_key 百炼拉图失败——沙箱 CDN 外链不可作识图输入；本项目 storage public URL（database.meoo.xyz/storage/v1/object/public/…）可正常拉图。
- ❌ Storage API DELETE 对象需桶名路径且受 RLS 限制，删不掉时用 cloud query 查 storage.objects 确认存在即可，勿反复重试。
- ✅ 现行 client.ts 已导出 projectUrlId → 所有裸 fetch 云函数必须带 'OneDay-App-Id': projectUrlId（services/shelf.ts fnHeaders()）；curl 公网直连测试同样要加该 header，且必须用 .env 的 VITE_SUPABASE_URL 域名（api.meoo.host/functions 会返回网页兜底 HTML）。
- ❌ supabase update 传 `Record<string, unknown>` 会 TS2345 → patch 用显式字面量对象（`as const`）。
- ❌ JSONB 字段与前端联合类型互转需经 `as unknown as X`，直接断言报 TS2352/TS2322。
- Write 上传封面必须用 BUCKET_ID（`src/services/shelf.ts` 常量）+ decode(base64) 的 ArrayBuffer。
- ❌ `_layout.index.tsx` 存在时必须删除模板 `src/routes/index.tsx`，否则路由生成器报 Conflicting configuration paths "/"。
