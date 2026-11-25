# ARCHITECTURE — k8s-shopify-sync-pocharlies

Manifests de los CronJobs de `shopify-sync-app` (Shopify ↔ Picqer).

## Clientes y versiones
- Sin cliente: solo CronJobs en el namespace `skirmshop`. Tronco: `main` (Application `shopify-sync`, path `k8s`).

## Dependencias (ambos sentidos)
- Ejecuta la imagen `harbor.e-dani.com/homelab/shopify-sync-app` (código en `pocharlies-org/shopify-sync-app`), fijada por digest.
- Secrets por `externalsecret.yaml` (Vault + external-secrets). PVC (`pvc.yaml`) para el estado de sincronización de almacén.
- Lo consume: nadie; escribe en Shopify y Picqer.

## Stack
Kustomize plano (`k8s/kustomization.yaml`, namespace `skirmshop`). Sin Helm.

## Componentes compartidos
Ninguno propio; no usa la base del framework (no es app web).

## Cómo se construye
Un CronJob por job: `cronjob.yaml` (diario, `17 5 * * *`), `cronjob-weights.yaml` (`41 */6 * * *`), `cronjob-order-audit.yaml` (plantilla sin horario, SC-6595). Las decisiones de diseño están en `plans/sc-229-order-audit-exit-semantics.md`.
- Ficha `shopify-sync-warehouse` (SC-5888, A3 de SC-5806): el camino principal (Picqer→Shopify) va por evento — move del picker → `picqer.skirmshop.product.moved` → workflow synapse `skirmshop.k8s.shopify-sync-warehouse` → Job desde este CronJob con `SOLO_SKU` (script `sync-warehouse-locations.ts sync` en la imagen, mismo join y misma escritura atómica). El horario queda como red diaria declarada: sentido Shopify→Picqer (metafield de variante sin webhook) y eventos perdidos.
- Ficha `shopify-picqer-order-audit` (SKIRM-628, ley SC-5617): el camino principal va por evento — `shopify.skirmshop.orders.paid` → workflow synapse `skirmshop.k8s.shopify-picqer-order-audit` → Job desde este CronJob con `SOLO_ORDER=<ORDnnnnn>` (script `audit-shopify-picqer-orders.ts` en la imagen, mismo join, min-age esperada y relectura, mismas excepciones; el gauge SC-389 lo empuja solo el barrido). Desde SC-6595 (SC-5806 C-2) el CronJob es solo la plantilla (suspend, `0 0 31 2 *`): sin barrido diario, nadie cubre los eventos perdidos, los pedidos que nunca disparan `orders/paid` ni los de campaña con fulfillment order ON_HOLD (en el bus no hay `fulfillment_orders/open`), y el gauge SC-389 no se empuja, hasta el reloj único de la ola 2. Las excepciones del env llevan la fecha de su re-verificación.
- Imagen: la construye la CI del repo fuente `pocharlies-org/shopify-sync-app` (`release.yml` → reusable `reusable-release.yml` de `k8s-gitops-pocharlies`, runner `arc-k8s`: docker build + Trivy + cosign → `harbor.e-dani.com/homelab/shopify-sync-app`), disparada por tag `v*` o `workflow_dispatch` con `version` (p. ej. `k8s-20261116-skirm500-r2`). Aquí no hay `release.yml`: el pin por digest se actualiza editando `cronjob.yaml`/`weight-receiver.yaml` tras cada release.

## Tests y validaciones
`reusable-ci.yml` renderiza kustomize + kubeconform y yamllint. El código se prueba en el repo fuente.

## CI/CD y despliegue
`ci.yml`, `pr-review.yml`. Sin `release.yml` aquí: el pin de digest se actualiza editando los manifests (no verificado quién). ArgoCD sincroniza `main`.

## Decisiones y trampas
- Un CronJob fallido significa «la auditoría no se pudo ejecutar», nunca «pedidos desincronizados» (ver el ARCHITECTURE.md de `shopify-sync-app`).
