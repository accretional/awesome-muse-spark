# awesome-muse-spark
Tools and unofficial knowledge base / resources for Meta's Muse Spark LLM

# Day 1 - Codex

First issue I ran into getting Muse Spark 1.1 working with Codex after following their guide at https://dev.meta.ai/docs/guides/coding-agents (seems non-public so not sharing its contents here): **tool-calling for anything besides the server-side tools didn't seem to work.**

TODO(Claude) add error log / contex in a collapsible ui element

Maybe I missed something, but I checked the API docs pretty thoroughly I think and verified my codex install was up to date, and didn't find anything wrong. I think it could be a genuine launch bug.

I sicced a different agent on the problem; it also thought it was just a format mismatch, and gave me this reverse proxy to run within the container I was running codex and muse to get them talking to each other. **It worked**: <img width="914" height="414" alt="image" src="https://github.com/user-attachments/assets/7c03a091-b6c2-4f6f-8ef7-ec0020657db3" />

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

Liking this model so far and will add more later - happy musing!
