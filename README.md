# 402Street — Device Simulator

Simple CLI simulator that mimics a Node One device:
- connects to gateway WebSocket: `wss://<gateway>/ws?deviceId=...`
- prints received unlock events
- can request a payment challenge (`GET /pay/:deviceId`)
- can automatically send a mock verify (`POST /verify`) to gateway (if you run a verifier stub)

## Quick start

1. Clone or create this folder, then:
```bash
npm install
cp .env.example .env
# edit .env if needed (GATEWAY_BASE, DEVICE_ID)
```
**Run simulator (default):**
```bash
node src/sim.js
```
**CLI options:**
```sql
--id <deviceId>          Device id (overrides .env DEVICE_ID)
--gateway <url>          Gateway base URL (overrides .env GATEWAY_BASE)
--auto-verify [true|false]  If true, after issuing a payment, the simulator will send a mock POST /verify
--amount <num>           Amount used for GET /pay request (default 0.25)
--currency <CUR>         Currency for request (default USDC)
```
**Examples:**
```bash
# connect as DEVICE_1 and wait for unlock events
node src/sim.js --id DEVICE_1

# request payment challenge, then auto-verify
node src/sim.js --id VEND_10 --auto-verify true --amount 0.5
```
Behavior

- When started it opens a WebSocket to GATEWAY_BASE + WS_PATH?deviceId=... and logs any incoming messages.
- If you issue the --auto-verify flag, after requesting /pay/:deviceId the simulator will:
1. parse the reference from the gateway response,
2. wait for VERIFY_DELAY_MS,
3. POST { txid: "<VERIFIER_TX_PREFIX><random>", deviceId, reference } to /verify on the gateway.

**This is handy for end-to-end testing without a real wallet or blockchain.**


---

