# 🤖 Bot "Blast Nomor" — Node.js + Telegraf (Siap Jalan di Termux)

Sebelum masuk ke kode, satu catatan teknis penting: **Telegram Bot API tidak bisa mengirim pesan ke nomor HP mentah**. Bot hanya bisa mengirim ke user Telegram yang pernah menekan START ke bot tersebut. Karena itu saya desain kontak terkumpul lewat **tautan undangan opt-in** (`?start=ref_...`) — ini satu-satunya cara yang resmi, dan sekaligus memenuhi syarat anti-spam yang kamu sebut.

Saya juga mengganti MySQL ke **penyimpanan file JSON** agar **pasti jalan di Termux tanpa install server database** (semua dependensi murni JavaScript, tanpa kompilasi).

---

## 📁 Struktur Project

```
blast-nomor/
├── package.json
├── .env
├── src/
│   ├── index.js              # Entry point
│   ├── config.js             # Konfigurasi & paket
│   ├── db.js                 # Database JSON
│   ├── keyboards.js          # Semua tombol inline
│   ├── helpers.js            # Util kecil
│   ├── state.js              # State percakapan
│   ├── handlers/
│   │   ├── commands.js       # /start /admin /setplan /broadcast
│   │   ├── callbacks.js      # Semua tombol menu
│   │   └── messages.js       # Input user (kontak, pesan, jadwal)
│   └── services/
│       ├── blast.js          # Mesin blast + rate limiter
│       └── scheduler.js      # Cron jadwal per menit
└── data/                     # Auto-dibuat (database.json)
```

---

## 📲 Instalasi di Termux

```bash
pkg update -y && pkg upgrade -y
pkg install -y nodejs-lts nano

mkdir blast-nomor && cd blast-nomor
# buat semua file di bawah ini (nano nama_file), lalu:
npm install
nano .env        # isi BOT_TOKEN & ADMIN_IDS
npm start
```

