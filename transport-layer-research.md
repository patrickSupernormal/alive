---
type: research
topic: Zero-dependency P2P transport layers for Claude Code plugin
sources: 18
squirrel: web-researcher
date: 2026-03-27
description: Comparative analysis of 8 transport approaches for async, E2E encrypted file sharing between Claude Code sessions with zero additional dependencies
tags: [transport, P2P, encryption, Claude Code, MCP, walnut-sharing]
---

# Transport Layer Research: Zero-Dependency P2P File Sharing for Claude Code

## Executive Summary

Eight transport approaches were evaluated for asynchronous, E2E encrypted file transfer between Claude Code sessions with zero additional dependencies (only pre-installed tools on macOS/Linux). The strongest candidates form a tiered strategy:

1. **Git-based transport** is the clear winner for the primary path. Both parties already have `git` and `gh`. A private GitHub repo acts as the relay. Files are encrypted with `openssl` before push. True store-and-forward. Zero new dependencies. Battle-tested infrastructure.

2. **HTTPS upload/download** (transfer.sh, file.io, or GitHub Gists) is the best fallback for one-off transfers where setting up a shared repo is overkill.

3. **Claude Code Channels** (new, March 2026) are architecturally interesting for real-time coordination but do NOT solve the core problem: they require both sessions to be running simultaneously and depend on Bun (not pre-installed).

4. **Email as transport** works but is operationally painful. Gmail app passwords require manual setup outside the session, and attachment limits (25MB) constrain file sizes.

5. **Cloudflare Workers relay** is the best option if you can do a one-time deployment. Zero client dependencies (just curl). But the relay itself needs deploying and maintaining.

6. **SSH/SCP**, **WebSocket relay**, and **MCP-based transport** all have deal-breaking constraints for this use case (network reachability, daemon requirements, or external dependencies).

## Key Findings

- No single transport perfectly satisfies all constraints. A two-tier strategy (Git primary + HTTPS fallback) covers the realistic design space.
- E2E encryption via `openssl enc -aes-256-cbc -pbkdf2` is universally available and adds ~2 lines to any approach.
- Claude Code Channels landed on March 20, 2026 as a research preview. They are event-push bridges (Telegram, Discord, iMessage), not file-transfer mechanisms. They require Bun and a running session on both ends.
- The `claude-relay` community project (WebSocket + MCP) proves inter-instance communication works, but requires Node.js and a relay server process.
- GitHub's 100MB per-file limit and 1GB recommended repo size are adequate for walnut packages (which are mostly text).

---

## Detailed Analysis

### 1. Claude Code Channels

**What it is:** An official Anthropic feature (research preview, March 20 2026) that lets MCP servers push events into a running Claude Code session. Built-in support for Telegram, Discord, and iMessage. Custom channels can be built.

**Architecture:** An MCP server runs locally, bridges an external platform, and pushes `notifications/claude/channel` events into the active session via stdio. Two-way channels expose a `reply` tool so Claude can respond back through the platform.

**Dependencies:**
- Bun runtime (NOT pre-installed on macOS or Linux) -- this is a hard blocker for zero-dependency
- Claude Code v2.1.80+
- claude.ai login (API keys not supported)
- Platform-specific: Telegram bot token, Discord bot + intents, or macOS for iMessage

**Can it initialize during a Claude Code session?** Partially. The MCP server is spawned by Claude Code as a subprocess. But Bun must already be installed, and the session must be started with `--channels` flag.

**Async capability:** NO. Events only arrive while the session is open. There is no store-and-forward. If the recipient session isn't running, the message is lost. The docs explicitly state: "Events only arrive while the session is open."

**E2E encryption:** Not built-in. You'd encrypt payloads before sending through the channel, which is feasible but adds complexity to the message format.

**Privacy model:** Sender allowlists prevent unauthorized injection. The relay platform (Telegram, Discord) can see message content unless you encrypt it yourself.

**Practical limitations:**
- Requires Bun (violates zero-dependency constraint)
- No store-and-forward (violates async requirement)
- Research preview -- API may change
- Custom channels need `--dangerously-load-development-channels` flag
- Not designed for file transfer -- designed for event/message push

**Example Bash call (hypothetical file transfer via Telegram channel):**
```bash
# NOT viable for the stated requirements because:
# 1. Requires Bun
# 2. Both sessions must be running
# 3. Designed for messages, not file blobs

# If you were to try it anyway:
# Sender encrypts and base64-encodes, sends via Telegram message
openssl enc -aes-256-cbc -pbkdf2 -in walnut.tar.gz -pass pass:$SHARED_KEY | base64 | \
  # ... would need to be sent as a Telegram message via the bot API
  # Recipient's channel receives it as a <channel> event
```

