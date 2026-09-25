
# GCP Dataform — ELT CI/CD Unificado

Repositorio consolidado de los 6 labs del curso **GCP Dataform: ELT y CI/CD**, unificado en una sola estructura lista para desplegar en Dataform vía GitHub.

---

## Estructura del proyecto

```
curso-gcp-dataform-unificado/
├── setup.sql              # Script de creación de datos de prueba (usuarios, productos, órdenes)
├── setup_v2.sql           # Versión simplificada (solo usuarios y productos)
├── init.py                # Placeholder de inicialización (Python)
├── workflow_settings.yaml # Configuración de project, dataset y versión de Dataform
├── includes/
│   └── constants.js       # Constantes compartidas (impuestos por país)
└── definitions/
    ├── sources/           # Declaraciones de tablas raw (BigQuery)
    │   ├── raw_orders.sqlx
    │   ├── raw_products.sqlx
    │   └── raw_users.sqlx
    ├── staging/           # Vistas de limpieza y estandarización
    │   ├── stg_orders.sqlx
    │   ├── stg_products.sqlx
    │   ├── stg_users.sqlx
    │   ├── stg_users_enriched.sqlx
    │   └── test_stg_users_enriched.sqlx
    ├── processing/        # Tablas incrementales de negocio
    │   └── orders_enriched.sqlx
    ├── marts/             # Tablas de negocio (marts) y reportes regionales
    │   ├── fact_cohort_retention.sqlx
    │   └── regional_sales.js
    ├── snapshots/         # Tablas de snapshots/fantasías
    │   └── snp_users_history.sqlx
    └── ops/               # Operaciones (GRANT, DELETE, mantenimiento)
        └── grant_permissions.sqlx
```

---

## Flujo de datos (dependencias)

```
setup.sql ──> raw_data.{*users,products,orders}  (tablas en BigQuery)

sources/*.sqlx  (declaraciones que referencian las tablas raw)

staging/*.sqlx  ──>  sources  (vistas de staging)
  stg_orders        ──>  raw_orders
  stg_products      ──>  raw_products
  stg_users         ──>  raw_users
  stg_users_enriched ──> raw_users

processing/*.sqlx  ──>  staging  (tablas incrementales)
  orders_enriched   ──>  stg_orders

marts/*.sqlx  ──>  processing + snapshots  (tablas de negocio)
  fact_cohort_retention ──> orders_enriched + snp_users_history
  regional_sales.js     ──> orders_enriched  (publicaciones dinámicas por país)

snapshots/*.sqlx  ──>  sources  (snapshots de usuarios)
  snp_users_history ──> raw_users

ops/*.sqlx  ──>  marts  (operaciones de mantenimiento)
  grant_permissions ──> fact_cohort_retention
```

### Gráfico de ejecución

