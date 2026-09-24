# Rich UI Embeds in Open WebUI — How They Work + What We're Building

## 1. What a Rich UI embed is

A Rich UI embed is an HTML document that Open WebUI renders as a **sandboxed iframe directly inside a chat message**. Unlike artifacts (which open in a side panel), embeds live inline in the conversation, stay attached to the message that produced them, and remain visible in chat history as you scroll.

Anything a normal web page can do (HTML, CSS, JS, charts, sliders, buttons, animations, CDN libraries) can go in an embed. There is no restriction on content type. The limits come from the iframe sandbox, not from what kind of UI you build.

Mental model: **an embed is the answer rendered as UI**, not a document being iterated on.

## 2. How an embed gets into the chat

### From a Tool (via Open WebUI's middleware)
Return a FastAPI `HTMLResponse` with the header `Content-Disposition: inline`. The middleware detects it and renders it.

```python
from fastapi.responses import HTMLResponse
return HTMLResponse(content=html, headers={"Content-Disposition": "inline"})
```

Return a `(HTMLResponse, context)` tuple to tell the LLM what was rendered. `context` can be a str, dict, or list. Without it, the model only gets a generic "embedded UI is active" message.

### From our Pipe (our case)
Our pipe calls the LLM provider directly and runs the tool-call loop itself, so **it bypasses the middleware**. That means the pipe must emit the embed manually:

1. Execute the tool.
2. If the result is an `HTMLResponse` with `inline` disposition (or a `(HTMLResponse, context)` tuple), decode the body.
3. Emit it:
   ```python
   await __event_emitter__({"type": "embeds", "data": {"embeds": [html]}})
   ```
4. Return a **text summary** (the context), not raw HTML, to the LLM as the tool result.

Because we own the pipe, we could also skip tool calls and parse marker blocks (e.g. `<ui>...</ui>`) out of the model's text. We're starting with tool calls because they give the model a clean decision point.

### Where it renders
- Tool-call embeds render inline at the "View Result from..." line.
- Message-level embeds render above the message text.
- ⚠️ **To verify first:** one section of the docs says `embeds`-event embeds go through the "chat-controls Embeds panel." Confirm our pipe's embeds show inline in the message, not in a side panel, before building further.

## 3. The sandbox

- **Always on:** `allow-scripts`, `allow-downloads`.
- **User toggles** (Settings → Interface, both off by default): *Allow Same Origin*, *Allow Forms*. Message-attached embeds always allow forms.
- **We keep same-origin OFF.** The model may influence the HTML, and giving model-generated JS access to the parent page is a prompt-injection risk.

What that means with same-origin off:

| Works | Doesn't work |
|---|---|
| Any in-iframe interactivity (sliders, toggles, hover, tabs) | Reading the parent's DOM, cookies, localStorage |
| Loading libraries from a CDN (e.g. Chart.js from jsDelivr) | Auto-injected Chart.js / Alpine (same-origin only) |
| Sending prompts to the chat via postMessage (with a confirm dialog) | `window.args` (same-origin only, and never set on the `embeds` path anyway) |
| | State persisting across reloads |
| | Token-by-token streaming of the UI |

**Implication:** bake all data directly into the HTML string when generating it.

## 4. Required plumbing in every embed

### Height reporting (mandatory)
With same-origin off, the parent can't measure the iframe. Without this script, the embed stays tiny with a scrollbar. Put it at the end of `<body>`:

```html
<script>
  function reportHeight() {
    parent.postMessage({ type: 'iframe:height', height: document.documentElement.scrollHeight }, '*');
  }
  window.addEventListener('load', reportHeight);
  new ResizeObserver(reportHeight).observe(document.body);
</script>
```

### Talking back to the chat
| Message type | Effect |
|---|---|
| `input:prompt` | Fill the chat input (no send) |
| `input:prompt:submit` | Fill and send. Shows a confirm dialog when same-origin is off |
| `action:submit` | Send whatever is already in the input |

```js
parent.postMessage({ type: 'input:prompt:submit', text: 'Break down by region' }, '*');
```

This is the key feature: **the UI can drive the next turn** (embed → prompt → model → new embed).

## 5. The two components we're building

Shared approach: a Python HTML shell our pipe controls (CSS variables for light/dark, base styles, the height script). Each tool only fills in its body. This keeps output consistent and keeps the model from writing boilerplate.

### 5.1 `show_chart`: inline interactive chart

```python
def show_chart(self, title: str, labels: list[str], series: dict[str, list[float]], kind: str = "line"):
    """
    Render an interactive chart inline in the chat.
    USE WHEN: the answer involves 4+ numeric data points that show a trend
    over time or a comparison across categories.
    DO NOT USE: for 1-3 numbers (just state them), for non-numeric info,
    or when the user asked for text/code only.
    After calling, write at most 2 sentences of insight. Don't restate the numbers.
    """
```

- **Renders:** title, Chart.js chart (`line` or `bar`) loaded from CDN, hover tooltips, legend when there are multiple series.
- **Data:** serialized into the HTML as JSON at generation time.
- **Returns to LLM:** e.g. `"Rendered line chart 'Q3 revenue': 12 labels, 1 series (revenue)."`
- **Validation:** every series has the same length as `labels`, `kind` is in {line, bar}. On failure, return an error string to the LLM instead of an embed.

### 5.2 `show_choices`: clickable options that drive the next turn

```python
def show_choices(self, question: str, options: list[str]):
    """
    Show 2-4 clickable options; the user's click is sent as their next message.
    USE WHEN: you need the user to pick between distinct directions before
    continuing (e.g. which breakdown, which plan).
    DO NOT USE: for yes/no questions, open-ended questions, or more than 4 options.
    """
```

- **Renders:** the question plus 2–4 buttons.
- **On click:** `postMessage({type: 'input:prompt:submit', text: option})`. The user sees a confirm dialog, which is expected with same-origin off. Disable the buttons after a click.
- **Returns to LLM:** e.g. `"Showed choices for 'Which breakdown?': [By region, By product, By month]. Waiting for user pick."`
- **Validation:** 2–4 non-empty, unique options.

## 6. How the model decides when to use them

1. **Tool descriptions** (above) carry explicit USE WHEN / DO NOT USE rules with concrete thresholds. This is the main lever.
2. **System prompt section:**
   ```
   Default to plain text. Use UI tools only when they beat text.
   At most one UI card per reply.
   Never describe what's in a card; the user can see it.
   ```
3. **Context returned to the model** so it knows what's on screen and doesn't re-render it.

## 7. Test plan

1. **Plumbing:** hardcode one `show_chart` call and confirm it renders inline in the message (the section 2 open question), auto-sizes, and works in light and dark mode.
2. **Choices loop:** click → confirm dialog → prompt sends → model responds.
3. **Decision eval:** ~20 prompts, 10 that should trigger a card and 10 that shouldn't:
   - Should trigger: "show revenue by month for 2025", "which should I tackle first?"
   - Should not trigger: "what's 12% of 80", "explain row-level security", "write a SQL query for X"

   Re-run after every prompt or description tweak. False positives (cards on trivial answers) are the usual failure; fix them by tightening the DO NOT USE lines.

## 8. Later ideas (not now)

Tables, quiz cards, parameter sliders (e.g. a filter plot with R/C sliders), and a model-writes-HTML `render_ui` tool built on the same shell.

## Reference

- Open WebUI docs, Rich UI Embedding: https://docs.openwebui.com/features/extensibility/plugin/development/rich-ui/
