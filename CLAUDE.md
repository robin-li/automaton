# CLAUDE.md - Conway Automaton

## Project Overview

Conway Automaton 是一個 **Sovereign AI Agent Runtime** — 自主 AI 代理運行時。代理擁有自己的以太坊錢包，透過合法工作賺取生存所需的 USDC，在生存壓力下自主運行。

- **版本**: 0.2.1
- **授權**: MIT
- **倉庫**: https://github.com/Conway-Research/automaton.git
- **官網**: https://conway.tech

## Tech Stack

- **語言**: TypeScript 5.9 (strict mode, ES2022, ESM)
- **運行時**: Node.js >= 20.0.0
- **套件管理**: pnpm 10.28.1
- **資料庫**: better-sqlite3 (SQLite)
- **區塊鏈**: viem (EVM/Base), @solana/web3.js (Solana), siwe (Sign-In With Ethereum)
- **AI 推理**: openai SDK, js-tiktoken
- **測試**: vitest 2.0
- **構建**: tsc

## Common Commands

```bash
pnpm install          # 安裝依賴
pnpm build            # 編譯 TypeScript (tsc) + 構建所有子套件
pnpm dev              # 開發模式 (tsx watch)
pnpm typecheck        # 類型檢查 (不輸出)
pnpm test             # 運行所有測試
pnpm test:coverage    # 測試覆蓋率報告
pnpm test:security    # 安全相關測試
pnpm test:financial   # 財務相關測試
pnpm clean            # 清除 dist/
```

## Project Structure

```
src/
├── index.ts              # 入口點
├── config.ts             # 配置載入
├── types.ts              # 全域類型定義 (~39KB)
├── agent/                # ReAct 循環核心 (loop, tools, system-prompt, policy-engine)
│   └── policy-rules/     # 安全策略規則 (authority, financial, command-safety, path-protection)
├── state/                # SQLite 持久層 (database, schema, 11 個 migration)
├── memory/               # 四層記憶系統 (working, episodic, semantic, procedural)
├── orchestration/        # 多代理協調 (orchestrator, planner, task-graph, messaging)
├── heartbeat/            # 排程守護進程 (DurableScheduler, cron 任務)
├── identity/             # 錢包與鏈身份 (EVM + Solana, SIWE)
├── inference/            # 模型路由與預算控制
├── conway/               # Conway Cloud API 客戶端 (credits, x402 支付)
├── replication/          # 子代理生成與族譜
├── registry/             # ERC-8004 鏈上註冊
├── survival/             # 信用監控與生存等級 (Normal → Low Compute → Critical → Dead)
├── skills/               # 插件系統
├── social/               # 代理間社交通訊
├── soul/                 # SOUL.md 身份文檔演化
├── self-mod/             # 運行時自我修改 (audit-logged)
├── setup/                # 首次運行設定精靈
├── git/                  # 狀態版本控制
├── observability/        # 日誌與指標
├── ollama/               # Ollama 本地模型支援
└── __tests__/            # ~54 測試套件, ~1900 測試用例
packages/cli/             # Creator CLI (status, logs, fund, send)
```

## Architecture Key Points

- **ReAct Loop**: Think → Act → Observe → Persist，核心在 `src/agent/loop.ts`
- **Policy Engine (3 層)**: Authority → Financial → Command Safety，所有工具調用前驗證
- **Memory (4 層)**: Working (當前上下文) → Episodic (歷史壓縮) → Semantic (知識庫) → Procedural (學習的技能)
- **57+ 內建工具**: 定義在 `src/agent/tools.ts`
- **Heartbeat Daemon**: 背景排程任務，即使代理休眠也持續運作
- **Chain-Aware**: 雙鏈支援 (EVM + Solana)，透過 `ChainIdentity` 抽象
- **Constitution**: 三條不可變法則 (Never harm > Earn existence > Never deceive)

## Configuration

- 運行時配置: `~/.automaton/automaton.json`
- 排程配置: `~/.automaton/heartbeat.yml`
- 錢包: `~/.automaton/wallet.json` (mode 0600)
- 資料庫: `~/.automaton/state.db`

## Testing Conventions

- 測試位於 `src/__tests__/`
- 使用 vitest，超時 30 秒
- 覆蓋率門檻: statements 60%, branches 50%, functions 55%, lines 60%
- 測試分類: security, financial, general

## Code Conventions

- ESM only (`"type": "module"`)，import 路徑需帶 `.js` 擴展名
- Strict TypeScript，所有類型在 `src/types.ts` 中定義
- 資料庫 schema 使用遞增式 migration (`src/state/schema.ts`)
- 自我修改操作必須通過 audit log 記錄
- Constitution 相關文件受保護，不可修改

## Key Documentation

- `ARCHITECTURE.md` — 完整系統設計文檔 (~39KB)
- `DOCUMENTATION.md` — 使用者/操作者參考手冊 (~41KB)
- `constitution.md` — 不可變憲法 (三法則)
- `README.md` / `README.zh-CN.md` / `README.zh-TW.md` — 多語言說明