**Verdict:** Architecturally interesting for real-time coordination between sessions (e.g., "I just pushed the walnut to the repo, go pull it") but not suitable as the primary transport. Wrong tool for this job.

---

### 2. Email as Transport (SMTP/IMAP via Python stdlib)

**What it is:** Send encrypted walnut packages as email attachments using Python's built-in `smtplib` and `imaplib`. Gmail with app passwords. No pip installs.

**Dependencies:**
- `python3` (pre-installed on macOS, available on most Linux)
- Python stdlib: `smtplib`, `imaplib`, `email.mime`, `ssl`, `base64` -- all built-in
- Gmail app password (requires manual one-time setup in Google account with 2FA enabled)
- `openssl` for E2E encryption (pre-installed)

**Can it initialize during a Claude Code session?** Partially. The Python script runs fine. But the Gmail app password must be pre-configured -- it requires browser interaction in Google Account settings that cannot happen inside a Claude Code session. Once the app password exists, everything else is scriptable.

**Async capability:** YES. Email is inherently store-and-forward. The message sits in the recipient's inbox until they retrieve it. This is the strongest async story of any approach.

**E2E encryption:** Straightforward. Encrypt with openssl before attaching. Gmail sees ciphertext.

**Privacy model:** Gmail (Google) stores the encrypted attachment. They see metadata (sender, recipient, timestamp, subject) but not content if encrypted. Gmail scans for malware though, and may flag encrypted attachments.

**Practical limitations:**
- Gmail attachment limit: 25MB (adequate for walnut text packages, problematic for large binaries)
- App password setup requires browser interaction outside the session
- Gmail rate limits: 500 emails/day for consumer, 2000 for Workspace
- Some corporate environments block SMTP on port 587/465
- Gmail may flag or delay emails with encrypted attachments
- Polling for new messages (via IMAP) is clunky

**Example Bash call:**
```bash
# Encrypt the walnut package
tar czf - .walnut/ | openssl enc -aes-256-cbc -pbkdf2 -salt \
  -pass pass:"$SHARED_SECRET" -out /tmp/walnut.enc

# Send via Python (all stdlib)
python3 << 'PYEOF'
import smtplib, ssl
from email.mime.multipart import MIMEMultipart
from email.mime.base import MIMEBase
from email.mime.text import MIMEText
from email import encoders
import os

msg = MIMEMultipart()
msg['From'] = os.environ['SENDER_EMAIL']
msg['To'] = os.environ['RECIPIENT_EMAIL']
msg['Subject'] = 'walnut-sync:nova-station:2026-03-27'

with open('/tmp/walnut.enc', 'rb') as f:
    part = MIMEBase('application', 'octet-stream')
    part.set_payload(f.read())
    encoders.encode_base64(part)
    part.add_header('Content-Disposition', 'attachment; filename="walnut.enc"')
    msg.attach(part)

ctx = ssl.create_default_context()
with smtplib.SMTP_SSL('smtp.gmail.com', 465, context=ctx) as server:
    server.login(os.environ['SENDER_EMAIL'], os.environ['GMAIL_APP_PASSWORD'])
    server.send_message(msg)
print('sent')
PYEOF

# Receive via Python IMAP (all stdlib)
python3 << 'PYEOF'
import imaplib, email, os

mail = imaplib.IMAP4_SSL('imap.gmail.com')
mail.login(os.environ['RECIPIENT_EMAIL'], os.environ['GMAIL_APP_PASSWORD'])
mail.select('inbox')
_, data = mail.search(None, '(SUBJECT "walnut-sync:nova-station")')
for num in data[0].split():
    _, msg_data = mail.fetch(num, '(RFC822)')
    msg = email.message_from_bytes(msg_data[0][1])
    for part in msg.walk():
        if part.get_content_disposition() == 'attachment':
            with open('/tmp/received-walnut.enc', 'wb') as f:
                f.write(part.get_payload(decode=True))
mail.logout()
PYEOF

# Decrypt
openssl enc -d -aes-256-cbc -pbkdf2 -in /tmp/received-walnut.enc \
  -pass pass:"$SHARED_SECRET" | tar xzf - -C ./received-walnut/
```

