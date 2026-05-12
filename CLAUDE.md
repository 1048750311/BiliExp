# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Run (source mode)
node main.js start    # 启动抽奖
node main.js check    # 中奖检查
node main.js clear    # 清理动态和关注
node main.js account  # 查看账号信息
node main.js login    # 扫码登录更新 Cookie
node main.js update   # 检查更新

# NPM shortcuts
yarn start / yarn check / yarn clear / yarn account / yarn login / yarn update

# Lint
yarn lint          # eslint lib/ test/ *.js
yarn lint-fix      # eslint --fix

# Test
yarn test          # node test/index.js

# Build (requires bash)
yarn pkg           # 打包为可执行文件 (script/build/pkg.sh)
```

## Configuration Files

Two files must exist in the working directory at runtime:

- **`env.js`** — exports `account_parm` (COOKIE, push tokens, multi-account array). Template: `env.example.js`.
- **`my_config.js`** — exports `default_config` plus per-account overrides `config_1`, `config_2`, … Template: `my_config.example.js`.

Config is loaded fresh each loop iteration via `delete require.cache[config_file]` in `lib/data/config.js:raw_config()`.

## Architecture

### Entry & Dispatch (`main.js`)

1. Loads `lib/data/env.js` → sets process env vars (COOKIE, push keys, etc.)
2. Loads `lib/data/config.js` → merges `default_config` into the singleton `config` object
3. Sets `process.env.lottery_mode` from `process.argv[2]`
4. For multi-account mode (`ENABLE_MULTIPLE_ACCOUNT`): iterates the account array sequentially, re-entering `main()` for each account with overridden env vars
5. For single-account mode: calls `global_var.init(COOKIE, NUMBER)` then switches on `lottery_mode`

### Global State (`lib/data/global_var.js`)

Singleton key-value store. `init()` parses COOKIE to extract `myUID` and `csrf`, generates a random `buvid3` if absent, and builds the `Lottery` array — an ordered flat list of `[sourceType, sourceValue]` pairs derived from `config.LotteryOrder`.

### Lottery Pipeline (`lib/lottery.js` → `lib/core/monitor.js` → `lib/core/searcher.js`)

- `start()` drives an event loop via `event_bus`: emits `Turn_on_the_Monitor` to process each entry in the `Lottery` array one at a time.
- `Monitor` extends `Searcher`. Its `init()` checks the follow partition, loads the attention list, then calls `startLottery()`.
- `Searcher` has five `getLotteryInfoBy*` methods, one per source type:
  - `UIDs` → scans a user's feed pages, extracts forwarded (type=1) dynamics
  - `TAGs` → fetches hot + paginated tag feed
  - `Articles` → searches articles by keyword, extracts dyids from article body
  - `APIs` → fetches a remote JSON endpoint returning `{ lottery_info: LotteryInfo[] }`; `file://` prefix reads a local JSON file
  - `TxT` → reads dyids line-by-line from a local file in `lottery_dyids/`

### Network Layer (`lib/net/`)

- `bili.js` — `Line` class wraps multiple API endpoint variants with automatic failover (`switchLine()`). All Bilibili API calls go through here.
- `api.bili.js` — defines the actual URL strings and request builders for each Bilibili API.
- `http.js` — thin wrapper over Node's `https` module; respects `process.env.https_proxy`.

### Deduplication (`lib/helper/d_storage.js`)

Tracks forwarded dynamic IDs in `dyids/dyid.txt` (account 1) or `dyids/dyid{N}.txt` (account N). `config.check_if_duplicated` controls the strategy:
- `-1` — no check
- `0` / `2` — check via like-status from API
- `1` / `2` / `3` — check local dyid file

### Push Notifications (`lib/helper/notify.js`)

`sendNotify(title, content)` iterates all configured push channels (env vars: `SENDKEY`, `TG_BOT_TOKEN`/`TG_USER_ID`, `BARK_PUSH`, `DD_BOT_TOKEN`, `QYWX_AM`, `PUSH_PLUS_TOKEN`, SMTP, etc.). Each channel is independent; all configured channels fire.

### Dynamic Card Parsing (`lib/core/searcher.js:parseDynamicCard`)

Bilibili has two API response formats. The function auto-detects: if `data.card.desc.uid` exists it calls `oldParseDynamicCard` (legacy v2 format); otherwise it parses the current v3 format with `modules.*` fields. Both return the same `UsefulDynamicInfo` shape.

### Key Config Options

| Option | Effect |
|---|---|
| `LotteryOrder` | Array of source indices `[0=UIDs, 1=TAGs, 2=Articles, 3=APIs, 4=TxT]` determining processing order |
| `model` | `'00'` disables forwarding, `'10'` official only, `'01'` non-official only, `'11'` all |
| `chatmodel` | Same format, controls auto-comment |
| `check_if_duplicated` | Dedup strategy (`-1/0/1/2/3`) |
| `lottery_loop_wait` | Non-zero value causes `main.js` to sleep and re-run in a loop |
