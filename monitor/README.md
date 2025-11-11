# monitor — CLI untuk memantau token baru Pump.fun (Solana)

Tool CLI berbasis Node.js/TypeScript untuk **melacak semua token baru** yang dibuat di **Pump.fun** (Solana) dan menampilkan ringkasannya langsung di terminal.

## Fitur

* Real-time stream via **PumpPortal WebSocket**.
* **Filter ketat** hanya untuk token baru Pump.fun: `pool === "pump"`, `txType === "create"`, **mint berakhiran `pump`**.
* Output CLI rapi dengan format:

  ```
  nama               : <nama token>
  mint address       : <mint address berakhiran pump>
  tx hash            : <signature>
  creator token      : <public key pembuat>
  waktu token di buat: <ISO waktu on-chain>
  ```
* Ambil **waktu on-chain (blockTime)** secara **batched** via `getSignatureStatuses` untuk menghindari rate limit.
* **Retry & throttle** otomatis ketika RPC membatasi (HTTP 429).
* Dukungan **RPC pribadi** via `.env` agar lebih stabil.

---

## Prasyarat

* Ubuntu 22.04 (atau lingkungan Linux serupa)
* Node.js LTS (disarankan via nvm)

```bash
sudo apt-get update && sudo apt-get install -y build-essential git curl
# Pasang Node via nvm
curl -fsSL https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
export NVM_DIR="$HOME/.nvm"
source "$NVM_DIR/nvm.sh"
nvm install --lts
nvm use --lts

node -v && npm -v   # verifikasi
```

---

## Setup Proyek

```bash
# 1) Buat folder proyek
mkdir -p ~/monitor && cd ~/monitor
npm init -y

# 2) Install dependencies
npm i ws chalk @solana/web3.js dotenv
npm i -D typescript tsx @types/node @types/ws

# 3) Buat tsconfig.json
cat > tsconfig.json << 'EOF'
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "rootDir": "src",
    "outDir": "dist",
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true,
    "resolveJsonModule": true,
    "skipLibCheck": true,
    "strict": true
  },
  "include": ["src"]
}
EOF

# 4) Atur scripts di package.json
npm pkg set type="module"
npm pkg set scripts.dev="tsx src/index.ts"
npm pkg set scripts.build="tsc"
npm pkg set scripts.start="node dist/index.js"
```

---

## Konfigurasi `.env`

Buat file `.env` di root proyek. Isi dengan endpoint Solana mainnet pribadi (opsional tapi direkomendasikan agar tidak 429):

```dotenv
RPC_URL=https://<endpoint-solana-mainnet-pribadi>
```

> Jika tidak diisi, aplikasi akan fallback ke endpoint publik standar.

---

## Kode Utama (`src/index.ts`)

Buat folder dan file berikut:

```bash
mkdir -p src
```

**`src/index.ts`**