**Ambil token:** chat [@BotFather](https://t.me/BotFather) → `/newbot` → salin token ke `.env`.
**Ambil ID admin-mu:** jalankan bot, tekan `/start` — bot akan menampilkan ID-mu, lalu isi ke `.env`.

Agar tidak mati saat HP dikunci:

```bash
termux-wake-lock
nohup npm start > blast.log 2>&1 &
tail -f blast.log   # lihat log
```

---

## 📄 `package.json`

```json
{
  "name": "blast-nomor",
  "version": "1.0.0",
  "description": "Bot Telegram Blast Nomor - broadcast ke kontak opt-in",
  "main": "src/index.js",
  "scripts": {
    "start": "node src/index.js"
  },
  "dependencies": {
    "dotenv": "^16.4.5",
    "node-cron": "^3.0.3",
    "telegraf": "^4.16.3"
  }
}
```

## 📄 `.env`

```env
BOT_TOKEN=PASTE_TOKEN_BOTFATHER_DISINI
ADMIN_IDS=
RATE_DELAY_MS=2500
```

## 📄 `src/config.js`

```js
require('dotenv').config();

const cfg = {
  BOT_TOKEN: process.env.BOT_TOKEN || '',
  ADMIN_IDS: (process.env.ADMIN_IDS || '')
    .split(',').map(s => parseInt(s.trim(), 10)).filter(n => !isNaN(n)),
  RATE_DELAY_MS: parseInt(process.env.RATE_DELAY_MS || '2500', 10),

  PLANS: {
    free:    { label: '🆓 Free',    maxContacts: 50 },
    pro:     { label: '💎 Pro',     maxContacts: 500 },
    premium: { label: '👑 Premium', maxContacts: 5000 }
  }
};

cfg.isAdmin = (id) => cfg.ADMIN_IDS.includes(Number(id));

module.exports = cfg;
```

## 📄 `src/helpers.js`

```js
const esc = (s = '') => String(s)
  .replace(/&/g, '&amp;').replace(/</g, '&lt;').replace(/>/g, '&gt;');

const fmtDate = (iso) => iso
  ? new Date(iso).toLocaleString('id-ID', { timeZone: 'Asia/Jakarta' }) : '-';

const truncate = (s = '', n = 80) =>
  s.length > n ? s.slice(0, n) + '…' : s;

const sleep = (ms) => new Promise(r => setTimeout(r, ms));

module.exports = { esc, fmtDate, truncate, sleep };
```

## 📄 `src/state.js`

```js
const states = new Map();

const setState = (userId, s) => states.set(String(userId), s);
const getState = (userId) => states.get(String(userId));
const clearState = (userId) => states.delete(String(userId));

module.exports = { setState, getState, clearState };
```

## 📄 `src/db.js`

```js
const fs = require('fs');
const path = require('path');
const cfg = require('./config');

const DATA_DIR = path.join(__dirname, '..', 'data');
const DB_FILE = path.join(DATA_DIR, 'database.json');

const DEFAULTS = { users: {}, contacts: [], campaigns: [], deliveries: [], drafts: {} };
let db = null;

function load() {
  if (!fs.existsSync(DATA_DIR)) fs.mkdirSync(DATA_DIR, { recursive: true });
  if (fs.existsSync(DB_FILE)) {
    try {
      db = Object.assign({}, DEFAULTS, JSON.parse(fs.readFileSync(DB_FILE, 'utf8')));
      console.log('✅ Database dimuat:', DB_FILE);
    } catch (e) {
      console.error('⚠️ Database rusak, membuat baru:', e.message);
      db = JSON.parse(JSON.stringify(DEFAULTS));
    }
  } else {
    db = JSON.parse(JSON.stringify(DEFAULTS));
    console.log('🆕 Database baru dibuat.');
  }
  save();
}

function save() { fs.writeFileSync(DB_FILE, JSON.stringify(db)); }
function get() { if (!db) load(); return db; }

function nextId(prefix) {
  return prefix + '_' + Date.now().toString(36) + Math.random().toString(36).slice(2, 6);
}

/* ---------- USERS ---------- */
function upsertUser(from) {
  const data = get();
  const key = String(from.id);
  if (!data.users[key]) {
    data.users[key] = {
      telegram_id: from.id,
      username: from.username || '',
      first_name: from.first_name || '',
      plan: 'free',
      credits: 0,
      joined_at: new Date().toISOString()
    };
    save();
  } else {
    const u = data.users[key];
    u.username = from.username || u.username;
    u.first_name = from.first_name || u.first_name;
  }
  return data.users[key];
}

const getUser = (id) => get().users[String(id)];

function setPlan(id, plan) {
  const u = getUser(id);
  if (u && ['free', 'pro', 'premium'].includes(plan)) { u.plan = plan; save(); return true; }
  return false;
}

/* ---------- CONTACTS ---------- */
function maxContacts(ownerId) {
  const u = getUser(ownerId);
  return cfg.PLANS[u ? u.plan : 'free'].maxContacts;
}

function findContact(ownerId, chatId) {
  return get().contacts.find(c => c.owner_id === String(ownerId) && c.chat_id === Number(chatId));
}

function activeContacts(ownerId) {
  return get().contacts.filter(c => c.owner_id === String(ownerId) && !c.opt_out);
}

function addContact(ownerId, chatId, name, username, phone, source) {
  const data = get();
  ownerId = String(ownerId);
  if (findContact(ownerId, chatId)) return { ok: false, reason: 'exists' };
  if (activeContacts(ownerId).length >= maxContacts(ownerId)) return { ok: false, reason: 'limit' };

  const contact = {
    id: nextId('ct'),
    owner_id: ownerId,
    chat_id: Number(chatId),
    name: name || '',
    username: username || '',
    phone: phone || '',
    opt_out: false,
    source: source || 'manual',
    created_at: new Date().toISOString()
  };
  data.contacts.push(contact);
  save();
  return { ok: true, contact };
}

function setOptOut(contactId, val = true) {
  const c = get().contacts.find(x => x.id === contactId);
  if (c) { c.opt_out = val; save(); }
  return c;
}

function removeContactByChatId(ownerId, chatId) {
  const data = get();
  const i = data.contacts.findIndex(c => c.owner_id === String(ownerId) && c.chat_id === Number(chatId));
  if (i === -1) return false;
  data.contacts.splice(i, 1);
  save();
  return true;
}

/* ---------- CAMPAIGNS ---------- */
function createCampaign(ownerId, type, content, status, scheduledAt = null) {
  const c = {
    id: nextId('cmp'),
    owner_id: String(ownerId),
    type, content,
    status,
    total: activeContacts(ownerId).length,
    sent: 0, failed: 0,
    scheduled_at: scheduledAt,
    started_at: null, finished_at: null,
    created_at: new Date().toISOString()
  };
  get().campaigns.push(c);
  save();
  return c;
}

const getCampaign = (id) => get().campaigns.find(c => c.id === id);

function scheduledCampaigns(ownerId) {
  return get().campaigns.filter(c => c.owner_id === String(ownerId) && c.status === 'scheduled');
}

function cancelScheduled(ownerId, id) {
  const c = getCampaign(id);
  if (c && c.owner_id === String(ownerId) && c.status === 'scheduled') {
    c.status = 'cancelled'; save(); return true;
  }
  return false;
}

function campaignHistory(ownerId, limit = 5) {
  return get().campaigns
    .filter(c => c.owner_id === String(ownerId))
    .sort((a, b) => new Date(b.created_at) - new Date(a.created_at))
    .slice(0, limit);
}

/* ---------- DELIVERIES ---------- */
function addDelivery(campaignId, contact, status, error = '') {
  get().deliveries.push({
    id: nextId('dlv'),
    campaign_id: campaignId,
    contact_id: contact.id,
    chat_id: contact.chat_id,
    status, error,
    sent_at: new Date().toISOString()
  });
}

/* ---------- DRAFTS ---------- */
function setDraft(ownerId, draft) { get().drafts[String(ownerId)] = draft; save(); }
function getDraft(ownerId) { return get().drafts[String(ownerId)] || null; }
function clearDraft(ownerId) { delete get().drafts[String(ownerId)]; save(); }

module.exports = {
  load, save, get, nextId,
  upsertUser, getUser, setPlan,
  maxContacts, findContact, activeContacts, addContact, setOptOut, removeContactByChatId,
  createCampaign, getCampaign, scheduledCampaigns, cancelScheduled, campaignHistory,
  addDelivery,
  setDraft, getDraft, clearDraft
};
```

## 📄 `src/keyboards.js`

```js
const { Markup } = require('telegraf');

const mainMenu = () => Markup.inlineKeyboard([
  [Markup.button.callback('📱 Blast Nomor', 'menu:blast')],
  [Markup.button.callback('✉️ Buat Pesan', 'menu:compose')],
  [Markup.button.callback('📊 Statistik', 'menu:stats'), Markup.button.callback('🕐 Jadwal', 'menu:schedule')],
  [Markup.button.callback('💳 Paket', 'menu:plans'), Markup.button.callback('👤 Akun', 'menu:account')]
]);

const blastMenu = () => Markup.inlineKeyboard([
  [Markup.button.callback('➕ Tambah Kontak', 'blast:add')],
  [Markup.button.callback('📋 Daftar Kontak', 'blast:list')],
  [Markup.button.callback('📤 Mulai Blast', 'blast:start')],
  [Markup.button.callback('🗑 Hapus Kontak', 'blast:remove')],
  [Markup.button.callback('⬅️ Menu Utama', 'menu:main')]
]);

const addContactMenu = () => Markup.inlineKeyboard([
  [Markup.button.callback('🔗 Tautan Undangan', 'add:link')],
  [Markup.button.callback('📨 Forward Pesan', 'add:forward')],
  [Markup.button.callback('#️⃣ Kirim Chat ID', 'add:chatid')],
  [Markup.button.callback('⬅️ Kembali', 'menu:blast')]
]);

const composeMenu = () => Markup.inlineKeyboard([
  [Markup.button.callback('📝 Text', 'compose:text')],
  [Markup.button.callback('🖼 Foto + Caption', 'compose:photo')],
  [Markup.button.callback('🎞 Video + Caption', 'compose:video')],
  [Markup.button.callback('⬅️ Menu Utama', 'menu:main')]
]);

const scheduleMenu = () => Markup.inlineKeyboard([
  [Markup.button.callback('📤 Kirim Sekarang', 'blast:start')],
  [Markup.button.callback('🕐 Jadwalkan Blast', 'schedule:new')],
  [Markup.button.callback('📋 Jadwal Aktif', 'schedule:list')],
  [Markup.button.callback('⬅️ Menu Utama', 'menu:main')]
]);

const confirmBlastMenu = () => Markup.inlineKeyboard([
  [Markup.button.callback('🚀 MULAI', 'blast:confirm'), Markup.button.callback('❌ BATAL', 'blast:cancel')]
]);

const stopMenu = () => Markup.inlineKeyboard([[Markup.button.callback('⏹ STOP BLAST', 'blast:stop')]]);

const optOutMenu = (contactId) => Markup.inlineKeyboard([
  [Markup.button.callback('🚫 Berhenti Berlangganan', `optout:${contactId}`)]
]);

const backMain = () => Markup.inlineKeyboard([[Markup.button.callback('⬅️ Menu Utama', 'menu:main')]]);

module.exports = { mainMenu, blastMenu, addContactMenu, composeMenu, scheduleMenu, confirmBlastMenu, stopMenu, optOutMenu, backMain };
```

## 📄 `src/services/blast.js`

```js
const cfg = require('../config');
const db = require('../db');
const kb = require('../keyboards');
const { sleep, fmtDate } = require('../helpers');

const running = new Map(); // campaignId -> { stop }

const isRunning = (id) => running.has(id);

function stopCampaign(id) {
  const r = running.get(id);
  if (r) { r.stop = true; return true; }
  return false;
}

function sendToContact(bot, contact, campaign) {
  const extra = { reply_markup: kb.optOutMenu(contact.id).reply_markup };
  if (campaign.type === 'photo')
    return bot.telegram.sendPhoto(contact.chat_id, campaign.content.file_id,
      { caption: campaign.content.caption || '', ...extra });
  if (campaign.type === 'video')
    return bot.telegram.sendVideo(contact.chat_id, campaign.content.file_id,
      { caption: campaign.content.caption || '', ...extra });
  return bot.telegram.sendMessage(contact.chat_id, campaign.content.text, extra);
}

async function runCampaign(bot, campaignId) {
  const campaign = db.getCampaign(campaignId);
  if (!campaign || isRunning(campaignId)) return;

  const flag = { stop: false };
  running.set(campaignId, flag);

  const contacts = db.activeContacts(campaign.owner_id);
  campaign.status = 'running';
  campaign.total = contacts.length;
  campaign.started_at = new Date().toISOString();
  db.save();

  let progress = null;
  try {
    progress = await bot.telegram.sendMessage(campaign.owner_id,
      `🚀 <b>BLAST DIMULAI</b>\n\n👥 Target: ${contacts.length} kontak\n⏱ Jeda: ${cfg.RATE_DELAY_MS / 1000}s / pesan\n\nTekan ⏹ untuk menghentikan.`,
      { parse_mode: 'HTML', reply_markup: kb.stopMenu().reply_markup });
  } catch (_) {}

  let sent = 0, failed = 0, i = 0;

  for (const contact of contacts) {
    if (flag.stop) break;
    i++;

    try {
      await sendToContact(bot, contact, campaign);
      sent++;
      db.addDelivery(campaignId, contact, 'sent');
    } catch (err) {
      const code = err.code || (err.response && err.response.error_code);
      const retryAfter = err.parameters && err.parameters.retry_after;

      if (code === 429 && retryAfter) {
        // Rate limit Telegram — tunggu lalu coba 1x lagi
        await sleep((retryAfter + 1) * 1000);
        try {
          await sendToContact(bot, contact, campaign);
          sent++;
          db.addDelivery(campaignId, contact, 'sent');
        } catch (e2) {
          failed++;
          db.addDelivery(campaignId, contact, 'failed', (e2.description || e2.message).slice(0, 200));
        }
      } else {
        failed++;
        let desc = err.description || err.message || 'unknown';
        if (code === 403) { contact.opt_out = true; desc += ' → ditandai opt-out (user block bot)'; }
        db.addDelivery(campaignId, contact, 'failed', desc.slice(0, 200));
      }
    }

    campaign.sent = sent;
    campaign.failed = failed;

    if (i % 10 === 0 || i === contacts.length) {
      db.save();
      if (progress) {
        bot.telegram.editMessageText(progress.chat.id, progress.message_id, undefined,
          `📡 <b>PROGRESS BLAST</b>\n\n✅ Terkirim: ${sent}\n❌ Gagal: ${failed}\n📊 Progres: ${i}/${contacts.length}`,
          { parse_mode: 'HTML', reply_markup: kb.stopMenu().reply_markup }).catch(() => {});
      }
    }

    if (i < contacts.length) await sleep(cfg.RATE_DELAY_MS);
  }

  campaign.status = flag.stop ? 'stopped' : 'done';
  campaign.finished_at = new Date().toISOString();
  db.save();
  running.delete(campaignId);

  bot.telegram.sendMessage(campaign.owner_id,
    `${flag.stop ? '⏹ <b>BLAST DIHENTIKAN</b>' : '✅ <b>BLAST SELESAI</b>'}\n\n` +
    `🆔 Campaign: <code>${campaign.id}</code>\n` +
    `✅ Terkirim: ${sent}\n❌ Gagal: ${failed}\n📊 Target: ${contacts.length}\n` +
    `🕐 Selesai: ${fmtDate(campaign.finished_at)}`,
    { parse_mode: 'HTML' }).catch(() => {});
}

module.exports = { runCampaign, stopCampaign, isRunning };
```

## 📄 `src/services/scheduler.js`

```js
const cron = require('node-cron');
const db = require('../db');
const { runCampaign } = require('./blast');

function start(bot) {
  cron.schedule('* * * * *', () => {
    const data = db.get();
    const now = Date.now();
    for (const c of data.campaigns) {
      if (c.status === 'scheduled' && c.scheduled_at && new Date(c.scheduled_at).getTime() <= now) {
        runCampaign(bot, c.id).catch(e => console.error('Scheduler error:', e.message));
      }
    }
  });
  console.log('⏰ Scheduler aktif — cek jadwal setiap menit');
}

module.exports = { start };
```

## 📄 `src/handlers/commands.js`

```js
const cfg = require('../config');
const db = require('../db');
const kb = require('../keyboards');
const { esc, sleep } = require('../helpers');
const { clearState } = require('../state');

function register(bot) {
  bot.start(async (ctx) => {
    clearState(ctx.from.id);
    db.upsertUser(ctx.from);

    // ===== Opt-in lewat tautan undangan: ?start=ref_<ownerId> =====
    const payload = ctx.startPayload || '';
    if (payload.startsWith('ref_')) {
      const ownerId = payload.slice(4);
      const owner = db.getUser(ownerId);
      if (owner && ownerId !== String(ctx.from.id)) {
        const res = db.addContact(ownerId, ctx.from.id, ctx.from.first_name, ctx.from.username, '', 'link');
        if (res.ok) {
          await ctx.reply(`✅ Halo <b>${esc(ctx.from.first_name)}</b>! Kamu terdaftar untuk menerima info dari admin ini. 🎉`, { parse_mode: 'HTML' });
          bot.telegram.sendMessage(ownerId,
            `🔔 <b>Kontak baru terdaftar!</b>\n\n👤 ${esc(ctx.from.first_name)}${ctx.from.username ? ' (@' + esc(ctx.from.username) + ')' : ''}\n🆔 <code>${ctx.from.id}</code>\n📤 Sumber: Tautan Undangan`,
            { parse_mode: 'HTML' }).catch(() => {});
        } else if (res.reason === 'exists') {
          await ctx.reply('ℹ️ Kamu sudah terdaftar sebelumnya.');
        } else {
          await ctx.reply('⚠️ Daftar kontak admin sudah penuh. Hubungi admin.');
        }
      }
    }

    if (cfg.ADMIN_IDS.length === 0) {
      await ctx.reply(`ℹ️ ID Telegram kamu: <code>${ctx.from.id}</code>\n\nMasukkan ID ini ke file .env pada baris ADMIN_IDS agar akunmu jadi admin.`, { parse_mode: 'HTML' });
    }

    await ctx.reply(
      `🤖 <b>BLAST NOMOR</b>\n\n` +
      `Selamat datang, ${esc(ctx.from.first_name)}! 👋\n` +
      `Bot broadcast ke kontak Telegram yang sudah memberi izin (opt-in).\n\n👇 Pilih menu di bawah:`,
      { parse_mode: 'HTML', reply_markup: kb.mainMenu().reply_markup }
    );
  });

  bot.help(ctx => ctx.reply(
    `📖 <b>BANTUAN</b>\n\n` +
    `/start — Menu utama\n/admin — Panel admin (khusus admin)\n\n` +
    `<b>Alur singkat:</b>\n1️⃣ Buat pesan di ✉️ Buat Pesan\n2️⃣ Tambah kontak di 📱 Blast Nomor\n3️⃣ 📤 Mulai Blast atau 🕐 Jadwalkan\n\n` +
    `⚠️ Pesan hanya dikirim ke kontak yang opt-in.`,
    { parse_mode: 'HTML', reply_markup: kb.backMain().reply_markup }
  ));

  /* ---------- ADMIN ---------- */
  bot.command('admin', async (ctx) => {
    if (!cfg.isAdmin(ctx.from.id)) return ctx.reply('⛔ Perintah khusus admin.');
    const data = db.get();
    await ctx.reply(
      `🛡 <b>ADMIN PANEL</b>\n\n` +
      `👥 Total pengguna: ${Object.keys(data.users).length}\n` +
      `📇 Total kontak: ${data.contacts.length}\n` +
      `📣 Total campaign: ${data.campaigns.length}\n` +
      `▶️ Berjalan: ${data.campaigns.filter(c => c.status === 'running').length}\n` +
      `🕐 Terjadwal: ${data.campaigns.filter(c => c.status === 'scheduled').length}\n` +
      `⏱ Rate delay: ${cfg.RATE_DELAY_MS} ms\n\n` +
      `<b>Perintah admin:</b>\n` +
      `/setplan &lt;id&gt; &lt;free|pro|premium&gt;\n` +
      `/setdelay &lt;ms&gt;\n` +
      `/broadcast &lt;teks&gt;`,
      { parse_mode: 'HTML' }
    );
  });

  bot.command('setplan', async (ctx) => {
    if (!cfg.isAdmin(ctx.from.id)) return;
    const [, id, plan] = ctx.message.text.split(/\s+/);
    if (!id || !plan) return ctx.reply('Format: /setplan <id> <free|pro|premium>');
    ctx.reply(db.setPlan(id, plan) ? `✅ Plan ${id} → ${plan}` : '❌ Gagal. Cek ID user & nama plan.');
  });

  bot.command('setdelay', async (ctx) => {
    if (!cfg.isAdmin(ctx.from.id)) return;
    const ms = parseInt(ctx.message.text.split(/\s+/)[1], 10);
    if (!ms || ms < 500) return ctx.reply('Format: /setdelay <ms> (min. 500)');
    cfg.RATE_DELAY_MS = ms;
    ctx.reply(`✅ Rate delay diubah: ${ms} ms`);
  });

  bot.command('broadcast', async (ctx) => {
    if (!cfg.isAdmin(ctx.from.id)) return;
    const text = ctx.message.text.split(/\s+/).slice(1).join(' ');
    if (!text) return ctx.reply('Format: /broadcast <teks>');
    const ids = Object.keys(db.get().users);
    const status = await ctx.reply(`📡 Mengirim ke ${ids.length} pengguna...`);
    let ok = 0, fail = 0;
    for (const id of ids) {
      try {
        await bot.telegram.sendMessage(id, `📣 <b>PENGUMUMAN</b>\n\n${esc(text)}`, { parse_mode: 'HTML' });
        ok++;
      } catch (_) { fail++; }
      await sleep(300);
    }
    await ctx.telegram.editMessageText(status.chat.id, status.message_id, undefined,
      `📡 Broadcast selesai.\n✅ ${ok} · ❌ ${fail}`);
  });
}

module.exports = { register };
```

## 📄 `src/handlers/callbacks.js`

```js
const { Markup } = require('telegraf');
const cfg = require('../config');
const db = require('../db');
const kb = require('../keyboards');
const { esc, truncate, fmtDate } = require('../helpers');
const { setState } = require('../state');
const { runCampaign, stopCampaign } = require('../services/blast');

async function edit(ctx, text, markup) {
  const extra = { parse_mode: 'HTML', ...(markup ? { reply_markup: markup.reply_markup } : {}) };
  try {
    await ctx.editMessageText(text, extra);
  } catch (e) {
    if (String(e.message).includes('message is not modified')) return;
    await ctx.reply(text, extra).catch(() => {});
  }
}

function register(bot) {
  bot.on('callback_query', async (ctx) => {
    const data = ctx.callbackQuery.data || '';
    const userId = ctx.from.id;
    const answer = (t) => ctx.answerCbQuery(t).catch(() => {});

    try {
      /* ============ NAVIGASI UTAMA ============ */
      if (data === 'menu:main') {
        await edit(ctx, `🤖 <b>BLAST NOMOR</b>\n\nHalo ${esc(ctx.from.first_name)}! 👋\n👇 Pilih menu:`, kb.mainMenu());
        return answer();
      }

      if (data === 'menu:blast') {
        await edit(ctx, `📱 <b>BLAST NOMOR</b>\n\n📇 Kontak aktif: <b>${db.activeContacts(userId).length}</b>\n👇 Pilih:`, kb.blastMenu());
        return answer();
      }

      if (data === 'menu:compose') {
        const d = db.getDraft(userId);
        const info = d
          ? `📝 <b>Draft saat ini:</b> ${d.type}\n<i>${esc(truncate(d.type === 'text' ? d.content.text : (d.content.caption || '(media, tanpa caption)'), 100))}</i>`
          : '📝 Belum ada draft pesan.';
        await edit(ctx, `✉️ <b>BUAT PESAN</b>\n\n${info}\n\nPilih jenis pesan:`, kb.composeMenu());
        return answer();
      }

      if (data === 'menu:stats') {
        const all = db.get();
        const mine = all.campaigns.filter(c => c.owner_id === String(userId));
        const contacts = all.contacts.filter(c => c.owner_id === String(userId));
        const active = contacts.filter(c => !c.opt_out).length;
        await edit(ctx,
          `📊 <b>STATISTIK</b>\n\n` +
          `📇 Kontak: ${contacts.length} (✅ aktif ${active} · 🚫 opt-out ${contacts.length - active})\n` +
          `📣 Campaign: ${mine.length} (▶️ ${mine.filter(c => c.status === 'running').length} · 🕐 ${mine.filter(c => c.status === 'scheduled').length})\n` +
          `✅ Pesan terkirim: ${mine.reduce((a, c) => a + (c.sent || 0), 0)}\n` +
          `❌ Pesan gagal: ${mine.reduce((a, c) => a + (c.failed || 0), 0)}`,
          kb.backMain());
        return answer();
      }

      if (data === 'menu:schedule') {
        await edit(ctx, `🕐 <b>JADWAL BLAST</b>\n\nJadwal aktif: <b>${db.scheduledCampaigns(userId).length}</b>\n👇 Pilih:`, kb.scheduleMenu());
        return answer();
      }

      if (data === 'menu:plans') {
        await edit(ctx,
          `💳 <b>PAKET LANGGANAN</b>\n\n` +
          `🆓 <b>Free</b> — maks 50 kontak\n` +
          `💎 <b>Pro</b> — maks 500 kontak\n` +
          `👑 <b>Premium</b> — maks 5.000 kontak\n\n` +
          `Pilih paket untuk info upgrade:`,
          Markup.inlineKeyboard([
            [Markup.button.callback('🆓 Free', 'plan:free'), Markup.button.callback('💎 Pro', 'plan:pro')],
            [Markup.button.callback('👑 Premium', 'plan:premium')],
            [Markup.button.callback('⬅️ Menu Utama', 'menu:main')]
          ]));
        return answer();
      }

      if (data === 'menu:account') {
        const u = db.getUser(userId) || {};
        const plan = cfg.PLANS[u.plan || 'free'];
        const hist = db.campaignHistory(userId, 5);
        const histStr = hist.length
          ? hist.map(c => `• <code>${c.id}</code> — ${c.status} · ${c.sent}/${c.total}${c.scheduled_at ? ' · 🕐 ' + fmtDate(c.scheduled_at) : ''}`).join('\n')
          : '  (belum ada campaign)';
        await edit(ctx,
          `👤 <b>AKUN</b>\n\n` +
          `🆔 ID: <code>${userId}</code>\n` +
          `👤 Nama: ${esc(ctx.from.first_name)}\n` +
          `💬 Username: ${ctx.from.username ? '@' + esc(ctx.from.username) : '-'}\n` +
          `💳 Paket: ${plan.label} (maks ${plan.maxContacts})\n` +
          `📇 Kontak terpakai: ${db.activeContacts(userId).length}/${plan.maxContacts}\n` +
          `📅 Bergabung: ${fmtDate(u.joined_at)}\n\n` +
          `📜 <b>Riwayat 5 campaign terakhir:</b>\n${histStr}`,
          kb.backMain());
        return answer();
      }

      /* ============ TAMBAH KONTAK ============ */
      if (data === 'blast:add') {
        await edit(ctx,
          `➕ <b>TAMBAH KONTAK</b>\n\n` +
          `🔗 <b>Tautan Undangan</b> — paling mudah. Bagikan link; siapa pun yang menekan START otomatis masuk daftarmu (opt-in ✅).\n\n` +
          `📨 <b>Forward Pesan</b> — forward satu pesan dari orang yang mau ditambahkan.\n\n` +
          `#️⃣ <b>Chat ID</b> — kirim angka ID Telegram-nya.`,
          kb.addContactMenu());
        return answer();
      }

      if (data === 'add:link') {
        const me = await bot.telegram.getMe();
        await edit(ctx,
          `🔗 <b>TAUTAN UNDANGAN</b>\n\n` +
          `<code>https://t.me/${me.username}?start=ref_${userId}</code>\n\n` +
          `Saat seseorang menekan <b>START</b> lewat link ini, dia otomatis masuk daftar kontakmu dan kamu dapat notifikasi. Ini metode paling aman karena mereka sadar & setuju menerima pesanmu (opt-in).`,
          kb.addContactMenu());
        return answer();
      }

      if (data === 'add:forward') {
        setState(userId, { action: 'add_forward' });
        await edit(ctx,
          `📨 <b>TAMBAH VIA FORWARD</b>\n\nForward (teruskan) satu pesan dari orang yang ingin ditambahkan ke chat ini.\n\n⚠️ Jika identitas pengirim tersembunyi (privasi), pakai metode 🔗 tautan undangan.`,
          kb.addContactMenu());
        return answer();
      }

      if (data === 'add:chatid') {
        setState(userId, { action: 'add_chatid' });
        await edit(ctx, `#️⃣ <b>TAMBAH VIA CHAT ID</b>\n\nKirim angka chat ID Telegram, contoh:\n<code>123456789</code>`, kb.addContactMenu());
        return answer();
      }

      /* ============ DAFTAR & HAPUS ============ */
      if (data === 'blast:list') {
        const contacts = db.get().contacts.filter(c => c.owner_id === String(userId));
        if (!contacts.length) {
          await edit(ctx, '📇 <b>Daftar kontak masih kosong.</b>', kb.blastMenu());
          return answer();
        }
        const rows = contacts.slice(0, 30).map((c, i) =>
          `${i + 1}. ${esc(c.name || 'Tanpa nama')}${c.username ? ' (@' + esc(c.username) + ')' : ''}\n     🆔 <code>${c.chat_id}</code> · ${c.opt_out ? '🚫 opt-out' : '✅ aktif'} · via ${c.source}`);
        await edit(ctx,
          `📋 <b>DAFTAR KONTAK</b> — total ${contacts.length}\n\n${rows.join('\n')}` +
          (contacts.length > 30 ? `\n\n… dan ${contacts.length - 30} lainnya` : ''),
          kb.blastMenu());
        return answer();
      }

      if (data === 'blast:remove') {
        setState(userId, { action: 'remove_contact' });
        await edit(ctx,
          `🗑 <b>HAPUS KONTAK</b>\n\nKirim <b>chat ID</b> kontak yang ingin dihapus (lihat di 📋 Daftar Kontak).`,
          kb.blastMenu());
        return answer();
      }

      /* ============ MULAI BLAST ============ */
      if (data === 'blast:start') {
        const draft = db.getDraft(userId);
        if (!draft) {
          await edit(ctx, '✉️ <b>Belum ada pesan!</b>\nBuat dulu di ✉️ Buat Pesan.', kb.composeMenu());
          return answer();
        }
        const contacts = db.activeContacts(userId);
        if (!contacts.length) {
          await edit(ctx, '📇 <b>Belum ada kontak aktif!</b>\nTambah kontak dulu di 📱 Blast Nomor.', kb.blastMenu());
          return answer();
        }
        const preview = draft.type === 'text' ? draft.content.text : (draft.content.caption || '(media tanpa caption)');
        await edit(ctx,
          `📤 <b>MULAI BLAST</b>\n\n` +
          `👥 Target : ${contacts.length} kontak\n` +
          `✉️ Tipe   : ${draft.type}\n` +
          `📝 Pesan  : ${esc(truncate(preview, 120))}\n` +
          `⏱ Mode   : Sekarang (jeda ${cfg.RATE_DELAY_MS / 1000} detik/pesan)\n\n` +
          `⚠️ Blast hanya dikirim ke kontak yang terdaftar & belum opt-out.`,
          kb.confirmBlastMenu());
        return answer();
      }

      if (data === 'blast:confirm') {
        const draft = db.getDraft(userId);
        if (!draft) {
          await edit(ctx, '✉️ Draft tidak ditemukan. Buat ulang di ✉️ Buat Pesan.', kb.mainMenu());
          return answer();
        }
        if (db.get().campaigns.find(c => c.owner_id === String(userId) && c.status === 'running')) {
          await edit(ctx, '⚠️ Masih ada blast berjalan. Tunggu selesai atau tekan ⏹ STOP.', kb.backMain());
          return answer();
        }
        const campaign = db.createCampaign(userId, draft.type, draft.content, 'running');
        await edit(ctx, `🚀 <b>BLAST DIMULAI!</b>\n\n🆔 Campaign: <code>${campaign.id}</code>\n\nLaporan progres dikirim di chat ini.`, kb.backMain());
        runCampaign(bot, campaign.id).catch(e => console.error('Blast error:', e.message));
        return answer();
      }

      if (data === 'blast:cancel') {
        await edit(ctx, '❌ <b>Blast dibatalkan.</b>', kb.backMain());
        return answer();
      }

      if (data === 'blast:stop') {
        const camp = db.get().campaigns.find(c => c.owner_id === String(userId) && c.status === 'running');
        if (camp && stopCampaign(camp.id)) await answer('⏹ Menghentikan blast…');
        else await answer('Tidak ada blast berjalan');
        return;
      }

      /* ============ BUAT PESAN ============ */
      if (data === 'compose:text') {
        setState(userId, { action: 'compose_text' });
        await edit(ctx, `📝 <b>BUAT PESAN TEKS</b>\n\nKirim teks pesan sekarang (maks 4096 karakter).`, kb.composeMenu());
        return answer();
      }
      if (data === 'compose:photo') {
        setState(userId, { action: 'compose_photo' });
        await edit(ctx, `🖼 <b>BUAT PESAN FOTO</b>\n\nKirim <b>foto</b> (boleh dengan caption).\n⚠️ Kirim sebagai foto, bukan sebagai file/dokumen.`, kb.composeMenu());
        return answer();
      }
      if (data === 'compose:video') {
        setState(userId, { action: 'compose_video' });
        await edit(ctx, `🎞 <b>BUAT PESAN VIDEO</b>\n\nKirim <b>video</b> (boleh dengan caption).`, kb.composeMenu());
        return answer();
      }

      /* ============ JADWAL ============ */
      if (data === 'schedule:new') {
        if (!db.getDraft(userId)) {
          await edit(ctx, '✉️ <b>Belum ada pesan!</b>\nBuat dulu di ✉️ Buat Pesan, lalu jadwalkan.', kb.composeMenu());
          return answer();
        }
        setState(userId, { action: 'schedule_time' });
        await edit(ctx,
          `🕐 <b>JADWALKAN BLAST</b>\n\nKirim waktu dengan format:\n<code>YYYY-MM-DD HH:MM</code>\n\nContoh:\n<code>2025-06-30 19:00</code>\n\n⏰ Mengikuti waktu perangkatmu.`,
          kb.backMain());
        return answer();
      }

      if (data === 'schedule:list' || data.startsWith('sched:cancel:')) {
        if (data.startsWith('sched:cancel:')) {
          const ok = db.cancelScheduled(userId, data.split(':')[2]);
          await answer(ok ? 'Jadwal dibatalkan ✅' : 'Gagal / tidak ditemukan');
        }
        const list = db.scheduledCampaigns(userId);
        if (!list.length) {
          await edit(ctx, '🕐 <b>Tidak ada jadwal aktif.</b>', kb.scheduleMenu());
          return;
        }
        const buttons = list.map(c => [Markup.button.callback(`❌ ${fmtDate(c.scheduled_at)} · ${c.total} kontak`, `sched:cancel:${c.id}`)]);
        buttons.push([Markup.button.callback('⬅️ Kembali', 'menu:schedule')]);
        await edit(ctx, `📋 <b>JADWAL AKTIF</b> (${list.length})\n\nTekan tombol untuk membatalkan:`, Markup.inlineKeyboard(buttons));
        return;
      }

      /* ============ PAKET ============ */
      if (data.startsWith('plan:')) {
        const planName = data.split(':')[1];
        const p = cfg.PLANS[planName];
        if (!p) return answer();
        await edit(ctx,
          `💳 <b>${p.label}</b>\n\n📇 Maksimal kontak: <b>${p.maxContacts}</b>\n📡 Blast tanpa batas campaign\n🕐 Scheduler included\n\nTekan tombol untuk meminta aktivasi:`,
          Markup.inlineKeyboard([
            [Markup.button.callback('📩 Minta Aktivasi ke Admin', `planreq:${planName}`)],
            [Markup.button.callback('⬅️ Kembali', 'menu:plans')]
          ]));
        return answer();
      }

      if (data.startsWith('planreq:')) {
        const planName = data.split(':')[1];
        const p = cfg.PLANS[planName] || cfg.PLANS.free;
        for (const adminId of cfg.ADMIN_IDS) {
          bot.telegram.sendMessage(adminId,
            `📩 <b>PERMINTAAN UPGRADE</b>\n\n👤 ${esc(ctx.from.first_name)}${ctx.from.username ? ' (@' + esc(ctx.from.username) + ')' : ''}\n🆔 <code>${ctx.from.id}</code>\n💳 Paket: ${p.label}\n\nAktifkan dengan:\n/setplan ${ctx.from.id} ${planName}`,
            { parse_mode: 'HTML' }).catch(() => {});
        }
        await answer('Permintaan terkirim ✅');
        await edit(ctx, `📩 Permintaan upgrade ke <b>${p.label}</b> sudah dikirim ke admin.`, kb.backMain());
        return;
      }

      /* ============ OPT-OUT (dari penerima) ============ */
      if (data.startsWith('optout:')) {
        db.setOptOut(data.slice('optout:'.length), true);
        await answer('✅ Berhasil. Kamu tidak akan menerima pesan lagi dari bot ini.');
        return;
      }

      return answer();
    } catch (e) {
      console.error('Callback error:', e.message);
      return answer('⚠️ Terjadi kesalahan');
    }
  });
}

