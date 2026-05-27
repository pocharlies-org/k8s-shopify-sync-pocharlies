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
