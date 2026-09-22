# Despliegue de Dataform vía GitHub Actions

## Pre-requisitos en GCP

1. **Proyecto BigQuery**: `curso-dataform-488319` (o el project ID real)
2. **Datasets necesarios**:
   - `raw_data` — datos fuente
   - `dataform_staging` — views staging
   - `dataform_processing` — tablas processing
   - `dataform_marts` — tablas marts
   - `dataform_snapshots` — snapshots
   - `dataform_assertions` — tests/assertions

3. **Habilitar Dataform API** en el proyecto

4. **Service Account** con permisos:
   - `BigQuery Data Viewer`
   - `BigQuery Job User`
   - `Dataform API Editor` (o el role equivalente)

## Configuración de GitHub Secrets

| Secret | Valor |
|--------|-------|
| `GCP_PROJECT_ID` | ID del proyecto GCP (ej: `curso-dataform-488319`) |
| `GCP_SA_KEY` | JSON de service account key (si usas Opción B) |
| `GCP_WORKLOAD_POOL_PROVIDER` | Resource name del provider WIF (si usas Opción A) |

## Flujo de ejecución

1. `git push` a `main` → trigger del workflow
2. `dataform compile` — valida la configuración (no requiere auth)
3. `dataform run` — ejecuta el pipeline (requiere auth GCP)

## Nota sobre Workload Identity Federation

Ver documento adjunto: `doc_google_identity.txt`

WIF permite que GitHub Actions acceda a GCP sin service account keys, usando federación OIDC. Es el enfoque más seguro.

Para configurar WIF necesitas:
1. Crear workload identity pool en GCP
2. Crear OIDC provider para GitHub
3. Otorgar permisos al pool/provider
4. Configurar los secrets en GitHub
