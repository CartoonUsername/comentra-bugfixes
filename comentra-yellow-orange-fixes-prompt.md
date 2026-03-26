# Claude Code Prompt — Comentra Wichtige Fixes (Gelb + Orange)

Fixe folgende Features in der Comentra SaaS-Plattform.
Deployment: docker compose up --build -d auf Hostinger VPS. Kein Vercel.

---

## Fix #5 — Analytics: Echte DB-Queries statt hardcodierte Daten

**Problem:**
Heatmap und Cohort-Daten sind hardcodiert, keine echten DB-Queries.

**Fix:**

Heatmap (Lageraktivität nach Stunde/Tag):
- Query: SELECT DATE_TRUNC('hour', created_at) as hour, COUNT(*) as events
  FROM inventory_movements WHERE tenant_id = ? AND created_at > now() - interval '30 days'
  GROUP BY hour ORDER BY hour
- Rendere als 7x24 Heatmap-Grid (Wochentag × Stunde)

Cohort-Analyse (Kunden-Retention):
- Query: erste Bestellung pro Kunde + Folgebestellungen pro Monat
  SELECT DATE_TRUNC('month', first_order) as cohort,
  DATE_TRUNC('month', ordered_at) as period,
  COUNT(DISTINCT customer_id) as customers
  FROM orders JOIN (SELECT customer_id, MIN(ordered_at) as first_order
  FROM orders WHERE tenant_id = ? GROUP BY customer_id) first_orders USING (customer_id)
  WHERE tenant_id = ? GROUP BY cohort, period
- Rendere als Cohort-Tabelle mit Retention-Prozentsätzen

Leere States: "Keine Daten verfügbar" wenn DB leer, nie hardcodierte Werte.

---

## Fix #6 — Autopilot-Engine: Detection-Regeln implementieren

**Problem:**
Autopilot-Engine nur ~40% implementiert. Detection-Regeln sind Skelett.

**Fix:**
Implementiere folgende Detection-Regeln vollständig:

Regel 1 — Low Stock Alert:
- Query location_stock WHERE qty_available < reorder_point
- Erstelle autopilot_events (type: 'low_stock', payload: {sku, location, qty})
- Trigger replenishment_tasks INSERT

Regel 2 — Slow Mover Detection:
- Produkte die in den letzten 30 Tagen < 2x verkauft wurden
- Query order_items JOIN products, aggregiere pro SKU
- Erstelle autopilot_events (type: 'slow_mover', payload: {sku, last_sale, qty_on_hand})

Regel 3 — Überbestand:
- location_stock.qty_available > (avg_daily_sales * 90)
- Erstelle autopilot_events (type: 'overstock', payload: {sku, qty, days_of_stock})

Alle Regeln:
- Laufen via Cron (täglich 06:00)
- tenant_id immer mitfiltern
- Duplikate vermeiden: kein Event wenn identisches Event < 24h alt

---

## Fix #7 — Billing: Stripe Webhook + Commission-Logik

**Problem:**
Stripe-Webhook-Handler fehlt. Commission-Berechnung ist nur DB-Insert ohne Logik.

**Fix:**

Stripe Webhook (POST /api/billing/webhook):
- Verifiziere Stripe Signature (STRIPE_WEBHOOK_SECRET aus .env)
- Handle folgende Events:
  checkout.session.completed → subscription aktivieren
  customer.subscription.updated → Plan updaten in subscriptions Tabelle
  customer.subscription.deleted → subscription deaktivieren
  invoice.payment_failed → tenant benachrichtigen, Grace Period setzen

Commission-Berechnung:
- Wenn Order in Supabase eingeht: berechne marketplace_fee
  commission = order.total * (products_marketplace.marketplace_fee_pct / 100)
- INSERT in billing_usage_events (tenant_id, type: 'commission', amount, order_id)
- Monatliche Aggregation in billing_period_items

.env.example ergänzen:
STRIPE_SECRET_KEY=
STRIPE_WEBHOOK_SECRET=
STRIPE_PRICE_ID_BASIC=
STRIPE_PRICE_ID_PRO=

---

## Fix #8 — Shipping: DHL/Shipcloud Basis-Integration

**Problem:**
Shipping nur Stubs, nicht implementiert.

**Fix (Basis-Integration, kein Full-Feature):**

Shipcloud API Wrapper (services/shipping/shipcloud.ts):
- POST /shipments → Versandetikett erstellen
  Body: { carrier, from, to, package: {weight, length, width, height} }
  Response: { id, tracking_code, label_url }
- GET /shipments/:id → Tracking-Status abrufen

In Order-Flow einbauen:
- Wenn order.status → 'shipped': Shipcloud-Label erstellen
- tracking_code in shipments Tabelle speichern
- Label-URL in order.metadata speichern

.env.example ergänzen:
SHIPCLOUD_API_KEY=
SHIPCLOUD_SANDBOX=true

Wichtig: Nur Shipcloud als Abstraktionsschicht — DHL/UPS/DPD
werden über Shipcloud angebunden, nicht direkt.

---

## Fix #9 (Orange) — Team/CRM Backend

**Problem:**
Team-Bereich nur UI, kein Backend. Chat, Calls, Files funktionieren nicht.

**Fix (Basis-Backend, kein Full-Feature):**

Chat (team_messages, team_conversations):
- GET /api/team/conversations — Liste aller Conversations für Tenant
- GET /api/team/conversations/[id]/messages — Messages laden
- POST /api/team/conversations/[id]/messages — Message senden
- Supabase Realtime für Live-Updates nutzen (kein Websocket-Server nötig)

Files (team_files):
- POST /api/team/files/upload — Upload zu Supabase Storage
  Bucket: team-files/{tenant_id}/
- GET /api/team/files — Liste aller Files für Tenant
- Delete: Soft-Delete (deleted_at setzen)

Calls: Noch nicht implementieren — zu komplex für diese Session.
Stattdessen: Placeholder "Coming Soon" im UI.

---

## Wichtige Hinweise
- tenant_id bei allen Supabase-Queries Pflicht
- Kommentare auf Deutsch
- Kein TypeScript-Fehler (npm run build prüfen)
- Deployment: docker compose up --build -d auf Hostinger VPS
- Kein Vercel, kein Auto-Deploy
- Stripe + Shipcloud im Sandbox/Test-Modus lassen