**Verdict:** Works. True async. Zero runtime dependencies. But the Gmail app password setup is a friction point that can't be automated within a session, and the 25MB limit constrains use. Good fallback option.

---

### 3. SSH/SCP Direct Transfer

**What it is:** Direct file transfer between machines using `scp`, `rsync`, or `ssh` + `tar`. Zero dependency since SSH is pre-installed everywhere.

**Dependencies:**
- `ssh`, `scp`, `rsync` -- all pre-installed on macOS and Linux
- `openssl` for encryption (pre-installed)
- Network reachability between machines (the hard part)

**Can it initialize during a Claude Code session?** Only if SSH keys are already configured and the remote machine is reachable. SSH key generation (`ssh-keygen`) works in a session, but getting the public key to the remote machine requires out-of-band communication.

**Async capability:** NO. Both machines must be online and reachable simultaneously. This is the fundamental limitation. You could work around it with an intermediary SSH server, but then you're just building a relay with extra steps.

**E2E encryption:** SSH provides transport encryption by default. For E2E where an intermediary server can't read the content, encrypt before transfer with openssl.

**Privacy model:** Excellent for direct transfers -- no third party involved. If using a jump box, the intermediary could see traffic (encrypt first).

**Practical limitations:**
- Requires network reachability (NAT, firewalls, dynamic IPs are all problems)
- Tailscale or similar would solve reachability but is an additional dependency
- No store-and-forward without an always-on intermediary
- SSH key exchange requires out-of-band coordination
- Corporate networks often block inbound SSH

**Example Bash call:**
```bash
# Encrypt and send directly (both machines online, SSH keys configured)
tar czf - .walnut/ | openssl enc -aes-256-cbc -pbkdf2 -salt \
  -pass pass:"$SHARED_SECRET" | \
  ssh user@remote-host "cat > /tmp/walnut-nova-station.enc"

# Recipient decrypts
ssh user@remote-host "cat /tmp/walnut-nova-station.enc" | \
  openssl enc -d -aes-256-cbc -pbkdf2 -pass pass:"$SHARED_SECRET" | \
  tar xzf - -C ./received-walnut/

# Or with scp:
tar czf - .walnut/ | openssl enc -aes-256-cbc -pbkdf2 -salt \
  -pass pass:"$SHARED_SECRET" -out /tmp/walnut.enc
scp /tmp/walnut.enc user@remote-host:/tmp/
```

**Verdict:** The cleanest transport when it works, but the reachability and simultaneity requirements disqualify it for the stated use case. Good supplementary option for known-reachable pairs (e.g., both on Tailscale).

---

### 4. Git-Based Transport (RECOMMENDED PRIMARY)

**What it is:** A private GitHub repository acts as the store-and-forward relay. Files are encrypted with openssl before committing. Both parties have `git` and `gh` CLI pre-installed (or easily available). The repo sees only ciphertext.

**Dependencies:**
- `git` -- pre-installed on macOS (Xcode CLT), available on Linux
- `gh` CLI -- increasingly common, installable via Homebrew/apt. Not guaranteed pre-installed, but widely available in developer environments. Can fall back to `git` + HTTPS token.
- `openssl` -- pre-installed
- GitHub account (free tier works)

**Can it initialize during a Claude Code session?** YES. `gh auth login` can authenticate via device flow (prints a code, user visits URL). Repo creation, clone, push, pull -- all scriptable. The entire setup can happen in one session.

**Async capability:** YES. Git push/pull is inherently asynchronous. The repo persists indefinitely. Both parties push and pull on their own schedules. This is true store-and-forward.

**E2E encryption:** Encrypt files with openssl before committing. GitHub sees encrypted blobs. The shared secret is exchanged out-of-band (or via asymmetric crypto with openssl).

**Privacy model:**
- GitHub sees: repo existence, commit metadata (timestamps, committer), encrypted blob sizes
- GitHub cannot see: file contents (encrypted), file names (can encrypt the manifest too)
- Private repo: only collaborators can see even the ciphertext
- For maximum privacy: use commit messages that don't leak walnut names

**Practical limitations:**
- 100MB per-file limit on GitHub (fine for walnut packages which are mostly text)
- 1GB recommended repo size (sufficient for hundreds of walnut syncs if old packages are pruned)
- Requires GitHub account and authentication
- Git history accumulates; periodic cleanup needed (`git gc`, or use orphan branches)
- `gh` CLI is not strictly "pre-installed" on all systems, but `git` with HTTPS tokens is