module.exports = { register };
```

## 📄 `src/handlers/messages.js`

```js
const db = require('../db');
const kb = require('../keyboards');
const { esc, truncate } = require('../helpers');
const { getState, clearState } = require('../state');

function register(bot) {
  bot.on('message', async (ctx) => {
    const state = getState(ctx.from.id);
    if (!state) return;
    const userId = ctx.from.id;

    // Jika user mengetik perintah saat dalam state, batalkan state
    if (ctx.message.text && ctx.message.text.startsWith('/')) {
      clearState(userId);
      return;
    }

    /* ----- Tambah via forward ----- */
    if (state.action === 'add_forward') {
      let fwd = ctx.message.forward_from || null;
      if (!fwd && ctx.message.forward_origin && ctx.message.forward_origin.sender_user) {
        fwd = ctx.message.forward_origin.sender_user;
      }
      if (!fwd || !fwd.id) {
        return ctx.reply('⚠️ Identitas pengirim pesan itu tersembunyi (privasi). Gunakan 🔗 Tautan Undangan atau kirim Chat ID-nya.');
      }
      const res = db.addContact(userId, fwd.id, fwd.first_name, fwd.username, '', 'forward');
      clearState(userId);
      if (res.ok) return ctx.reply(`✅ <b>Kontak ditambahkan!</b>\n\n👤 ${esc(fwd.first_name || '-')} <code>${fwd.id}</code>`, { parse_mode: 'HTML', reply_markup: kb.blastMenu().reply_markup });
      if (res.reason === 'exists') return ctx.reply('ℹ️ Kontak sudah ada di daftarmu.');
      return ctx.reply('❌ Kuota kontak paketmu penuh. Upgrade di 💳 Paket.');
    }

    /* ----- Tambah via chat ID ----- */
    if (state.action === 'add_chatid') {
      const id = parseInt((ctx.message.text || '').trim(), 10);
      if (!id) return ctx.reply('⚠️ Kirim angka chat ID saja. Contoh: 123456789');
      const res = db.addContact(userId, id, '', '', '', 'manual');
      clearState(userId);
      if (res.ok) return ctx.reply(`✅ <b>Kontak ditambahkan!</b>\n\n🆔 <code>${id}</code>`, { parse_mode: 'HTML', reply_markup: kb.blastMenu().reply_markup });
      if (res.reason === 'exists') return ctx.reply('ℹ️ Kontak sudah ada di daftarmu.');
      return ctx.reply('❌ Kuota kontak paketmu penuh. Upgrade di 💳 Paket.');
    }

    /* ----- Hapus kontak ----- */
    if (state.action === 'remove_contact') {
      const id = parseInt((ctx.message.text || '').trim(), 10);
      if (!id) return ctx.reply('⚠️ Kirim angka chat ID kontak yang mau dihapus.');
      const ok = db.removeContactByChatId(userId, id);
      clearState(userId);
      return ctx.reply(ok ? `🗑 Kontak <code>${id}</code> dihapus.` : '❌ Kontak tidak ditemukan di daftarmu.',
        { parse_mode: 'HTML', reply_markup: kb.blastMenu().reply_markup });
    }

    /* ----- Compose: text ----- */
    if (state.action === 'compose_text') {
      if (!ctx.message.text) return ctx.reply('⚠️ Kirim pesan berupa teks ya.');
      db.setDraft(userId, { type: 'text', content: { text: ctx.message.text }, updated_at: new Date().toISOString() });
      clearState(userId);
      return ctx.reply(`✅ <b>Pesan tersimpan sebagai draft!</b>\n\n📝 Preview:\n<i>${esc(truncate(ctx.message.text, 500))}</i>`,
        { parse_mode: 'HTML', reply_markup: kb.mainMenu().reply_markup });
    }

    /* ----- Compose: photo ----- */
    if (state.action === 'compose_photo') {
      if (!ctx.message.photo) return ctx.reply('⚠️ Kirim sebagai <b>foto</b> (bukan dokumen), boleh dengan caption.', { parse_mode: 'HTML' });
      const fileId = ctx.message.photo[ctx.message.photo.length - 1].file_id;
      db.setDraft(userId, { type: 'photo', content: { file_id: fileId, caption: ctx.message.caption || '' }, updated_at: new Date().toISOString() });
      clearState(userId);
      return ctx.reply('✅ <b>Foto + caption tersimpan!</b> Siap di-blast.', { parse_mode: 'HTML', reply_markup: kb.mainMenu().reply_markup });
    }

    /* ----- Compose: video ----- */
    if (state.action === 'compose_video') {
      if (!ctx.message.video) return ctx.reply('⚠️ Kirim <b>video</b>, boleh dengan caption.');
      db.setDraft(userId, { type: 'video', content: { file_id: ctx.message.video.file_id, caption: ctx.message.caption || '' }, updated_at: new Date().toISOString() });
      clearState(userId);
      return ctx.reply('✅ <b>Video + caption tersimpan!</b> Siap di-blast.', { parse_mode: 'HTML', reply_markup: kb.mainMenu().reply_markup });
    }

    /* ----- Jadwalkan ----- */
    if (state.action === 'schedule_time') {
      const m = (ctx.message.text || '').trim().match(/^(\d{4})-(\d{2})-(\d{2})\s+(\d{2}):(\d{2})$/);
      if (!m) return ctx.reply('⚠️ Format salah. Gunakan: YYYY-MM-DD HH:MM\nContoh: 2025-06-30 19:00');
      const dt = new Date(+m[1], +m[2] - 1, +m[3], +m[4], +m[5]);
      if (isNaN(dt.getTime()) || dt.getTime() <= Date.now()) return ctx.reply('⚠️ Waktu tidak valid atau sudah lewat.');
      const draft = db.getDraft(userId);
      if (!draft) { clearState(userId); return ctx.reply('✉️ Buat pesan dulu di ✉️ Buat Pesan.'); }
      const campaign = db.createCampaign(userId, draft.type, draft.content, 'scheduled', dt.toISOString());
      clearState(userId);
      return ctx.reply(
        `🕐 <b>Blast terjadwal!</b>\n\n` +
        `🆔 Campaign: <code>${campaign.id}</code>\n` +
        `📅 Waktu: ${dt.toLocaleString('id-ID')}\n` +
        `👥 Target: ${campaign.total} kontak\n\n` +
        `Cek/batalkan lewat: 🕐 Jadwal → 📋 Jadwal Aktif`,
        { parse_mode: 'HTML', reply_markup: kb.mainMenu().reply_markup });
    }
  });
}

