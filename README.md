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

## Shopify → Picqer order audit

`shopify-picqer-order-audit` runs every day at 02:30 Europe/Madrid. It reads
the last 14 days of paid Shopify orders and verifies that each physical order
has become an operational Picqer order. The 15-minute minimum age avoids the
normal import race immediately after checkout.

The audit detects:

- physical order lines without a SKU;
- SKUs that are not active Picqer product codes;
- missing Picqer webshop orders;
- webshop orders in `error`, `ignored`, `expected` or otherwise without an
  `idorder`.

An operational manual Picqer repair with an exact Shopify reference such as
`ORD18753` is accepted, so historical failed webshop rows do not alert forever.
The job is read-only: it never creates products/orders or changes inventory.
Findings fail the Job and are handled by the cluster-wide
`K8sCronJobFailed` alert and cron log analyzer.
