[README.md](https://github.com/user-attachments/files/29610583/README.md)
# Pine — Restaurant Platform (POS · Handheld · KDS)

A full-stack restaurant operating system: one Node/Express + Socket.io
backend keeps a **POS terminal**, a **server handheld**, and a **Kitchen
Display System (KDS)** in perfect real-time sync. Fire an order from a
handheld and it lands on the kitchen screen instantly; bump a ticket in the
kitchen and every register updates the same second — no refresh, anywhere.

On top of the front-of-house apps, Pine ships a complete **back office**:
a transaction viewer with refunds, voids, and edits; gift cards and emailed
vouchers; card **batch settlements**; and a guest directory with addresses
and order history — all exposed as a REST API.

---

## 1. Quick start

```bash
npm install
npm start
```

The server starts on **http://localhost:4000** (set `PORT` to change it) and
prints links to every frontend:

| Frontend | URL | Designed for |
|---|---|---|
| **POS Terminal** | `/pos` | Register / manager station, large screen |
| **Handheld** | `/handheld` | Phones & tablets — servers on the floor |
| **Kitchen Display (KDS)** | `/kds` | Large TV/monitor mounted in the kitchen |
| **REST API** | `/api/…` | Full back-office API (see §5) |

First run seeds a realistic dataset for **The Riverside Bistro**: 8 staff, a
32-item menu, a 17-table floor plan, 15 inventory items, 7 guest profiles
(with addresses), ~185 historical transactions across 7 days, 6 closed
settlement batches, an open batch of today's card payments, 2 gift cards,
and 1 outstanding voucher — so every dashboard, report, and back-office
screen has real numbers on day one.

To wipe and reseed, stop the server, delete `server/data/db.json`, and
`npm start` again.

### Login PINs (demo)

| Name | Role | PIN |
|---|---|---|
| Maria Lopez | Manager | `1234` |
| James Chen | Server | `2222` |
| Aisha Patel | Server | `3333` |
| Tom Reilly | Cook | `4444` |
| Sara Kim | Host | `5555` |
| Diego Ruiz | Bartender | `6666` |
| Priya Nair | Server | `7777` |
| Owen Brooks | Cook | `8888` |

### No-install previews

Each app also has a standalone `preview.html`
(`public/{pos,handheld,kds}/preview.html`) with the theme, icons, and seed
data inlined — open it straight in a browser, no server needed. Regenerate
them after editing any app with:

```bash
node scripts/build-previews.js
```

---

## 2. Demoing real-time sync

Open three browser windows (or three devices on the same Wi-Fi pointed at
`http://<your-LAN-IP>:4000/…`):

1. **KDS** (`/kds`) on a big screen.
2. **Handheld** (`/handheld`) — log in as Aisha Patel (`3333`).
3. **POS Terminal** (`/pos`) — log in as Maria Lopez (`1234`).

Then:

- On the **handheld**: seat a table → add items (try a **Classic Burger**
  with modifiers) → *Send to Kitchen*. The ticket appears on the **KDS**
  within ~150 ms with a flash and an alert tone (tap the bell once first —
  browsers require a tap before audio).
- On the **KDS**: tap **Ready**, then **Bump All & Serve** — item statuses
  flip live on the POS and handheld.
- On the **POS**: toggle **Floor Edit** on the Tables page and drag a table
  to a new spot — the layout persists and syncs to every device.
- Refund a transaction in the back office and watch the numbers move
  everywhere.

Every device shows a **● Live** badge that turns red if the connection
drops, then reconnects automatically.

---

## 3. Design — "The Pass"

The interface is built from the material of a real restaurant: warm
ticket-paper surfaces, brass/rust/pine accents, Fraunces serif display type,
and monospace numerals wherever money or ticket numbers appear. Order
tickets read like printed chits, status labels like rubber stamps. The KDS
runs dark — real kitchens dim the line display — with a colored punch-rail
down each ticket's edge for urgency (green → amber at 8 min → red at 12).
Shared tokens live in `public/shared/theme.css`.

---

## 4. What's in each frontend

### POS Terminal (`/pos`)
The full app, role-gated by PIN:

- **Dashboard** — sales KPIs, 7-day trend, needs-attention queue, activity.
- **Tables** — a real spatial floor plan: tables positioned where they sit
  in the room, drag-to-rearrange in manager-only **Floor Edit** mode, with
  positions persisted and synced.
- **Order** — menu grid with modifiers, ticket panel, discounts, split
  checks, tip presets, card/cash payment.
- **Kitchen** — the ticket board inside the POS.
- **Menu / Inventory / Staff / Reports / Settings** — management suite,
  including a time clock and clock-out **tip-outs** (split-evenly + live
  allocation tracker).
- **Transactions** (back office — see §5).
- **Guests** — CRM with name, address, phone, email, loyalty, lifetime
  spend, and per-guest **order history**.

Role-based access: servers, cooks, and hosts land on a personal **My Sales**
view (their orders, tips, and tip-outs) and never see restaurant-wide
reports, inventory, payroll, or settings.

### Handheld (`/handheld`)
Mobile-first for servers and bartenders: table list, full menu + modifier
sheets, cart with live totals, *Send to Kitchen*, payment sheet, kitchen
view, My Sales, and a clock-out flow with department breakdown and
peer-to-peer tip-outs — including a "you were tipped out $X" toast the next
time the recipient logs in.

### Kitchen Display (`/kds`)
Station tabs with live counts (Cold Line / Grill / Pizza Oven / Pastry /
Bar), urgency-colored tickets, per-item **Ready** bumps, **Bump All &
Serve**, and a toggleable audio + flash alert for new tickets.

---

## 5. Back office & REST API

The **Transactions** page (Manager-only) has three tabs:

- **Transactions** — filterable viewer (date range, method, status, search).
  Click any transaction for the detail modal: **edit** tip / payment method
  (until settled), **refund** full or partial — including *refund as store
  credit*, which mints a gift card on the spot — **void** with an audit
  trail (blocked once settled: "use a refund instead"), attach a **guest**,
  send a **voucher**, or issue a **gift card**.
- **Gift Cards & Vouchers** — issue, redeem (balance-checked), deactivate;
  send one-time voucher codes by (simulated) email and track redemption.
- **Settlements** — the open batch of unsettled card transactions with a
  net-deposit summary (pending card refunds netted against gross), one-click
  **Close Batch & Settle**, and full batch history.

Every operation is also a REST endpoint. Mutations persist to disk and
broadcast to all connected devices over the same Socket.io channel the
frontends use — change something via `curl` and watch the POS update live.

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/api/state` | Full canonical shared state |
| `GET` | `/api/health` | Liveness + connected-client count |
| `POST` | `/api/login` | `{ pin }` → staff record, clocks in |
| `GET` | `/api/customers?q=` | List / search guests |
| `GET` | `/api/customers/:id` | Guest **with order history** |
| `POST` | `/api/customers` | Create guest (name, address, phone, email) |
| `PUT` | `/api/customers/:id` | Update guest |
| `DELETE` | `/api/customers/:id` | Delete guest (detaches from transactions) |
| `GET` | `/api/transactions` | List; filters: `from,to,method,status,server,customerId,settled` |
| `GET` | `/api/transactions/:id` | One transaction |
| `PUT` | `/api/transactions/:id` | Edit tip / method / guest (pre-settlement) |
| `POST` | `/api/transactions/:id/refund` | Full or partial refund |
| `POST` | `/api/transactions/:id/void` | Void (pre-settlement only) |
| `GET` | `/api/giftcards` | List gift cards |
| `POST` | `/api/giftcards` | Issue a card |
| `POST` | `/api/giftcards/:code/redeem` | Redeem (balance-checked) |
| `POST` | `/api/giftcards/:code/deactivate` | Deactivate |
| `GET` | `/api/vouchers` | List vouchers |
| `POST` | `/api/vouchers` | Create + "send" a voucher |
| `POST` | `/api/vouchers/:code/redeem` | One-time redemption |
| `GET` | `/api/settlements` | Open batch summary + history |
| `POST` | `/api/settlements/close` | Close the batch, settle everything |

**Settlement rules** mirror a real processor: cash never settles; card
transactions accumulate in the open batch; voids are only possible before
settlement (they drop out of the batch entirely); refunds on already-settled
transactions stay pending and net against the *next* batch; closing a batch
locks tips and stamps every transaction with its batch ID.

---

## 6. Architecture

```
pine/
├── server/
│   ├── index.js        ← Express + Socket.io, REST API, persistence
│   ├── seed.js         ← Seed data (menu, staff, tables, history, batches)
│   └── data/db.json    ← Canonical state on disk (auto-created, gitignored)
├── public/
│   ├── shared/
│   │   ├── sync.js     ← PineSync — real-time sync client for all 3 apps
│   │   ├── theme.css   ← "The Pass" design tokens & shared components
│   │   └── icons.js    ← Shared SVG icon set
│   ├── pos/            ← POS Terminal (index.html + standalone preview.html)
│   ├── handheld/       ← Handheld (index.html + preview.html)
│   └── kds/            ← Kitchen Display (index.html + preview.html)
└── scripts/
    └── build-previews.js  ← Regenerates the standalone previews
```

**How sync works.** The server holds one canonical JSON document — the whole
restaurant's live state — persisted to `db.json` (debounced 400 ms). Each
frontend keeps a local copy plus session-only fields (`currentStaff`, `view`)
that never leave the device.

1. On load, a frontend fetches `GET /api/state`.
2. On every action it updates local state immediately (the UI never waits),
   then `PineSync.push(state)` emits the shared slice over Socket.io.
3. The server merges, persists, and broadcasts to every other client.
4. Everyone else merges and re-renders — typically within ~150 ms.

Deliberately simple for a single location; see §7 for the production path.

---

## 7. Beyond the MVP

- **Diff-based sync** (send changed records, not the whole shared slice).
- **Server-side ID generation** to eliminate the theoretical collision when
  two devices create an order in the same instant.
- **A real database** (Postgres) with transactions, audit trails, backups.
- **Real payments** — Stripe Terminal / Connect instead of simulated charges.
- **Accounts & multi-tenancy** — real auth, org/location hierarchy, and
  role-based access enforced server-side rather than in the frontend.
- **Cloud hosting** with HTTPS and horizontal scaling.
- **Push notifications** to handhelds ("Table 5's food is up").
