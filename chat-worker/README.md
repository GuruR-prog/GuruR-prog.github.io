# chat-worker

The one non-static piece of this site — a Cloudflare Worker that answers
the "Ask about Guru" chat widget's questions using Claude, with the bio
content baked into its system prompt (see `src/index.js`). Your Anthropic
API key lives here as a Cloudflare secret, never in the static site or
this repo.

## Deploy (one-time setup)

```bash
cd chat-worker
npm install

npx wrangler login          # opens a browser to authorize with Cloudflare
npx wrangler secret put ANTHROPIC_API_KEY
# paste your key at the prompt — get one at console.anthropic.com/settings/keys
# this goes straight into Cloudflare's encrypted secret store, never into a file

npm run deploy
```

That last command prints your Worker's URL, something like:

```
https://guru-site-chat.<your-subdomain>.workers.dev
```

Copy it — you'll need to paste it into `CHAT_API_URL` near the top of
`../index.html`'s `<script>` block (search for `CHAT_API_URL`). Without
that, the chat widget has nothing to call.

## Local development

```bash
npm run dev
```

Runs the Worker locally (Wrangler prints a `localhost` URL). Test it
directly without a browser:

```bash
curl -X POST http://localhost:8787 \
  -H "Content-Type: application/json" \
  -d '{"question":"What does Guru do at Coupang?"}'
```

## What it does and doesn't do

- Answers are scoped to the bio content in `SYSTEM_PROMPT` — it's
  instructed to say "I don't have that information" rather than invent
  anything, and to decline anything unrelated to Guru's background.
- Caps question length (500 chars) and conversation history (last 6
  turns) to keep costs bounded — this is a personal site's chat widget,
  not a general-purpose assistant.
- CORS is locked to `https://gurur-prog.github.io` — the widget only
  works from the real site, not if someone copies the fetch call
  elsewhere in a browser. That said, CORS doesn't stop direct
  server-to-server requests (`curl`, scripts) — for real abuse
  protection, turn on **Cloudflare's Rate Limiting rules** for this
  Worker in the Cloudflare dashboard (Workers & Pages → your Worker →
  Settings → Triggers → Rate limiting). The free tier includes basic
  rate limiting; this repo doesn't attempt to reimplement that in code.

## Redeploying after a content change

Whenever the resume content on the site changes, update the matching
text in `SYSTEM_PROMPT` (`src/index.js`) and run `npm run deploy` again
— the two aren't automatically kept in sync.