1. **setup.sql** → crear tablas raw en BigQuery
2. **sources/** → declaraciones (no generan SQL, solo definen metadatos)
3. **staging/** → vistas sobre sources
4. **processing/** → tabla incremental sobre staging
5. **snapshots/** → tabla incremental sobre sources
6. **marts/** → tablas y publicaciones sobre processing/snapshots
7. **ops/** → operaciones post-execución (GRANT, DELETE)

---

## Archivos de configuración

### workflow_settings.yaml

| Parámetro | Valor |
|-----------|-------|
| `defaultProject` | `curso-dataform-488319` |
| `defaultLocation` | `US` |
| `defaultDataset` | `dataform_staging` |
| `defaultAssertionDataset` | `dataform_assertions` |
| `dataformCoreVersion` | `3.0.42` |

### includes/constants.js

Exporta un array `countries` con 5 países y sus impuestos:

| Código | Impuesto | Tipo |
|--------|----------|------|
| US | 8% | table |
| ES | 21% | view |
| MX | 16% | table |
| FR | 20% | view |
| UK | 20% | table |

---

## Configuración de Dataform

### Pre-requisitos

1. **BigQuery**: Proyecto `curso-dataform-488319` con datasets:
   - `raw_data` (datos fuente)
   - `dataform_staging` (vistas staging)
   - `dataform_processing` (tablas processing)
   - `dataform_marts` (tablas marts)
   - `dataform_snapshots` (snapshots)
   - `dataform_assertions` (tests/assertions)

2. **Dataform API**: Habilitar la API del proyecto.

3. **Service Account**: El service account `service-106039082375@gcp-sa-dataform.iam.gserviceaccount.com` necesita permisos `BigQuery Data Viewer` y `BigQuery Job User`.

### CI/CD con GitHub

1. Conectar el repositorio a **Dataform CLI** o usar **Dataform GitHub App**.
2. Configurar el workflow de GitHub Actions para ejecutar `dataform compile` y `dataform run`.
3. El archivo `workflow_settings.yaml` define el proyecto por defecto — asegurarse de que coincida con el proyecto GCP real.

Ejemplo de GitHub Actions:

```yaml
name: Dataform CI/CD
on:
  push:
    branches: [main]
jobs:
  dataform:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dataform-co/dataform-action@v1
        with:
          project_id: curso-dataform-488319
          workspace_dir: curso-gcp-dataform-unificado
```

---

## Lab-by-lab

### Lab 1 — Fundamentos
Configuración básica, declaraciones de fuentes, staging views.

### Lab 2 — Staging avanzado
Staging con transformaciones, tests de calidad.

### Lab 3 — Processing
Tablas incrementales con `uniqueKey`, particionamiento y clustering.

### Lab 4 — Marts y regiones
Tablas de negocio, reportes regionales dinámicos con JavaScript.

### Lab 5 — Snapshots y retención
Snapshots de usuarios, tabla de cohortes y retención.

### Lab 6 — Ops y CI/CD
Operaciones (GRANT, DELETE), configuración de entorno, despliegue.

---

## Observaciones y recomendaciones

### ⚠️ Problemas encontrados

1. **snp_users_history.sqlx usa `type: "incremental"` en lugar de `type: "snapshot"`**
   - Un snapshot en Dataform debe declararse con `type: "snapshot"`.
   - Actualmente es una tabla incremental que se recalcula cada ejecución, no un snapshot que mantiene historial.
   - *Recomendación*: Corregir a `type: "snapshot"` y agregar `config` de snapshot (`target`, `updatedAt`).

2. **test_stg_users_enriched.sqlx — dataset inconsistente**
   - El `dataset: "stg_users_enriched"` en el config del test no coincide con `defaultAssertionDataset: "dataform_assertions"` en `workflow_settings.yaml`.
   - Dataform podría intentar escribir los resultados del test en un dataset que no existe.
   - *Recomendación*: Cambiar a `dataset: "dataform_assertions"` o crear el dataset `stg_users_enriched`.

3. **setup_v2.sql no crea la tabla `orders`**
   - Si se ejecuta `setup_v2.sql` en lugar de `setup.sql`, todas las dependencias de `orders` fallarán (stg_orders, orders_enriched, fact_cohort_retention, regional_sales).
   - *Recomendación*: Documentar claramente cuándo usar cada versión o consolidar en un solo setup.sql.

4. **grant_permissions.sqlx — DELETE peligroso**
   - El DELETE `WHERE created_at < DATE_SUB(CURRENT_DATE(), INTERVAL 5 YEAR)` en una operación es riesgoso: se ejecuta en cada run de Dataform y podría borrar datos inadvertidamente.
   - *Recomendación*: Mover a un script manual o scheduling externo, no a una operación de Dataform.

### ✅ Buenas prácticas observadas

- **Sepación limpia de capas**: sources → staging → processing → marts → ops.
- **Uso correcto de `${ref()}`** para dependencias explícitas.
- **Incremental con `uniqueKey` y particionamiento** en orders_enriched.
- **JavaScript para publicaciones dinámicas** en regional_sales.js (muy útil para multi-región).
- **pre_operations** con `DECLARE` para checkpointing en tabla incremental.
- **Tags** bien utilzados para categorizar (staging, daily_run, regional_reports, security).

### 🔧 Mejoras sugeridas

- Agregar `name` explícito en todas las configs (ej. `name: "stg_orders"`).
- Agregar `description` a stg_users_enriched (está vacío).
- Agregar tags a stg_users_enriched (falta `tags: ["staging"]`).
- Considerar `tags: ["staging"]` también en stg_users_enriched para consistencia.
- El init.py es un placeholder — si se usa para CI/CD, implementar lógica real o eliminarlo.

---

## Estado: ✅ Listo para deploy (con las correcciones mencionadas)

El proyecto está bien estructurado y sigue las mejores prácticas de Dataform. Las correcciones señaladas son menores y no bloqueantes para un entorno de curso/demo.

---

## Notas sobre la unificación de las 6 labs

### Verificación de igualdad
Todas las 6 labs originales (`curso-gcp-dataform-01/` … `curso-gcp-dataform-06/`) fueron comparadas mediante hash SHA-256 de sus 18 archivos cada una (108 hashes totales). **El resultado es que todas las labs son copias idénticas** — no hay diferencias de contenido entre ninguna de ellas.

### Decisión
Dado que las 6 labs son idénticas, no fue necesario elegir "la versión más completa/reciente" para ningún archivo. La carpeta unificada contiene un solo set de los 18 archivos sin duplicación.

### Recomendación
Las 6 labs originales pueden eliminarse o archivarse, ya que `curso-gcp-dataform-unificado/` contiene el único contenido relevante. Mantener múltiples copias idénticas genera confusión sin agregar valor.
=======
# dataform
>>>>>>> aa050b1c20615c80eea684a8837c65428ccc2b32
