# Riddle (PWA)

Write with your pen. The page drinks your ink and answers in a flowing hand.

A progressive-web-app take on the reMarkable "riddle" diary: an HTML canvas you handwrite on (Apple Pencil pressure supported), a 2.8-second idle timer that snapshots the page, a vision call to the Anthropic API that reads your handwriting directly from the image, and a reply animated glyph-by-glyph in Dancing Script sepia ink. Your ink fades away in the order you wrote it; the diary's words fade when you start writing again.

## Deploy on GitHub Pages

1. Create a new repository (e.g. `riddle-pwa`) and upload all files in this folder to the repo root.
2. In the repo: **Settings → Pages → Source: Deploy from a branch → Branch: `main`, folder `/ (root)` → Save**.
3. Open `https://<your-username>.github.io/riddle-pwa/` (first deploy takes a minute or two).
4. On an iPad, open it in Safari → **Share → Add to Home Screen**. It launches fullscreen with no browser chrome.
5. Tap the small **wax seal** in the top-right corner and paste an Anthropic API key from [console.anthropic.com](https://console.anthropic.com/). Write something, lift your pen, and wait.

## Providers and API keys (read this)

GitHub Pages is static hosting — there is no server to hide a secret on. This app therefore uses the **bring-your-own-key** pattern: the browser calls the provider directly and the key lives only in your device's `localStorage`. The wax-seal settings let you choose:

- **Anthropic (Claude)** — default model `claude-sonnet-4-6`. Anthropic officially permits direct browser calls via the `anthropic-dangerous-direct-browser-access` header, so this works out of the box.
- **Kimi (Moonshot, global)** — `api.moonshot.ai`, default model `kimi-k2.5` (vision-capable; `kimi-k2.6` and `moonshot-v1-8k-vision-preview` also work). Keys from [platform.moonshot.ai](https://platform.moonshot.ai/).
- **Kimi (Moonshot, China)** — same API at `api.moonshot.cn`, for keys from the mainland platform; useful when the global endpoint is slow or unreachable from where you are.

The app speaks Kimi's OpenAI-compatible format automatically: `Authorization: Bearer` auth, system prompt as a `system` message, handwriting as an `image_url` data-URL part, reply read from `choices[0].message.content`, and `thinking: {type: "disabled"}` added for K2.x models so the diary answers immediately.

**Kimi CORS caveat:** Moonshot does not document browser-direct access the way Anthropic does, so the direct call may be blocked by CORS depending on their server configuration. If replies fail immediately with a network error ("the connection was refused — perhaps CORS"), route through the proxy below with the Worker's upstream URL set to `https://api.moonshot.cn/v1/chat/completions` (or `.ai`) and `Authorization: Bearer` instead of `x-api-key`.

General key hygiene, whichever provider:

- Use a **dedicated key** for this app and set a **spend limit** on it in the provider console.
- The key never appears in the repo or the URL, but anyone with physical access to your unlocked device could read it from browser storage. Treat it accordingly, and never commit a key to the repository.
- Conversation memory (including snapshots of your handwriting) is held in page memory only and sent to the provider as part of each request; "Forget everything" in the settings clears it.

### Sharing it with others? Use a proxy instead

If you want a public URL where visitors *don't* supply their own key, put a tiny proxy in front of the API (a free Cloudflare Worker works) and point the `fetch` in `index.html` at it:

```js
export default {
  async fetch(request, env) {
    if (request.method === 'OPTIONS')
      return new Response(null, { headers: cors() });
    const body = await request.text();
    const r = await fetch('https://api.anthropic.com/v1/messages', {
      method: 'POST',
      headers: {
        'content-type': 'application/json',
        'x-api-key': env.ANTHROPIC_KEY,       // set as a Worker secret
        'anthropic-version': '2023-06-01'
      },
      body
    });
    return new Response(r.body, { status: r.status, headers: cors() });
  }
};
const cors = () => ({
  'Access-Control-Allow-Origin': 'https://<your-username>.github.io',
  'Access-Control-Allow-Headers': 'content-type',
  'Access-Control-Allow-Methods': 'POST, OPTIONS',
  'content-type': 'application/json'
});
```

Add rate limiting before sharing widely, or your key will fund the internet's diary habit.

## Optional: vendor the handwriting font

The flowing hand uses **Dancing Script** (SIL Open Font License), loaded as a TTF at runtime from the jsDelivr mirror of the google/fonts repo. For fully offline, self-contained hosting, download `DancingScript[wght].ttf` from [github.com/google/fonts](https://github.com/google/fonts/tree/main/ofl/dancingscript), rename it `DancingScript-Regular.ttf`, and drop it next to `index.html` — the app tries the local file first. If no vector font loads at all, it falls back to a simpler letter-by-letter cursive reveal.

## How it works

| reMarkable riddle | this PWA |
|---|---|
| Root SSH + display driver interposition | just a web page |
| Pen events from `/dev/input` | Pointer Events (+ coalesced events, pressure) |
| Idle detection daemon | 2.8 s `setTimeout` after pen-up |
| Screen dump → LLM vision call | `canvas.toDataURL()` → `/v1/messages` with an image block |
| Dancing Script rasterized, skeletonized (Zhang-Suen), replayed as pen strokes | Dancing Script glyph outlines via opentype.js, animated with SVG `stroke-dashoffset`, then filled |
| E-ink partial refresh tricks | the browser's compositor |

## Files

- `index.html` — the whole app (canvas, ink engine, API client, handwriting writer, settings)
- `manifest.webmanifest`, `icon-192.png`, `icon-512.png` — installability
- `sw.js` — offline app shell (the API itself still needs a connection)

## Settings (the wax seal)

- **Provider** — Anthropic, Kimi global, or Kimi China; each remembers its own key and model
- **API key** — stored in `localStorage` only, per provider
- **Model** — defaults: `claude-sonnet-4-6` (Anthropic), `kimi-k2.5` (Kimi)
- **Who answers** — the persona system prompt; make it whoever you want living in your diary
- **Forget everything** — clears conversation memory and the page
