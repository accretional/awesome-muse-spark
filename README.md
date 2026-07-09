# awesome-muse-spark
Tools and unofficial knowledge base / resources for Meta's Muse Spark LLM

# Codex Tool-Calling, and "missing required field `id`"

First issue I ran into getting Muse Spark 1.1 working with Codex after following their guide at https://dev.meta.ai/docs/guides/coding-agents (seems non-public so not sharing its contents here): **tool-calling for anything besides the server-side tools didn't seem to work.**

<details>
<summary>(click to expand) Error log + root cause per Claude Fable</summary>

**What it looks like in Codex.** The *first* turn works — the model runs a web
search and even executes a shell command — but every turn *after* a search dies
identically, so the session is effectively bricked once it searches once:

```text
$ codex exec "search the web for <query>, then run: echo OK"
OpenAI Codex v0.144.0
--------
model: muse-spark-1.1
provider: meta
--------
warning: Model metadata for `muse-spark-1.1` not found. Defaulting to fallback metadata; this can degrade performance and cause issues.

codex
web search: <query>
exec
  /bin/bash -lc 'echo OK'
  succeeded in 0ms:
  OK

web search: <query>
web search: <query>
ERROR: {"error":{"code":null,"message":"`input[6]` missing required field `id`","param":"input[6]","type":"invalid_request_error"}}
```

**Root cause.** It isn't your tools config — the `web_search` tool is injected
**server-side**, and the failure is in how history is *replayed*. The Responses
API requires an `id` on a replayed `web_search_call` item, but Codex strips
server-assigned ids when it rebuilds the `input` array for the next turn. So the
second request carries an id-less `web_search_call` and gets rejected — and since
that item stays in history, every subsequent turn fails the same way.

You can see it straight against the raw API, no Codex involved:

```console
# Replay a web_search_call WITHOUT an id  ->  400
$ curl -s https://api.meta.ai/v1/responses \
    -H "Authorization: Bearer $MODEL_API_KEY" -H 'Content-Type: application/json' \
    -d '{"model":"muse-spark-1.1","input":[
          {"type":"message","role":"user","content":[{"type":"input_text","text":"hi"}]},
          {"type":"web_search_call","status":"completed","action":{"type":"search","query":"x"}},
          {"type":"message","role":"user","content":[{"type":"input_text","text":"continue"}]}
        ]}'
{"error":{"code":null,"message":"`input[1]` missing required field `id`","param":"input[1]","type":"invalid_request_error"}}

# Same request, but with ANY id on the web_search_call  ->  200 OK
$ curl -s https://api.meta.ai/v1/responses ... \
    -d '{... {"type":"web_search_call","id":"ws_anything_001","status":"completed", ...} ...}'
{"id":"resp_...","object":"response","status":"completed","output":[ ... ]}
```

The server accepts a fabricated id, which is exactly what the shim below does —
it stamps an id back onto any id-less `web_search_call` and forwards the rest
untouched. (There's no Codex-side knob for this: the tool is server-side, and
`wire_api = "chat"` was removed in recent Codex, so you can't just drop off the
Responses surface.)

</details>

Maybe I missed something, but I checked the API docs pretty thoroughly I think and verified my Codex install was up to date (0.143.0, and the bug still happened after upgrading to 0.144.0), and didn't find anything wrong. I think it could be a genuine launch bug.

## Fix - Reverse proxy shim to fix `id` replay

I sicced a different agent on the problem; it also thought it was just a format mismatch, and gave me this reverse proxy to run inside the container where I was running Codex and Muse, to get them talking to each other. **It worked**: <img width="914" height="414" alt="image" src="https://github.com/user-attachments/assets/7c03a091-b6c2-4f6f-8ef7-ec0020657db3" />

The fix:

```
cat > /usr/local/bin/meta-shim.mjs <<'MJS'
// meta-shim: local reverse proxy in front of api.meta.ai for codex.
//
// Why: codex replays conversation history as Responses-API input items, but
// strips server-assigned ids. Meta's Responses implementation requires `id`
// on replayed web_search_call items (its server-side web-search tool), so one
// search permanently wedges the session with:
//   {"message":"`input[N]` missing required field `id`", ...}
// Meta accepts a fabricated id (validated empirically), so this shim stamps
// one onto any web_search_call input item that lacks it and forwards
// everything else byte-for-byte (responses stream back untouched).
import http from 'node:http';
import https from 'node:https';

const LISTEN = process.env.META_SHIM_LISTEN || '127.0.0.1:3400';
const UPSTREAM = 'api.meta.ai';
const [host, port] = LISTEN.split(':');

let counter = 0;
function patch(body) {
  try {
    const req = JSON.parse(body);
    if (Array.isArray(req.input)) {
      let touched = false;
      for (const item of req.input) {
        if (item && typeof item === 'object' && item.type === 'web_search_call' && !item.id) {
          item.id = `ws_shim_${Date.now()}_${counter++}`;
          touched = true;
        }
      }
      if (touched) return Buffer.from(JSON.stringify(req));
    }
  } catch { /* non-JSON or unexpected shape: forward as-is */ }
  return body;
}

http.createServer((cReq, cRes) => {
  const chunks = [];
  cReq.on('data', (c) => chunks.push(c));
  cReq.on('end', () => {
    const body = patch(Buffer.concat(chunks));
    const headers = { ...cReq.headers, host: UPSTREAM, 'content-length': Buffer.byteLength(body) };
    const uReq = https.request(
      { hostname: UPSTREAM, port: 443, path: cReq.url, method: cReq.method, headers },
      (uRes) => { cRes.writeHead(uRes.statusCode, uRes.headers); uRes.pipe(cRes); },
    );
    uReq.on('error', (e) => { cRes.writeHead(502, { 'content-type': 'application/json' }); cRes.end(JSON.stringify({ error: { message: `meta-shim upstream: ${e.message}` } })); });
    uReq.end(body);
  });
}).listen(Number(port), host, () => console.log(`meta-shim: ${LISTEN} -> https://${UPSTREAM}`));
MJS
```

### Running it

Start the shim wherever Codex runs (it listens on `127.0.0.1:3400` and forwards
on to the real `api.meta.ai`):

```bash
node /usr/local/bin/meta-shim.mjs &
```

Then point Codex's provider at the shim instead of Meta directly — just change
`base_url` in `~/.codex/config.toml` (everything else from the setup guide stays
the same):

```toml
[model_providers.meta]
name = "Meta Model API"
base_url = "http://127.0.0.1:3400/v1"   # was https://api.meta.ai/v1
env_key = "MODEL_API_KEY"
wire_api = "responses"
```

Now run `codex` as usual — web search, and any tool turn after it, works. If
Codex runs in a different container from the shim, set
`META_SHIM_LISTEN=0.0.0.0:3400` (default `127.0.0.1:3400`) and use that host in
`base_url`.

Liking this model so far and will add more later - happy musing!
