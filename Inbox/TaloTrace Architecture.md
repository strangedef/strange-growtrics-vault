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