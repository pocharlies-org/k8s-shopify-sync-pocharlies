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
Un CronJob por job: `cronjob.yaml` (`17 */4 * * *`), `cronjob-weights.yaml` (`41 */6 * * *`), `cronjob-order-audit.yaml` (`30 2 * * *`). Las decisiones de diseño están en `plans/sc-229-order-audit-exit-semantics.md`.

## Tests y validaciones
`reusable-ci.yml` renderiza kustomize + kubeconform y yamllint. El código se prueba en el repo fuente.

## CI/CD y despliegue
`ci.yml`, `pr-review.yml`. Sin `release.yml` aquí: el pin de digest se actualiza editando los manifests (no verificado quién). ArgoCD sincroniza `main`.

## Decisiones y trampas
- Un CronJob fallido significa «la auditoría no se pudo ejecutar», nunca «pedidos desincronizados» (ver el ARCHITECTURE.md de `shopify-sync-app`).
