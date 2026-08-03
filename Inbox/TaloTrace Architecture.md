### Sending to device plane
opencode (LLM loop)
  │  MCP tool call over stdio: find_and_tap("Checkout")
  ▼
appium_mcp server (Python, spawned per job by opencode's MCP config)
  │  resolves label → element: pulls a11y XML tree, matches;
  │  opaque screen → vision model on screenshot → coordinates
  │  WebDriver HTTP command: POST /session/:id/element/:id/click
  ▼
Appium server (Node, per lane, own port — lane's "local Appium endpoint")
  │  translates to UiAutomator2 driver
  ▼
UiAutomator2 server (runs INSIDE Android as instrumented apk)
  │  injects real input event via Android instrumentation
  ▼
Redroid container (Android in Docker, addressed by adb serial)

Points that matter:

- Two hops of translation: semantic intent → WebDriver protocol (our MCP server), WebDriver → Android input injection (Appium/UiAutomator2). Standard Appium below layer 2; nothing custom inside the container.
- Lane routing = port + serial. Each Redroid lane on VM has own Appium server instance + own adb serial. Host-agent hands job the pair; MCP server binds session there. No cross-VM traffic — action never leaves the box.
- adb is the side-channel: app install, launch_app, screen recording, pulling artifacts go over adb directly; interaction actions go through Appium session.
- Response path: UiAutomator2 result bubbles back up same chain; then MCP server runs settle gate (adb/Appium screenshots polled until quiet) and returns settled frame as tool result to opencode.

Web analog: opencode → proxy → Playwright MCP → CDP over websocket → browser process. Same idea, CDP replaces WebDriver+adb.

### Web testing flow
 how about playwright to test web?

Same shape, browser instead of device:

opencode (LLM loop)
  │  MCP tool call over stdio: browser_click(...)
  ▼
web_proxy_mcp (Python JSON-RPC interceptor, per job)
  │  transparent pass-through — does NOT reimplement tools
  │  adds: origin scoping, action recording, screenshot injection
  ▼
@playwright/mcp (unmodified Node child process it spawns)
  │  Playwright library: resolves accessibility-snapshot ref → locator
  ▼
CDP over websocket (Chrome DevTools Protocol)
  ▼
Chromium instance (headless, same VM, launched per job)

Differences vs mobile that matter:

- Element resolution is upstream's job. Playwright MCP gives agent an accessibility snapshot (structured text tree with element refs); agent clicks by ref. No custom vision-grounding layer needed for most cases — DOM is transparent, unlike opaque WebViews on mobile.
- Proxy adds, never replaces. Sits between two stdio pipes, forwards every JSON-RPC message. On navigating actions: holds upstream's reply → injects own browser_take_screenshot calls → polls until two frames quiet (same 1% pixel threshold, tighter caps ~2.5s since web settles faster) → attaches settled frame to held reply → releases to agent. Also writes step_NNN.png artifacts + records each action to replayable log.
- Playwright auto-wait covers half the problem free — reply already arrives post-action. Pixel loop covers what auto-wait can't see: SPA route transitions, modals animating in, skeleton loaders.
- Second MCP alongside: web_meta_mcp — the non-browser tools (goals, knowledge, emit_scenarios). Browser control and harness bookkeeping = separate servers.
- No adb equivalent needed — artifacts, video, tracing all come from Playwright/CDP itself.

Why proxy-not-fork: upstream @playwright/mcp evolves fast; interception layer survives upstream upgrades untouched.

### Job
- Navigation job = run ONE scenario on a device. Payload {run_id, scenario_id, build_id}. This is the "testcase execution" — closest match to your intuition. Run of 20 scenarios → 20 navigation jobs.

Rest are not testcases:
- SCENARIO_GEN — exploration that creates testcases.
- Video analysis / triage / resolve / finalize jobs — post-run pipeline that turns recordings into findings + report.
- Benchmark scoring, PR_CHECK, HITL continuation — other machinery.

So mapping: scenario = testcase (title/goal/steps, stored in catalog). Job = "do this unit of work" — sometimes that work is "execute testcase X", often it's generation, analysis, or bookkeeping around it. One testcase can also spawn multiple jobs over its life: executed by navigation job today, re-run tomorrow, its video analyzed by another job after.