## 5) `src/sim.js`
```js
#!/usr/bin/env node
/**
 * device-simulator: connects to gateway WS, requests payments and can auto-verify.
 *
 * Usage:
 *  node sim.js --id DEVICE_1 --gateway http://localhost:8080 --auto-verify true
 */

import WebSocket from "ws";
import axios from "axios";
import minimist from "minimist";
import dotenv from "dotenv";
import { randomBytes } from "crypto";
import path from "path";

dotenv.config({ path: path.resolve(process.cwd(), ".env") });

const argv = minimist(process.argv.slice(2), {
  string: ["id", "gateway", "currency", "amount"],
  boolean: ["auto-verify"],
  alias: { id: "i", gateway: "g", "auto-verify": "a" },
  default: {}
});

const DEVICE_ID = argv.id || process.env.DEVICE_ID || "DEVICE_1";
const GATEWAY_BASE = (argv.gateway || process.env.GATEWAY_BASE || "http://localhost:8080").replace(/\/+$/, "");
const WS_PATH = process.env.WS_PATH || "/ws";
const AUTO_VERIFY = typeof argv["auto-verify"] !== "undefined" ? !!argv["auto-verify"] : (process.env.AUTO_VERIFY === "true");
const VERIFY_DELAY_MS = parseInt(process.env.VERIFY_DELAY_MS || "1000", 10);
const VERIFIER_TX_PREFIX = process.env.VERIFIER_TX_PREFIX || "SIMTX_";

const defaultAmount = parseFloat(argv.amount || process.env.DEFAULT_AMOUNT || "0.25");
const defaultCurrency = argv.currency || process.env.DEFAULT_CURRENCY || "USDC";

function log(...args) { console.log(new Date().toISOString(), ...args); }

function deriveWsUrl(httpBase, wsPath) {
  const u = new URL(httpBase);
  u.protocol = (u.protocol === "https:" ? "wss:" : "ws:");
  u.pathname = wsPath;
  u.search = `?deviceId=${encodeURIComponent(DEVICE_ID)}`;
  return u.toString();
}

async function requestPayment(amount = defaultAmount, currency = defaultCurrency) {
  try {
    const url = `${GATEWAY_BASE}/pay/${encodeURIComponent(DEVICE_ID)}?amount=${encodeURIComponent(amount)}&currency=${encodeURIComponent(currency)}`;
    log("[HTTP] GET", url);
    const r = await axios.get(url, { validateStatus: () => true });
    log("[HTTP] Response status:", r.status);
    if (r.status === 402) {
      const paymentHeader = r.headers["x-payment-request"];
      const bodyPayment = r.data?.payment;
      const payment = paymentHeader ? JSON.parse(paymentHeader) : bodyPayment;
      log("[PAYMENT] Payment request:", payment);
      return payment;
    } else {
      log("[WARN] Unexpected status (expected 402). Body:", r.data);
      return null;
    }
  } catch (e) {
    log("[ERROR] requestPayment:", e.message || e);
    return null;
  }
}

async function postVerify(txid, reference) {
  try {
    const url = `${GATEWAY_BASE}/verify`;
    const body = { txid, deviceId: DEVICE_ID, reference };
    log("[HTTP] POST", url, body);
    const r = await axios.post(url, body, { timeout: 7000, validateStatus: () => true });
    log("[HTTP] verify response:", r.status, r.data);
    return r.data;
  } catch (e) {
    log("[ERROR] postVerify:", e.message || e);
    return null;
  }
}

function randomHex(n=8) { return randomBytes(n).toString("hex"); }

async function main() {
  log("Starting device simulator", { DEVICE_ID, GATEWAY_BASE, WS_PATH, AUTO_VERIFY });

  // WebSocket connection
  const wsUrl = deriveWsUrl(GATEWAY_BASE, WS_PATH);
  log("[WS] connecting to", wsUrl);
  const ws = new WebSocket(wsUrl);

  ws.on("open", () => log("[WS] connected as", DEVICE_ID));
  ws.on("close", (code, reason) => log("[WS] closed", code, reason?.toString?.() ?? reason));
  ws.on("error", (err) => log("[WS] error", err.message || err));
  ws.on("message", (d) => {
    try {
      const s = typeof d === "string" ? d : d.toString();
      const msg = JSON.parse(s);
      log("[WS] message:", msg);
    } catch (e) {
      log("[WS] raw message:", d.toString());
    }
  });

  // Simple interactive loop in console (non-blocking)
  process.stdin.setEncoding("utf8");
  process.stdin.on("data", async (chunk) => {
    const line = chunk.toString().trim();
    if (!line) return;
    const [cmd, ...rest] = line.split(/\s+/);
    try {
      if (cmd === "pay") {
        const amount = rest[0] ? parseFloat(rest[0]) : defaultAmount;
        const currency = rest[1] || defaultCurrency;
        const payment = await requestPayment(amount, currency);
        if (payment && AUTO_VERIFY) {
          const fakeTx = VERIFIER_TX_PREFIX + randomHex(6);
          log("[AUTO_VERIFY] will post verify in", VERIFY_DELAY_MS, "ms as", fakeTx);
          setTimeout(async () => {
            await postVerify(fakeTx, payment.reference);
          }, VERIFY_DELAY_MS);
        }
      } else if (cmd === "verify") {
        const tx = rest[0] || (VERIFIER_TX_PREFIX + randomHex(6));
        const ref = rest[1];
        if (!ref) return log("Usage: verify <txid> <reference>");
        await postVerify(tx, ref);
      } else if (cmd === "exit" || cmd === "quit") {
        log("Exiting...");
        ws.close();
        process.exit(0);
      } else if (cmd === "help") {
        console.log("Commands:\n  pay [amount] [currency]   - request payment\n  verify <txid> <ref>       - post verify\n  quit                      - exit\n");
      } else {
        log("Unknown command. Type 'help' for commands.");
      }
    } catch (e) {
      log("Command error:", e.message || e);
    }
  });

  // Print instructions
  log("Interactive console ready. Type 'help' for commands.");
}

main().catch((e) => {
  console.error("Fatal error:", e);
  process.exit(1);
});