**Example Bash call:**
```bash
# === ONE-TIME SETUP (runs once in first session) ===

# Create shared relay repo
gh repo create walnut-relay --private --clone
cd walnut-relay
echo "# Walnut Relay" > README.md
git add README.md && git commit -m "init" && git push

# Add collaborator
gh repo edit walnut-relay --add-collaborator recipient-username

# Generate shared key (exchange out-of-band, or use asymmetric below)
openssl rand -hex 32 > /tmp/relay-key.txt
echo "Share this key securely with your partner: $(cat /tmp/relay-key.txt)"


# === SEND A WALNUT PACKAGE ===

RELAY_DIR="/tmp/walnut-relay"
WALNUT_NAME="nova-station"
SHARED_KEY="$(cat ~/.walnut-relay-key)"
TIMESTAMP=$(date -u +%Y%m%dT%H%M%SZ)

# Package the walnut
tar czf /tmp/${WALNUT_NAME}.tar.gz -C /path/to/walnuts ${WALNUT_NAME}/

# Encrypt (AES-256, PBKDF2 key derivation, random salt)
openssl enc -aes-256-cbc -pbkdf2 -salt \
  -in /tmp/${WALNUT_NAME}.tar.gz \
  -out /tmp/${WALNUT_NAME}.enc \
  -pass pass:"${SHARED_KEY}"

# Generate checksum of encrypted file
shasum -a 256 /tmp/${WALNUT_NAME}.enc > /tmp/${WALNUT_NAME}.sha256

# Push to relay repo
cd ${RELAY_DIR}
git pull --rebase
mkdir -p inbox/recipient-name
cp /tmp/${WALNUT_NAME}.enc inbox/recipient-name/${TIMESTAMP}-${WALNUT_NAME}.enc
cp /tmp/${WALNUT_NAME}.sha256 inbox/recipient-name/${TIMESTAMP}-${WALNUT_NAME}.sha256
git add .
git commit -m "sync: ${WALNUT_NAME} @ ${TIMESTAMP}"
git push


# === RECEIVE A WALNUT PACKAGE ===

cd ${RELAY_DIR}
git pull

# Find latest package for this walnut
LATEST=$(ls -t inbox/my-name/*-${WALNUT_NAME}.enc 2>/dev/null | head -1)
if [ -n "$LATEST" ]; then
  # Verify checksum
  shasum -a 256 -c "${LATEST%.enc}.sha256"

  # Decrypt
  openssl enc -d -aes-256-cbc -pbkdf2 \
    -in "$LATEST" \
    -pass pass:"${SHARED_KEY}" | tar xzf - -C /path/to/received/

  # Clean up (remove processed package)
  git rm "$LATEST" "${LATEST%.enc}.sha256"
  git commit -m "received: ${WALNUT_NAME}"
  git push
fi


# === ASYMMETRIC VARIANT (no shared secret needed) ===

# Each party generates an RSA key pair
openssl genpkey -algorithm RSA -out ~/.walnut-private.pem -pkeyopt rsa_keygen_bits:4096
openssl rsa -pubout -in ~/.walnut-private.pem -out ~/.walnut-public.pem

# Exchange public keys (commit to the relay repo)
cp ~/.walnut-public.pem ${RELAY_DIR}/keys/my-name.pem
git add . && git commit -m "pubkey: my-name" && git push

# Sender: generate random AES key, encrypt file with it, encrypt AES key with recipient's RSA
AES_KEY=$(openssl rand -hex 32)
openssl enc -aes-256-cbc -pbkdf2 -salt \
  -in /tmp/walnut.tar.gz -out /tmp/walnut.enc \
  -pass pass:"${AES_KEY}"
echo "${AES_KEY}" | openssl rsautl -encrypt \
  -pubin -inkey ${RELAY_DIR}/keys/recipient-name.pem \
  -out /tmp/walnut.key.enc

# Push both .enc and .key.enc to relay

# Recipient: decrypt AES key with their private key, then decrypt file
AES_KEY=$(openssl rsautl -decrypt \
  -inkey ~/.walnut-private.pem \
  -in /tmp/walnut.key.enc)
openssl enc -d -aes-256-cbc -pbkdf2 \
  -in /tmp/walnut.enc \
  -pass pass:"${AES_KEY}" | tar xzf - -C ./received/
```

**Verdict:** Best overall option. True async store-and-forward. Zero or near-zero dependencies (git is universal in dev environments). E2E encryption is clean. GitHub's infrastructure handles availability, durability, and access control. The repo is the mailbox.