module.exports = { register };
```

## 📄 `src/index.js`

```js
const { Telegraf } = require('telegraf');
const cfg = require('./config');
const db = require('./db');
const commands = require('./handlers/commands');
const callbacks = require('./handlers/callbacks');
const messages = require('./handlers/messages');
const scheduler = require('./services/scheduler');

if (!cfg.BOT_TOKEN || cfg.BOT_TOKEN.includes('DISINI')) {
  console.error('❌ BOT_TOKEN belum diisi! Edit file .env terlebih dulu.');
  process.exit(1);
}

const bot = new Telegraf(cfg.BOT_TOKEN);
db.load();

bot.use(async (ctx, next) => {
  try { if (ctx.from && !ctx.from.is_bot) db.upsertUser(ctx.from); } catch (_) {}
  return next();
});

commands.register(bot);
callbacks.register(bot);
messages.register(bot);

bot.catch((err) => console.error('⚠️ Bot error:', err.message || err));

bot.launch().then(() => {
  console.log('🤖 BLAST NOMOR sedang berjalan...');
  console.log('🛡 Admin:', cfg.ADMIN_IDS.length ? cfg.ADMIN_IDS.join(', ') : '(belum diatur — lihat ID-mu via /start)');
});

process.once('SIGINT', () => bot.stop('SIGINT'));
process.once('SIGTERM', () => bot.stop('SIGTERM'));

scheduler.start(bot);
```

---

## ✅ Alur Pemakaian

1. `/start` → menu utama muncul
2. **✉️ Buat Pesan** → pilih text/foto/video → kirim kontennya
3. **📱 Blast Nomor → ➕ Tambah Kontak** → bagikan **🔗 Tautan Undangan** (kontak masuk otomatis saat orang tekan START)
4. **📤 Mulai Blast** → konfirmasi → progres live + tombol **⏹ STOP**
5. **🕐 Jadwal** → format `2025-06-30 19:00` → jalan otomatis lewat cron
6. Setiap pesan blast menyertakan tombol **🚫 Berhenti Berlangganan** untuk penerima (opt-out otomatis juga aktif jika penerima block bot)

## ⚠️ Catatan Penting

- **Rate limit default 2,5 detik/pesan** — jangan diturunkan drastis; Telegram akan membatasi/ban bot yang spam.
- **Jangan blast ke orang yang tidak opt-in** — laporan spam dari user = bot dibanned permanen.
- Tetap seperti saran sebelumnya: untuk **WhatsApp, gunakan WhatsApp Business API resmi**, bukan otomatisasi akun pribadi.
- Data tersimpan di `data/database.json` — backup file ini secara berkala.

