# k8s-shopify-sync-pocharlies

Disabled k8s skeleton for `sauvage:/home/ubuntu/skirmshop/shopify-sync-app`.

Current host cron:

```cron
17 */4 * * * cd /home/ubuntu/skirmshop/shopify-sync-app && /usr/bin/npx tsx scripts/sync-warehouse-locations.ts sync
```

This repo runs the k8s replacement for the former `sauvage` host cron.

The state PVC uses Longhorn, so the CronJob is pinned to an amd64 LAN node
rather than the `sauvage` edge node. The container reads state from
`WAREHOUSE_SYNC_STATE_FILE=/state/.warehouse-sync-state.json`.

Activation record:

1. Image built from `pocharlies-org/shopify-sync-app` and pushed to Harbor.
2. Shopify/Picqer secrets stored at `secret/skirmshop/shopify-sync`.
3. `shopify-sync-state` seeded from the host `.warehouse-sync-state.json`.
4. Manual Job `shopify-sync-manual-20260527045004` completed on 2026-05-27.
5. Host cron `warehouse_sync` removed after backup.

## Warehouse location sync (SC-6603, C-2 de SC-5806)

Desde el 25-11-2026 `shopify-sync-warehouse` **no se programa**: `suspend: true` y
`schedule: "0 0 31 2 *"` (31 de febrero). El manifiesto se queda en el repo porque
`synapse-adapter-k8s` crea el Job a partir de su `jobTemplate` (con `SOLO_SKU` inyectado)
y porque el `flock` de `/state` del comando sigue poniendo en fila el barrido manual y los
Jobs de evento. No hay red horaria (C-2, sin relojes): un evento perdido no se recupera hasta
que el SKU vuelva a tocarse, hasta el reloj único de la ola 2 de SC-5806, o hasta que alguien lance
el barrido a mano (`node dist/scripts/sync-warehouse-locations.js sync`).

Los disparadores, los tres por evento:

- `picqer.skirmshop.product.moved` — move en el picker (SC-5888).
- `picqer.skirmshop.product.changed` — edición de la ficha; desde SC-6603 `SOLO_SKU`
  también espeja el `productcode_supplier` de ese SKU.
- `eventos.shopify-sync.product_changed` — sentido Shopify→Picqer. Lo publica el receptor
  de pesos (`POST /eventos` del gateway, una clave por SKU del producto entregado) porque
  el webhook `products/update` es la única entrega que se ve siempre: el adaptador shopify
  de synapse descarta las entregas que no cambian facetas traducibles, y el metafield
  `custom.warehouse_location` es de variante.

Los dos primeros los lanza el workflow `skirmshop.k8s.shopify-sync-warehouse`; el tercero,
`skirmshop.k8s.shopify-sync-warehouse-shopify` (los eventos de `/eventos` llegan envueltos
por el outbox: `event.data.body.productcode`). El receptor necesita
`SYNAPSE_EVENTOS_URL` (fija en el Deployment) y `SYNAPSE_EVENTOS_SECRET` (clave del item
1Password `skirmshop-shopify-sync`, vía `envFrom`); sin la segunda no publica nada.

## Shopify → Picqer order audit

Main path is the event (SKIRM-628, SC-5617): `shopify.skirmshop.orders.paid`
→ synapse workflow `skirmshop.k8s.shopify-picqer-order-audit` → a Job created
from this CronJob's template with `SOLO_ORDER=<ORDnnnnn>`, which audits that
one order (waiting out the 15-minute minimum age, then re-reading it).

The CronJob itself is only the Job template since SC-6595 (SC-5806 C-2, no
clock-based nets): `suspend: true`, schedule `0 0 31 2 *`. The daily 02:30
sweep (last 14 days) is gone until the single clock of wave 2 exists, so lost
events, orders that never fire `orders/paid` (authorized/partially-paid/manual
captures) and campaign orders whose fulfillment order turns OPEN later are not
audited, and the SC-389 gauge is not pushed.

The audit detects:

- physical order lines without a SKU;
- SKUs that are not active Picqer product codes;
- missing Picqer webshop orders;
- webshop orders in `error`, `ignored`, `expected` or otherwise without an
  `idorder`.

An operational manual Picqer repair with an exact Shopify reference such as
`ORD18753` is accepted, so historical failed webshop rows do not alert forever.
The job is read-only: it never creates products/orders or changes inventory.
The exit code reflects whether the audit could run: an operational failure
exits non-zero and is handled by the cluster-wide `K8sCronJobFailed` alert and
cron log analyzer. Findings do NOT fail the Job — they are reported in the
`...finished` event (`findings: N`) and on stderr.

## Product weight by event (SKIRM-500)

`shopify-sync-weights` (every 6 h, whole-catalog sweep matched by exact
`productcode`) is gone: the weight now moves by event.

`shopify-sync-weight-receiver` (`k8s/weight-receiver.yaml`) runs
`dist/scripts/weight-server.js` from the same image and answers, under
`https://sauvage.e-dani.com/shopify-sync`:

- `POST /shopify-sync/webhooks/products` — Shopify's `products/create` and
  `products/update` (HMAC-verified with the app's API secret as
  `SHOPIFY_WEBHOOK_SECRET`, deduplicated by `X-Shopify-Webhook-Id`). It
  resyncs only that product.
- `GET /shopify-sync/v1/weights/<sku>` and `POST /shopify-sync/v1/weights`
  (`{"sku"}` / `{"skus":[…]}`) — bearer `WEIGHT_API_TOKEN`.

The two webhook subscriptions are created once in the Shopify Admin API
(`webhookSubscriptionCreate`, topics `PRODUCTS_CREATE` and `PRODUCTS_UPDATE`,
callback `https://sauvage.e-dani.com/shopify-sync/webhooks/products`) by the
app that owns the Admin token, `Pocharlies Catalog RAG Sync`; its API secret
key is the HMAC key. If the route changes, the subscriptions must be re-created
or the events stop arriving silently.

Both secrets come from the 1Password item `skirmshop-shopify-sync` through the
existing ExternalSecret (`refreshInterval: 24h`, force-sync to pick one up
sooner). The process refuses to start without them.