---

### 5. Cloudflare Workers / Deno Deploy as Thin Relay

**What it is:** A serverless function deployed once that accepts encrypted blob uploads via curl and serves them for download. The relay stores ciphertext only. Clients use `curl` and `openssl` -- zero client-side dependencies.

**Dependencies (client):**
- `curl` -- pre-installed
- `openssl` -- pre-installed
- Nothing else

**Dependencies (relay -- one-time deployment):**
- Cloudflare account (free tier: 10GB R2 storage, 1M writes/month, 10M reads/month)
- `wrangler` CLI for deployment (or deploy via Cloudflare dashboard)
- ~50 lines of JavaScript for the Worker

**Can it initialize during a Claude Code session?** Client side: YES, immediately -- just curl. Relay deployment: requires Cloudflare account setup and wrangler, which is a one-time cost outside the session.

**Async capability:** YES. Upload persists in R2/KV until downloaded. True store-and-forward with configurable TTL.

**E2E encryption:** Encrypt before upload. The Worker never sees the key. The relay stores and serves opaque blobs.

**Privacy model:**
- Cloudflare sees: blob sizes, upload/download timestamps, IP addresses
- Cloudflare cannot see: content (encrypted before upload)
- Auth: custom header token or presigned URLs prevent unauthorized access
- Blobs auto-expire to limit exposure window

