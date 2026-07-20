---
name: demo-skill
description: Live browser demo of LibreChat's docx/xlsx/pptx code-interpreter generation, driven by Claude-in-Chrome for showing people how to use the feature
argument-hint: [docx|xlsx|pptx|all]
allowed-tools: [ToolSearch, mcp__claude-in-chrome__tabs_context_mcp, mcp__claude-in-chrome__tabs_create_mcp, mcp__claude-in-chrome__navigate, mcp__claude-in-chrome__computer, mcp__claude-in-chrome__find, mcp__claude-in-chrome__read_page, mcp__claude-in-chrome__gif_creator, mcp__claude-in-chrome__read_console_messages]
---

# Demo Skill (docx / xlsx / pptx)

Drives a live, watchable browser demo of LibreChat's file-generation skills (docx, xlsx, pptx) via the
`execute_code` code interpreter. Intended for showing a person how the feature works, not for CI/regression
testing — see `e2e/specs` for that instead.

## Arguments

`$ARGUMENTS` — one of `docx`, `xlsx`, `pptx`, or `all` (default: `all` if omitted). Multiple space-separated
values are also accepted, e.g. `docx xlsx`.

## Prerequisites

- LibreChat backend must already be running at `http://localhost:3080/` (`npm run backend` or
  `docker compose up -d`). If the page fails to load, stop and tell the user to start it — do not try to
  start the server yourself as part of a "demo."
- The `execute_code` sandbox (see [execute_code.md](../../../docs/local/execute_code.md)) must be up and
  healthy — it is a **separate** `docker compose` project from LibreChat's own, so `docker compose up -d` in
  the LibreChat repo does NOT start it, and it can be down even when `localhost:3080` loads fine. Before
  touching the browser, check with a plain HTTP request (e.g. `curl http://localhost:3112/v1/health`, or
  `Invoke-WebRequest` on Windows) — do not infer health from `docker ps` alone, since a partially-up stack
  (e.g. only `sandbox-runner` healthy, API gateway missing) still fails at execution time. If the health check
  fails, stop and tell the user the sandbox isn't running rather than proceeding — the demo will otherwise run
  to completion showing a false "text only, no file" failure that looks like a skill bug instead of an infra
  gap. Do not start the sandbox yourself; it's a separate repo/project the user manages.
  - The most common failure is a **half-up** stack: `sandbox-runner` healthy, but `api`/`redis`/`minio`/
    `egress_gateway`/`file_server`/`tool_call_server` all `Exited (255)` at the same timestamp — a Docker
    Desktop/WSL2 restart artifact, not an app crash (see [execute_code.md](../../../docs/local/execute_code.md#疑難排解)).
    When telling the user, give them the exact fix rather than a generic "it's down":
    `cd D:\github\code-interpreter && docker compose -f docker-compose.yaml -f docker-compose.mac.yml up -d`.
- The Claude-in-Chrome extension must be connected **for this conversation**. The pairing has been observed to
  be per-conversation, triggered by the user manually sending a `@browser:new_tab <url>` mention at least once
  — `ToolSearch`/`tabs_create_mcp` alone cannot bootstrap a connection that was never established. If
  `ToolSearch` returns no matches at all for the `mcp__claude-in-chrome__*` names (not just an empty tab list),
  stop and ask the user to run `@browser:new_tab http://localhost:3080/` once in this conversation, then retry
  — do not keep calling `ToolSearch` hoping it appears.

## Instructions

1. **Verify the `execute_code` sandbox is healthy before doing anything in the browser.** Run
   `curl -s -m 5 http://localhost:3112/v1/health` (or equivalent). If it doesn't return a healthy response,
   stop per the Prerequisites note above instead of running the browser flow — every format will otherwise
   fail at the code-execution step after several minutes of watching it retry, producing a fallback
   text-pasted script instead of a downloadable file.

2. **Load tools once, up front.** Call `ToolSearch` with
   `select:mcp__claude-in-chrome__tabs_context_mcp,mcp__claude-in-chrome__navigate,mcp__claude-in-chrome__computer,mcp__claude-in-chrome__find,mcp__claude-in-chrome__read_page,mcp__claude-in-chrome__tabs_create_mcp,mcp__claude-in-chrome__gif_creator`
   in a single call. If this returns zero matches (not just an empty tab list — actually no tool schemas at
   all), the extension has never been paired to this conversation; stop per the Prerequisites note above
   instead of retrying. If this is a fresh conversation and the user hasn't sent a `@browser` mention yet in
   it, ask them to do so once before continuing — do not assume `tabs_create_mcp` can establish that pairing.

3. **Get tab context.** Call `tabs_context_mcp`. If a LibreChat tab (`localhost:3080`) is already open and the
   user didn't ask for a fresh one, reuse it. Otherwise `tabs_create_mcp` a new tab and `navigate` to
   `http://localhost:3080/c/new`.

4. **Check for a login wall.** If the page shows a login/register form, stop and ask the user to log in (or
   supply demo credentials) rather than guessing or filling in credentials yourself.

5. **Enable Skills + Run Code for this conversation, every time.** Click the tools menu next to the paperclip
   icon in the message composer and check whether **Skills** and **執行程式碼 / Run Code** are both toggled on.
   New conversations (`/c/new`) do not inherit these from a previous chat — verify with `find`/`read_page`
   rather than assuming, and click to enable whichever is off. Without both enabled, LibreChat's `docx`/`xlsx`/
   `pptx` deployment skill (see [skills_docx.md](../../../docs/local/skills_docx.md)) never actually executes —
   the model only replies with text and no file is produced, which would make the whole demo silently wrong.

6. **Resolve which formats to demo** from `$ARGUMENTS` (default `all` → `docx`, `xlsx`, `pptx` in that order).

7. **For each format, run one self-contained round:**
   a. Start a `gif_creator` recording named `demo-<format>.gif`. Capture a couple of extra frames before/after
      real actions so playback isn't jarring.
   b. `find` the chat input, click it, and type a short natural prompt asking LibreChat to generate that file
      type via code interpreter, e.g.:
      - docx: "幫我用 code interpreter 產生一份簡單的 Word 文件（.docx），內容是一段歡迎詞。"
      - xlsx: "幫我用 code interpreter 產生一份 Excel 表格（.xlsx），內含 3 個月的範例銷售數字。"
      - pptx: "幫我用 code interpreter 產生一份 3 頁的簡報（.pptx），主題是產品介紹。"
   c. Send the message and wait for the assistant's reply to finish (poll `read_page` for the finished
      response / download attachment rather than guessing a fixed sleep).
   d. Confirm a downloadable file attachment appears in the reply. Do not click anything that isn't the
      obvious "download" affordance — no destructive or confirm-dialog-triggering elements.
   e. Stop the `gif_creator` recording for this format before moving to the next one, so each format gets its
      own clean clip.

8. **Report back**: which formats were demoed, whether each produced a downloadable file, and the saved GIF
   file name(s)/paths. If any format failed (e.g., no attachment appeared), say so plainly instead of calling
   the demo a success.

## Guardrails

- Never trigger a JS `alert`/`confirm`/`prompt` — avoid delete/destructive buttons per the browser-automation
  rules already in effect for this session.
- If a browser action fails or an element doesn't respond after 2–3 attempts, stop and report rather than
  retrying in a loop.
- This skill is for demos only. It does not assert correctness the way `e2e/specs` Playwright tests do — don't
  present a successful demo run as equivalent to passing test coverage.
