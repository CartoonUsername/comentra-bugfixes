# Claude Code Prompt — Comentra WMS Bug Fixes

Fixe folgende Bugs in der Comentra SaaS-Plattform (Hostinger VPS, Next.js, Supabase).
Alle Fixes in einer Session, Priorität von oben nach unten.

---

## Fix #1 — KRITISCH: autopilot_events.meta → payload (1-Zeilen-Fix)

**Problem:**
Der Live Activity Feed auf /wms crasht komplett weil der Code
`autopilot_events.meta` abfragt, die Spalte in der DB aber `payload` heißt.

**Fix:**
Suche alle Vorkommen von `autopilot_events` im Code und ersetze
jede Referenz auf `.meta` durch `.payload`.
Betrifft wahrscheinlich: WMS Dashboard, Activity Feed Komponente, API-Routen.

---

## Fix #2 — KRITISCH: Demo-Daten raus aus Pack / Restock / Workers

**Problem:**
Pack, Restock und Workers zeigen hardcodierte Demo-Daten (DEMO_PACK,
DEMO_RESTOCK, DEMO_WORKERS) statt echte Supabase-Daten.

**Fix:**
Finde alle DEMO_* Konstanten und ersetze sie durch echte Supabase-Queries:

Pack → query `pack_orders` + `pack_order_items` WHERE status = 'pending'
       JOIN product_variants für SKU + Name
       tenant_id immer mitfiltern

Restock → query `replenishment_tasks` WHERE status = 'pending'
          JOIN location_stock + storage_locations
          tenant_id immer mitfiltern

Workers → query `employees` WHERE status = 'active'
          JOIN aktuelle pick_orders / pack_orders per worker
          tenant_id immer mitfiltern

Wichtig: Leere States sauber behandeln (keine Demo-Daten als Fallback,
stattdessen "Keine offenen Aufgaben" anzeigen)

---

## Fix #3 — KRITISCH: Returns Worker Portal — leere Tasks

**Problem:**
getReturnsTaskSSR() gibt leere Tasks zurück obwohl Returns in der DB existieren.
Ursache: fehlendes `decision`-Feld-Handling.

**Fix:**
- Prüfe getReturnsTaskSSR() — wahrscheinlich filtert die Query auf
  `decision IS NOT NULL` oder einen bestimmten Status der Returns noch nicht haben
- Returns ohne `decision` Feld sollen auch angezeigt werden (status = 'pending')
- Query auf `returns` Tabelle: WHERE tenant_id = ? AND status IN ('pending', 'in_progress')
- decision-Feld: falls NULL → zeige "Entscheidung ausstehend" im UI

---

## Fix #4 — WICHTIG: Supabase Migration ausführen

**Problem:**
Die Datei supabase/migrations/20260326_amazon_wizard.sql wurde nie
auf der Live-DB ausgeführt. schema_cache und amazon_uploads Tabellen
fehlen — Amazon Wizard crasht.

**Fix:**
Führe die Migration aus:
```
npx supabase db push
```
Falls das nicht geht (Corporate Network Restrictions bekannt):
Lies den Inhalt von supabase/migrations/20260326_amazon_wizard.sql
und gib das SQL aus damit es manuell im Supabase Dashboard ausgeführt
werden kann.

---

## Fix #5 — WICHTIG: n8n URL aus Hardcode in ENV

**Problem:**
https://n8n.srv1299106.hstgr.cloud ist hardcoded in /wms/processes/page.tsx

**Fix:**
- Ersetze hardcodierte URL durch `process.env.NEXT_PUBLIC_N8N_BASE_URL`
- Füge NEXT_PUBLIC_N8N_BASE_URL=https://n8n.srv1299106.hstgr.cloud
  in .env.example + .env.local hinzu
- Prüfe ob weitere hardcodierte n8n URLs im Code existieren und ersetze alle

---

## Wichtige Hinweise
- tenant_id bei allen neuen Supabase-Queries Pflicht
- Keine Demo-Daten als Fallback — leere States mit sauberer UI
- Kommentare auf Deutsch
- Nach allen Fixes: einmal build prüfen (npm run build)
- Deployment: docker compose up --build -d auf Hostinger VPS