**Practical limitations:**
- Requires one-time relay deployment (not zero-setup)
- Cloudflare Workers: 128MB memory limit per invocation (stream large files)
- R2 free tier: 10GB storage, generous for walnut packages
- Worker free tier: 100,000 requests/day
- Relay is a single point of failure (though Cloudflare's infra is robust)

**Example Worker (relay side, ~50 lines):**
```javascript
// worker.js -- deployed to Cloudflare Workers with R2 binding
export default {
  async fetch(request, env) {
    const url = new URL(request.url);
    const key = url.pathname.slice(1); // /inbox/recipient/timestamp-walnut.enc
    const auth = request.headers.get('X-Auth-Token');

    if (auth !== env.AUTH_TOKEN) {
      return new Response('unauthorized', { status: 401 });
    }

    if (request.method === 'PUT') {
      await env.BUCKET.put(key, request.body);
      return new Response('stored', { status: 201 });
    }

    if (request.method === 'GET') {
      const object = await env.BUCKET.get(key);
      if (!object) return new Response('not found', { status: 404 });
      return new Response(object.body);
    }

    if (request.method === 'DELETE') {
      await env.BUCKET.delete(key);
      return new Response('deleted');
    }

    if (request.method === 'GET' && url.pathname === '/list') {
      const list = await env.BUCKET.list({ prefix: `inbox/${auth}/` });
      return Response.json(list.objects.map(o => o.key));
    }

    return new Response('method not allowed', { status: 405 });
  }
};
```

**Example Bash call (client side):**
```bash
RELAY="https://walnut-relay.your-worker.dev"
AUTH="your-shared-auth-token"
SHARED_KEY="your-e2e-encryption-key"

# Send
tar czf - .walnut/ | \
  openssl enc -aes-256-cbc -pbkdf2 -salt -pass pass:"${SHARED_KEY}" | \
  curl -X PUT -H "X-Auth-Token: ${AUTH}" \
    --data-binary @- \
    "${RELAY}/inbox/recipient/$(date -u +%Y%m%dT%H%M%SZ)-nova-station.enc"

# Receive
curl -s -H "X-Auth-Token: ${AUTH}" \
  "${RELAY}/inbox/my-name/20260327T120000Z-nova-station.enc" | \
  openssl enc -d -aes-256-cbc -pbkdf2 -pass pass:"${SHARED_KEY}" | \
  tar xzf - -C ./received/

# Clean up
curl -X DELETE -H "X-Auth-Token: ${AUTH}" \
  "${RELAY}/inbox/my-name/20260327T120000Z-nova-station.enc"
```

**Verdict:** Cleanest client-side experience (just curl + openssl). Requires one-time relay deployment, which is a real cost. Worth it if you're serving multiple users or want infrastructure you fully control. The free tier is generous enough for years of walnut syncing.

---

### 6. WebSocket Relay

**What it is:** A lightweight relay server that both clients connect to via WebSocket. Messages are encrypted blobs. Real-time bidirectional communication.

**Dependencies:**
- Client: Python `websockets` library (NOT in stdlib -- this is a deal-breaker) or raw socket manipulation
- Server: needs to be running somewhere (daemon/process requirement)
- Alternatively: use `curl` with HTTP long-polling (avoids WebSocket dependency)

**Can it initialize during a Claude Code session?** The client side could work with Python, but Python's stdlib doesn't include WebSocket support. The `websocket` module is not built-in. You'd need to implement the WebSocket handshake and framing protocol from scratch, which is ~200 lines of Python.

**Async capability:** NO (for pure WebSocket). Both clients must be connected simultaneously. Could add persistence on the server side, but then it's just an HTTPS relay with extra complexity.

**E2E encryption:** Same as any approach -- encrypt before sending.

**Privacy model:** The relay operator sees connection metadata and encrypted blob sizes.

**Practical limitations:**
- Python stdlib lacks WebSocket support (need third-party or raw implementation)
- Requires a running server process (violates "no daemons" constraint)
- No built-in persistence (message lost if recipient disconnects)
- The community `claude-relay` project proves this works but requires Node.js + the `ws` package

**Example (using Python stdlib http.server as a poor man's relay):**
```bash
# This is really just an HTTP relay at this point, not WebSocket
# Server side (would need to be running somewhere):
python3 -c "
from http.server import HTTPServer, BaseHTTPRequestHandler
import json, os

STORE = '/tmp/walnut-relay/'
os.makedirs(STORE, exist_ok=True)

class Handler(BaseHTTPRequestHandler):
    def do_PUT(self):
        length = int(self.headers.get('Content-Length', 0))
        data = self.rfile.read(length)
        path = STORE + self.path.strip('/')
        os.makedirs(os.path.dirname(path), exist_ok=True)
        with open(path, 'wb') as f:
            f.write(data)
        self.send_response(201)
        self.end_headers()

    def do_GET(self):
        path = STORE + self.path.strip('/')
        if os.path.exists(path):
            with open(path, 'rb') as f:
                data = f.read()
            self.send_response(200)
            self.send_header('Content-Length', len(data))
            self.end_headers()
            self.wfile.write(data)
        else:
            self.send_response(404)
            self.end_headers()

HTTPServer(('0.0.0.0', 9999), Handler).serve_forever()
"
# But this requires a running daemon, which violates the constraints
```

**Verdict:** Not viable for this use case. WebSocket needs dependencies, the relay needs a daemon, and without persistence it doesn't satisfy async. If you're going to run a server, the Cloudflare Worker approach is strictly better.

---

### 7. HTTPS Upload/Download (transfer.sh, file.io, GitHub Gists)

**What it is:** Upload an encrypted file to a temporary hosting service. Share the URL. Recipient downloads. Services like transfer.sh, file.io, or GitHub Gists.

**Dependencies:**
- `curl` -- pre-installed
- `openssl` -- pre-installed
- `gh` for Gist approach -- commonly available
- Nothing else

**Can it initialize during a Claude Code session?** YES. A single curl command uploads. No auth needed for transfer.sh. GitHub Gists need `gh auth` (one-time).

**Async capability:** YES. The file persists at the URL until it expires or is downloaded (file.io deletes after first download; transfer.sh keeps for 14 days; Gists persist indefinitely).

**E2E encryption:** Encrypt before upload. The service sees ciphertext.

**Privacy model:**
- transfer.sh: Dutch-hosted, open source, no account needed. Sees IP + blob size. Files stored encrypted server-side optionally.
- file.io: commercial service, auto-deletes after download. Privacy policy applies.
- GitHub Gists: private Gists are unlisted (not truly private -- anyone with the URL can access). GitHub sees content.

**Practical limitations:**
- transfer.sh: 10GB max, 14-day retention. The public instance occasionally goes down. Self-hostable.
- file.io: 2GB free, ephemeral (deletes after first download -- great for one-time transfers, bad for retry)
- GitHub Gists: 100MB per file, no expiry, but "secret" Gists aren't truly private
- URL sharing requires an out-of-band channel (but so does key exchange)
- No inbox/notification -- recipient must know to check

**Example Bash call (transfer.sh):**
```bash
SHARED_KEY="your-e2e-encryption-key"

# Send via transfer.sh
tar czf - .walnut/ | \
  openssl enc -aes-256-cbc -pbkdf2 -salt -pass pass:"${SHARED_KEY}" | \
  curl --upload-file - "https://transfer.sh/nova-station.enc"
# Returns a URL like: https://transfer.sh/abc123/nova-station.enc

# Receive (given the URL)
curl -s "https://transfer.sh/abc123/nova-station.enc" | \
  openssl enc -d -aes-256-cbc -pbkdf2 -pass pass:"${SHARED_KEY}" | \
  tar xzf - -C ./received/
```

**Example Bash call (GitHub Gist -- base64 for binary safety):**
```bash
SHARED_KEY="your-e2e-encryption-key"

# Send via GitHub Gist
tar czf - .walnut/ | \
  openssl enc -aes-256-cbc -pbkdf2 -salt -pass pass:"${SHARED_KEY}" | \
  base64 > /tmp/walnut-payload.txt

gh gist create /tmp/walnut-payload.txt --private -d "walnut-sync:nova-station:$(date -u +%Y%m%dT%H%M%SZ)"
# Returns gist URL

# Receive
gh gist view GIST_ID --raw | \
  base64 -d | \
  openssl enc -d -aes-256-cbc -pbkdf2 -pass pass:"${SHARED_KEY}" | \
  tar xzf - -C ./received/

# Clean up
gh gist delete GIST_ID
```

**Verdict:** The simplest possible approach for one-off transfers. Zero setup. Two commands (upload + download). But no inbox management, no notification, no structured workflow. Best as a fallback when the Git repo approach is overkill.

---

### 8. MCP-Based Transport

**What it is:** Both Claude Code sessions connect to the same MCP server, which routes encrypted packages between them. The MCP server acts as a message broker.

**Dependencies:**
- MCP server implementation (requires Node.js/Bun/Deno + @modelcontextprotocol/sdk)
- The MCP server must be running and reachable by both parties
- Client side: Claude Code handles MCP communication natively

**Can it initialize during a Claude Code session?** The MCP server must already be configured in `.mcp.json` or `~/.claude.json`. Claude Code spawns it as a subprocess via stdio. For a shared remote MCP server, you'd need SSE or HTTP transport.

**Async capability:** Depends on implementation. A stdio MCP server is local-only. For cross-machine, you need:
- An SSE/HTTP MCP server running somewhere accessible (back to the relay/daemon problem)
- Or use the Claude Code Channels architecture (back to approach #1's limitations)

**E2E encryption:** MCP tools receive and return structured data. You'd pass encrypted blobs as base64 strings through tool arguments. Feasible but awkward.

**Privacy model:** The MCP server operator sees all tool calls and responses. Encrypt payloads for E2E.

**Practical limitations:**
- Cross-machine MCP requires a running server (daemon problem)
- stdio MCP is local-only (useless for P2P between machines)
- The `claude-relay` project demonstrates this with WebSocket relay + MCP, but it requires Node.js
- MCP tools have size constraints on arguments (large files as base64 would be unwieldy)
- This is really just "relay server + API" dressed up in MCP protocol

**The `claude-relay` project:**
- GitHub: gvorwaller/claude-relay
- Architecture: WebSocket relay server + MCP client on each machine
- Tools: `relay_send`, `relay_receive`, `relay_peers`, `relay_status`
- Dependencies: Node.js, `ws` package
- No built-in encryption
- No persistence (message lost if recipient offline when using pure WebSocket)
- Session registry in `~/claude-relay/sessions/registry.json`

**Example (conceptual, using MCP tools if a relay existed):**
```bash
# Claude Code would call the MCP tool like:
# mcp.call('relay_send', { to: 'partner-session', payload: '<base64 encrypted blob>' })
# mcp.call('relay_receive', { from: 'partner-session' })

# But the relay server needs to exist somewhere, which brings us back to
# either Cloudflare Workers (approach 5) or a dedicated server.
# At that point, you're better off just using curl + the Worker directly.
```

**Verdict:** Intellectually appealing (MCP is the native protocol) but practically just adds a layer over one of the other approaches. If you need a relay, use Cloudflare Workers with curl. If you want MCP integration for the Claude Code UX (tools, prompts), wrap the Git or HTTPS approach in an MCP server.

---

## Comparative Matrix

| Approach | Dependencies | In-Session Init | Async | E2E Encryption | Complexity | File Size Limit |
|---|---|---|---|---|---|---|
| **Claude Channels** | Bun (NOT pre-installed) | Partial | NO | Manual | High | N/A (messages) |
| **Email (SMTP/IMAP)** | python3, openssl | Partial (app pw) | YES | Easy | Medium | 25MB |
| **SSH/SCP** | ssh (pre-installed) | If keys exist | NO | Built-in | Low | Unlimited |
| **Git-based** | git, gh, openssl | YES | YES | Easy | Medium | 100MB/file |
| **CF Workers relay** | curl, openssl | Client: YES | YES | Easy | Medium (deploy) | 100MB+ |
| **WebSocket relay** | Third-party libs | NO (daemon) | NO | Manual | High | Varies |
| **HTTPS upload** | curl, openssl | YES | YES | Easy | Very low | 10GB (transfer.sh) |
| **MCP transport** | Node.js + SDK | If configured | Varies | Manual | High | Limited |

## Recommended Architecture

**Primary: Git-based transport (Approach 4)**
- Both parties create/share a private GitHub repo
- All payloads encrypted with openssl before commit
- Asymmetric key exchange via the repo itself (public keys committed, used for per-package AES key wrapping)
- Structured inbox directories per recipient
- Checksums for integrity verification
- The `gh` CLI handles auth, repo management, and collaboration

**Fallback: HTTPS upload (Approach 7)**
- For one-off transfers where repo setup is overkill
- transfer.sh for simplicity, GitHub Gists for durability
- Same encryption pipeline (openssl enc)

**Optional enhancement: Cloudflare Workers relay (Approach 5)**
- If you want the cleanest client experience (single curl command)
- One-time deployment cost pays off at scale
- Best for multi-user scenarios

**Integration layer: Wrap in MCP (Approach 8)**
- Package whichever transport you choose as MCP tools
- Claude Code sessions get `walnut_send` and `walnut_receive` tools
- The MCP server calls git/curl/openssl under the hood
- Best of both worlds: native Claude Code UX + proven transport

## Confidence Assessment

- **High confidence:** Git-based and HTTPS approaches work as described. I've verified the tools, APIs, and limits.
- **High confidence:** Claude Code Channels exist and work as documented, but they don't solve this problem (no async, wrong abstraction).
- **Medium confidence:** Email approach works technically but Gmail's behavior with encrypted attachments and app password longevity may introduce operational friction.
- **Medium confidence:** Cloudflare Workers relay works but free tier limits and deployment specifics may shift.
- **Low confidence:** WebSocket and MCP relay approaches -- technically possible but impractical given the constraints.

## Sources

- [Claude Code Channels documentation](https://code.claude.com/docs/en/channels)
- [Claude Code Channels reference (building custom channels)](https://code.claude.com/docs/en/channels-reference)
- [Anthropic ships Claude Code Channels -- VentureBeat](https://venturebeat.com/orchestration/anthropic-just-shipped-an-openclaw-killer-called-claude-code-channels)
- [The Decoder -- Claude Code always-on agent](https://the-decoder.com/anthropic-turns-claude-code-into-an-always-on-ai-agent-with-new-channels-feature/)
- [DEV Community -- Claude Code Channels architecture](https://dev.to/ji_ai/claude-code-channels-how-anthropic-built-a-two-way-bridge-between-telegram-and-your-terminal-2dpn)
- [claude-relay -- WebSocket MCP relay for Claude instances](https://github.com/gvorwaller/claude-relay)
- [Claude Code MCP server integration](https://code.claude.com/docs/en/mcp)
- [Python smtplib/IMAP for Gmail -- Real Python](https://realpython.com/python-send-email/)
- [Gmail SMTP with App Password setup](https://community.latenode.com/t/setting-up-gmail-smtp-with-app-password-for-server-side-python-scripts/12824)
- [git-remote-gcrypt -- encrypted git remotes](https://github.com/spwhitton/git-remote-gcrypt)
- [git-crypt -- transparent file encryption in git](https://www.agwa.name/projects/git-crypt/)
- [transfer.sh -- command-line file sharing](https://transfer.sh/)
- [file.io -- ephemeral file sharing](https://www.file.io/)
- [GitHub CLI -- gh gist create](https://cli.github.com/manual/gh_gist_create)
- [GitHub -- file size and repo limits](https://docs.github.com/en/repositories/working-with-files/managing-large-files/about-large-files-on-github)
- [Cloudflare R2 pricing and free tier](https://developers.cloudflare.com/r2/pricing/)
- [Cloudflare Workers documentation](https://developers.cloudflare.com/workers/)
- [OpenSSL enc -- AES-256-CBC encryption](https://docs.openssl.org/3.3/man1/openssl-enc/)
- [python-websocket-server -- no dependency WebSocket](https://github.com/Pithikos/python-websocket-server)