```ts
import WebSocket from "ws";
import chalk from "chalk";
import { Connection, clusterApiUrl } from "@solana/web3.js";
import "dotenv/config";

const WS_URL = "wss://pumpportal.fun/api/data";

// Pakai RPC pribadi jika ada; fallback ke publik
const RPC = process.env.RPC_URL || clusterApiUrl("mainnet-beta");
const connection = new Connection(RPC, "confirmed");

const nowStr = () =>
  new Date().toISOString().replace("T", " ").replace("Z", "");

let ws: WebSocket | null = null;
let reconnectMs = 2000;

/** --------------------------
 *  BATCHER: blockTime by signatures (hemat kuota)
 *  -------------------------- */
const pendingResolvers = new Map<string, (iso: string) => void>();
let batch: string[] = [];
let batching = false;

function requestBlockTime(signature?: string): Promise<string> {
  if (!signature) return Promise.resolve(nowStr());
  return new Promise((resolve) => {
    if (pendingResolvers.has(signature)) {
      pendingResolvers.set(signature, resolve);
    } else {
      pendingResolvers.set(signature, resolve);
      batch.push(signature);
    }
    scheduleBatch();
  });
}

function scheduleBatch() {
  if (batching) return;
  batching = true;
  setTimeout(processBatch, 1500); // jalankan tiap 1.5s
}

async function processBatch() {
  batching = false;
  const sigs = batch.splice(0, 25); // batas aman batch
  if (sigs.length === 0) return;

  try {
    // Batch status; ambil blockTime bila tersedia
    const res = await connection.getSignatureStatuses(sigs as any, {
      searchTransactionHistory: true,
    } as any);
    const values = res?.value || [];
    for (let i = 0; i < sigs.length; i++) {
      const sig = sigs[i];
      const st: any = values[i];
      const blk = st?.blockTime;
      const iso = blk
        ? new Date(blk * 1000).toISOString().replace("T", " ").replace("Z", "")
        : nowStr();
      pendingResolvers.get(sig)?.(iso);
      pendingResolvers.delete(sig);
    }
  } catch {
    // Jika 429/error, requeue dan coba lagi
    for (const sig of sigs) batch.push(sig);
    setTimeout(processBatch, 2000);
  }

  if (batch.length > 0) scheduleBatch();
}

/** --------------------------
 *  WebSocket Pump.fun
 *  -------------------------- */
function connect() {
  ws = new WebSocket(WS_URL);

  ws.on("open", () => {
    console.log(chalk.green(`[${nowStr()}] Connected → ${WS_URL}`));
    ws!.send(JSON.stringify({ method: "subscribeNewToken" }));
    console.log(chalk.cyan(`[${nowStr()}] Sent subscribeNewToken`));
    console.log(chalk.gray(`[info] RPC in use: ${RPC}`));
  });

  ws.on("message", async (data) => {
    let msg: any;
    try {
      msg = JSON.parse(data.toString());
    } catch {
      return;
    }

    // Filter khusus Pump.fun token baru
    const pool = msg?.pool;
    const txType = msg?.txType;
    const mintRaw =
      msg?.mint ?? msg?.tokenMint ?? msg?.token ?? msg?.mintAddress ?? "";
    const isPumpMint = typeof mintRaw === "string" && mintRaw.endsWith("pump");
    if (pool !== "pump" || txType !== "create" || !isPumpMint) {
      return;
    }

    // Normalisasi field
    const name =
      msg?.name ?? msg?.tokenName ?? msg?.metadata?.name ?? "N/A";
    const mint = mintRaw || "N/A";
    const txHash =
      msg?.signature ?? msg?.tx ?? msg?.txHash ?? msg?.transaction ?? "";
    const creator =
      msg?.creator ??
      msg?.user ??
      msg?.owner ??
      msg?.creatorToken ??
      msg?.traderPublicKey ??
      "N/A";

    // Ambil blockTime via batcher (hemat kuota)
    const createdAt = await requestBlockTime(txHash);

    // Output format yang diminta
    console.log(chalk.bold(`\n[${nowStr()}] NEW TOKEN (Solana)`));
    console.log(`nama               : ${name}`);
    console.log(`mint address       : ${mint}`);
    console.log(`tx hash            : ${txHash || "N/A"}`);
    console.log(`creator token      : ${creator}`);
    console.log(`waktu token di buat: ${createdAt}`);
  });

  ws.on("close", (code, reason) => {
    console.log(
      chalk.red(`[${nowStr()}] Disconnected (${code}) ${reason.toString()}`)
    );
    scheduleReconnect();
  });

  ws.on("error", (err: any) => {
    console.log(chalk.red(`[${nowStr()}] WebSocket error: ${err?.message || err}`));
  });
}

function scheduleReconnect() {
  try { ws?.terminate(); } catch {}
  ws = null;
  const wait = reconnectMs;
  reconnectMs = Math.min(reconnectMs * 2, 30000);
  console.log(chalk.yellow(`[${nowStr()}] Reconnecting in ${wait} ms...`));
  setTimeout(connect, wait);
}

connect();

process.on("SIGINT", () => {
  console.log(chalk.gray(`\n[${nowStr()}] Bye.`));
  try { ws?.close(); } catch {}
  process.exit(0);
});
```

---

## Menjalankan

**Mode Dev (langsung TS):**

```bash
npm run dev
```

**Build & Run (produksi):**

```bash
npm run build
npm run start
```

Saat token baru muncul dari Pump.fun, output akan berupa 5 baris di atas.

---

## Menjalankan terus di VPS (opsional)

```bash
npm i -g pm2
pm2 start dist/index.js --name monitor
pm2 logs monitor
pm2 startup   # ikuti instruksi lalu
pm2 save
```

---

## Troubleshooting

* **Tidak ada output?**

  * Pastikan koneksi WS aktif dan tidak ada firewall yang memblokir.
  * Cek apakah ada token baru yang benar-benar muncul dalam rentang waktu itu.
* **Spam 429 Too Many Requests?**

  * Isi `RPC_URL` dengan endpoint pribadi.
  * Interval batch bisa dinaikkan (mis. 2000–2500 ms) atau batch size diturunkan.
* **ESM/CJS error saat dev:**

  * Proyek ini sudah di-set ESM penuh + runner `tsx`. Jika perlu, jalankan dengan `npm run build && npm run start` tanpa `tsx`.

---

## Lisensi

Gunakan seperlunya. Tidak ada garansi; gunakan dengan tanggung jawab sendiri.

---

## .env.example

```dotenv
# RPC Solana Mainnet (opsional tapi disarankan)
# Contoh: Helius, QuickNode, Triton, Alchemy, dll.
# Jika kosong, aplikasi akan memakai endpoint publik default (bisa lebih mudah rate-limited).
RPC_URL=https://your-solana-mainnet-rpc.example.com
```

