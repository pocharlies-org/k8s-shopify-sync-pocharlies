# k8s-shopify-sync-pocharlies

Disabled k8s skeleton for `sauvage:/home/ubuntu/skirmshop/shopify-sync-app`.

Current host cron:

```cron
17 */4 * * * cd /home/ubuntu/skirmshop/shopify-sync-app && /usr/bin/npx tsx scripts/sync-warehouse-locations.ts sync
```

This repo stages the k8s replacement as a suspended CronJob. It must stay
`suspend: true` until the image is built, a manual Job succeeds, and the host
cron is removed.

Activation steps:

1. Build and push `harbor.e-dani.com/homelab/shopify-sync-app:<tag>`.
2. Store required Shopify/Picqer secrets at `secret/skirmshop/shopify-sync`.
3. Replace the image digest.
4. Seed `shopify-sync-state` with the current host `.warehouse-sync-state.json`.
5. Run `kustomize build k8s`.
6. Create a manual Job from the CronJob and verify state in the PVC.
7. Remove the host cron, then set `suspend: false`.
