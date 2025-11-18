# Algoritmo de Creación de Pool de Proveedores

## Resumen

El sistema utiliza un algoritmo inteligente de matching basado en tags para generar automáticamente el pool de proveedores para licitaciones públicas. El algoritmo considera la compatibilidad de tags entre la licitación y los proveedores, incluyendo relaciones jerárquicas de tags.

---

## Estructura de Base de Datos

### Tabla de Configuración

```sql
CREATE TABLE business.vendor_pool_config (
    id integer NOT NULL PRIMARY KEY,
    threshold numeric NOT NULL, -- umbral normalizado (ej. 0.5 = 50%)
    min_pool_size integer NOT NULL, -- tamaño mínimo de pool (ej. 5)
    hierarchy_factor numeric NOT NULL, -- factor para match jerárquico de tags (ej. 0.5)
    effective_from timestamp without time zone NOT NULL, -- cuándo entra en vigencia esta config
    created_at timestamp without time zone NOT NULL,
    updated_at timestamp without time zone DEFAULT CURRENT_TIMESTAMP NOT NULL,
    CONSTRAINT valid_hierarchy CHECK ((hierarchy_factor >= 0 AND hierarchy_factor <= 1)),
    CONSTRAINT valid_threshold CHECK ((threshold >= 0 AND threshold <= 1))
);
```

**Nota:** La tabla incluye validaciones para asegurar que `threshold` y `hierarchy_factor` estén entre 0 y 1.

### Tabla de Pool de Proveedores

```sql
CREATE TABLE business.vendor_pool (
    tender_id uuid NOT NULL,
    vendor_id uuid NOT NULL,
    raw_score numeric DEFAULT 0 NOT NULL,
    normalized_score numeric DEFAULT 0 NOT NULL,
    matched_base_tags integer DEFAULT 5 NOT NULL,
    PRIMARY KEY (tender_id, vendor_id)
);
```

---

## Implementación en Rust

### Modelo de Configuración

```rust
use sqlx::FromRow;
use rust_decimal::Decimal;

#[derive(Debug, FromRow)]
pub struct VendorPoolConfig {
    pub threshold: Decimal,
    pub min_pool_size: i32,
    pub hierarchy_factor: Decimal,
}
```

### Repositorio: Cargar Configuración Vigente

```rust
impl TenderRepository {
    // obtiene la configuración activa del algoritmo de pool de proveedores
    pub async fn fetch_vendor_pool_config<'e, E: sqlx::Executor<'e, Database = Postgres>>(
        executor: E,
    ) -> Result<VendorPoolConfig, sqlx::Error> {
        let row = sqlx::query_as::<_, VendorPoolConfig>(
            r#"
            SELECT threshold, min_pool_size, hierarchy_factor
            FROM business.vendor_pool_config
            ORDER BY effective_from DESC
            LIMIT 1
        "#,
        )
        .fetch_one(executor)
        .await?;

        Ok(row)
    }
}
```

**Comportamiento:** Obtiene la configuración más reciente según `effective_from`, permitiendo cambios históricos de configuración.

### Repositorio: Generar Pool en Transacción

```rust
impl TenderRepository {
    // genera el pool de proveedores para una licitación usando el algoritmo de matching por tags
    pub async fn generate_vendor_pool(
        tx: &mut Transaction<'_, Postgres>,
        tender_id: Uuid,
    ) -> Result<(), sqlx::Error> {
        // obtiene la configuración activa del algoritmo
        let config = Self::fetch_vendor_pool_config(tx.as_mut()).await?;

        // ejecuta el cte recursivo que calcula scores y genera el pool
        let query = r#"
      WITH RECURSIVE tender_tag_hierarchy AS (
        -- termino base no recursivo
        SELECT tt.tag_id,
            tt.weight::NUMERIC AS tender_weight,
            t.parent_tag_id,
            1 AS depth,
            'base'::TEXT AS relation_type
        FROM business.tender_tags tt
            JOIN business.tags t ON tt.tag_id = t.id
        WHERE tt.tender_id = $1
        UNION ALL
        -- termino recursivo simple para ambos tags padres e hijos
        SELECT COALESCE(tp.id, tc.id) AS tag_id,
            -- resuelve tags padre/hijo
            CASE
                WHEN tp.id IS NOT NULL THEN th.tender_weight * $4 -- aplica el factor a los tag padre
                ELSE th.tender_weight -- mantiene el peso original para los tag hijos
            END::NUMERIC,
            COALESCE(tp.parent_tag_id, tc.parent_tag_id),
            th.depth + 1,
            CASE
                WHEN tp.id IS NOT NULL THEN 'parent'::TEXT
                ELSE 'child'::TEXT
            END
        FROM tender_tag_hierarchy th
            LEFT JOIN business.tags tp -- expansion de tag padres
            ON th.parent_tag_id = tp.id
            AND th.depth < 3
            LEFT JOIN business.tags tc -- expansion de tag hijos
            ON tc.parent_tag_id = th.tag_id
            AND th.depth < 3
        WHERE (
            tp.id IS NOT NULL
            OR tc.id IS NOT NULL
        )
      ),
      vendor_scores AS ( -- puntajes de los proveedores acorde a los tags
        SELECT vt.vendor_id,
            SUM(vt.weight * th.tender_weight) AS raw_score,
            COUNT(
                DISTINCT CASE
                    WHEN th.depth = 1 THEN th.tag_id
                END
            ) AS matched_base_tags
        FROM business.vendor_tags vt
        JOIN tender_tag_hierarchy th ON vt.tag_id = th.tag_id
        GROUP BY vt.vendor_id
      ),
      score_analysis AS ( -- analysis del puntaje
        SELECT GREATEST(MAX(raw_score), 1) AS max_score,
            COUNT(*) AS total_vendors
        FROM vendor_scores
        WHERE raw_score > 0
      ),
      normalized_scores AS ( -- puntajes normalizados
        SELECT vs.vendor_id,
            vs.raw_score,
            vs.raw_score / sa.max_score AS normalized_score,
            vs.matched_base_tags
        FROM vendor_scores vs
        CROSS JOIN score_analysis sa
      ),
      qualified_vendors AS ( -- proveedores calificados
        SELECT vendor_id,
            raw_score,
            normalized_score,
            matched_base_tags
        FROM normalized_scores
        WHERE normalized_score >= $2
        ORDER BY normalized_score DESC,
            matched_base_tags DESC,
            raw_score DESC
      ),
      fallback_vendors AS ( -- proveedores "faltantes"
        SELECT vendor_id,
            raw_score,
            normalized_score,
            matched_base_tags
        FROM normalized_scores
        WHERE vendor_id NOT IN (
            SELECT vendor_id
            FROM qualified_vendors
        )
        ORDER BY normalized_score DESC NULLS LAST,
            matched_base_tags DESC,
            raw_score DESC
        LIMIT GREATEST(
            0, $3 - (
                SELECT COUNT(*)
                FROM qualified_vendors
            )
        )
      )
      -- inserto en la pool
      INSERT INTO business.vendor_pool (
          tender_id,
          vendor_id,
          raw_score,
          normalized_score,
          matched_base_tags
      )
      SELECT $1,
          vendor_id,
          raw_score,
          normalized_score,
          matched_base_tags
      FROM (
          SELECT *
          FROM qualified_vendors
          UNION ALL
          SELECT *
          FROM fallback_vendors
      ) candidates;
      "#;

        sqlx::query(query)
            .bind(tender_id)
            .bind(config.threshold.to_f64().unwrap())
            .bind(config.min_pool_size)
            .bind(config.hierarchy_factor.to_f64().unwrap())
            .execute(tx.as_mut())
            .await?;

        Ok(())
    }
}
```

**Parámetros del Query:**
- `$1`: `tender_id` - ID de la licitación
- `$2`: `threshold` - Umbral de compatibilidad (ej: 0.5)
- `$3`: `min_pool_size` - Tamaño mínimo del pool (ej: 5)
- `$4`: `hierarchy_factor` - Factor jerárquico (ej: 0.5)

---

## Integración en el Servicio

### Creación de Licitación

El algoritmo se ejecuta automáticamente cuando se crea una licitación con `vendor_pool_type = Public`:

```rust
pub async fn create_tender(
    pool: &PgPool,
    buyer_id: Uuid,
    input: CreateTenderInput,
) -> Result<Tender, String> {
    let mut tx = begin_transaction(pool).await?;

    // crea la licitación
    let tender = TenderRepository::create_tender(&mut tx, buyer_id, &input).await?;

    // asocia los tags a la licitación
    TenderRepository::insert_tender_tags(&mut tx, tender.id, &input.tags).await?;

    // genera el pool automáticamente si es público
    if tender.vendor_pool_type == VendorPoolType::Public {
        TenderRepository::generate_vendor_pool(&mut tx, tender.id).await?;
    }

    // inserta estado inicial
    TenderRepository::insert_tender_status_history(
        &mut tx,
        tender.id,
        None,
        TenderStatus::Open,
        buyer_id,
        Some("Creación inicial".to_string()),
    ).await?;

    commit_transaction(tx).await?;
    Ok(tender)
}
```

### Actualización de Licitación

El pool también se regenera automáticamente cuando se actualiza una licitación pública y cambian los tags:

```rust
// si cambian los tags y es pool público, regenera el pool
if let Some(new_tags) = update.tags {
    if tender.vendor_pool_type == VendorPoolType::Public {
        // elimina el pool anterior
        sqlx::query!("DELETE FROM business.vendor_pool WHERE tender_id = $1", tender_id)
            .execute(tx.as_mut()).await?;
        
        // regenera con los nuevos tags
        TenderRepository::generate_vendor_pool(tx, tender_id).await?;
    }
}
```

---

## Funcionalidades Adicionales

### Pool Manual (Fixed)

Para licitaciones con `vendor_pool_type = Fixed`, los proveedores se agregan manualmente:

```rust
// inserta proveedores manualmente en el pool (pool tipo fixed)
pub async fn insert_manual_vendor_pool(
    tx: &mut Transaction<'_, Postgres>,
    tender_id: Uuid,
    vendor_ids: &[Uuid],
) -> Result<(), sqlx::Error> {
    for &vendor_id in vendor_ids {
        sqlx::query!(
            r#"INSERT INTO business.vendor_pool (tender_id, vendor_id, raw_score, normalized_score, matched_base_tags)
               VALUES ($1, $2, 1, 1, 0)"#,
            tender_id,
            vendor_id
        )
        .execute(tx.as_mut())
        .await?;
    }
    Ok(())
}
```

### Agregar Proveedores a Pool Existente

```rust
// agrega proveedores al pool de una licitación existente
pub async fn add_vendors_to_tender(
    tx: &mut Transaction<'_, Postgres>,
    tender_id: Uuid,
    vendor_ids: &[Uuid],
) -> Result<(), sqlx::Error> {
    for &vendor_id in vendor_ids {
        sqlx::query!(
            r#"INSERT INTO business.vendor_pool (tender_id, vendor_id, raw_score, normalized_score, matched_base_tags)
               VALUES ($1, $2, 1, 1, 0)
               ON CONFLICT (tender_id, vendor_id) DO NOTHING"#,
            tender_id,
            vendor_id
        )
        .execute(tx.as_mut())
        .await?;
    }
    Ok(())
}
```

### Eliminar Proveedores del Pool

```rust
// elimina proveedores del pool de una licitación
pub async fn remove_vendors_from_tender(
    tx: &mut Transaction<'_, Postgres>,
    tender_id: Uuid,
    vendor_ids: &[Uuid],
) -> Result<u64, sqlx::Error> {
    // implementación que elimina proveedores y retorna cantidad eliminada
}
```

### Obtener Pool de Proveedores

```rust
// obtiene el pool de proveedores para una licitación con sus scores
pub async fn get_vendor_pool(
    pool: &PgPool,
    tender_id: Uuid,
) -> Result<Vec<VendorPoolInput>, sqlx::Error> {
    // retorna lista de proveedores con sus scores y estadísticas
}
```

---

## Resumen del Funcionamiento del Algoritmo

### Paso 1: Recolectar los Datos

- **`tender_tags` (tt)**: Los tags que el comprador eligió para su licitación, con su peso (`weight`).
- **`vendor_tags` (vt)**: Los tags de cada proveedor, con su peso (`weight`) y su `parent_tag_id` para jerarquía.

### Paso 2: Expansión Jerárquica de Tags

El CTE recursivo `tender_tag_hierarchy` expande los tags de la licitación considerando relaciones padre/hijo:

- **Tags base**: Los tags directamente seleccionados para la licitación (depth = 1)
- **Tags padres**: Se expanden hacia arriba en la jerarquía con peso reducido por `hierarchy_factor`
- **Tags hijos**: Se expanden hacia abajo manteniendo el peso original
- **Límite de profundidad**: Máximo 3 niveles (`depth < 3`)

### Paso 3: Cálculo de Scores Brutos

En `vendor_scores`:
- Se hace `JOIN` entre `vendor_tags` y `tender_tag_hierarchy`
- Se calcula: `raw_score = Σ(vendor_tag.weight × tender_tag.expanded_weight)`
- Se cuenta: `matched_base_tags` = cantidad de tags base que coinciden

### Paso 4: Análisis y Normalización

En `score_analysis`:
- Se calcula `max_score = MAX(raw_score)` (el mejor score encontrado)
- Se usa `GREATEST(MAX(raw_score), 1)` para evitar división por cero

En `normalized_scores`:
- Se calcula: `normalized_score = raw_score / max_score`
- Resultado: valor entre 0 (sin coincidencias) y 1 (coincidencia perfecta)

### Paso 5: Filtrado por Umbral

En `qualified_vendors`:
- Se seleccionan proveedores con `normalized_score >= threshold`
- Ordenados por: `normalized_score DESC`, `matched_base_tags DESC`, `raw_score DESC`

### Paso 6: Fallback para Garantizar Cobertura

En `fallback_vendors`:
- Si no se alcanza `min_pool_size` con proveedores calificados
- Se seleccionan los mejores proveedores restantes (incluso con `normalized_score < threshold`)
- Se completa hasta alcanzar `min_pool_size`

### Paso 7: Inserción del Pool

- Se insertan en `business.vendor_pool` todos los proveedores de `qualified_vendors` + `fallback_vendors`
- Se guardan: `tender_id`, `vendor_id`, `raw_score`, `normalized_score`, `matched_base_tags`
- Todo dentro de una transacción para garantizar atomicidad

---

##  Ventajas del Algoritmo

- **Transparencia**: Todo queda registrado en la DB, con la configuración explícita
- **Flexibilidad**: Se puede ajustar `threshold`, `min_pool_size` y `hierarchy_factor` modificando solo la tabla de configuración
- **Equilibrio**: Combina cobertura (fallback) con relevancia (qualified + threshold)
- **Mantenibilidad**: La lógica está centralizada en el backend
- **Escalabilidad**: Usa CTEs recursivos eficientes de PostgreSQL
- **Auditabilidad**: Los scores se persisten para análisis histórico

---

## Explicación Detallada de la Configuración

### 1. `threshold` (Umbral de Compatibilidad)

**Valor típico:** `0.5` (50%)

**Propósito:** Define el nivel mínimo de compatibilidad requerido para que un proveedor sea considerado *calificado automáticamente*.

**Cómo funciona:**
- Proveedores con `normalized_score >= threshold` se incluyen en `qualified_vendors`
- Ejemplo: Si `threshold = 0.5` y un proveedor tiene 60% de compatibilidad (`0.6`), se incluirá directamente
- Proveedores con `normalized_score < threshold` solo se incluyen si es necesario para alcanzar `min_pool_size`

**Recomendación:** Valores entre `0.3` (más inclusivo) y `0.7` (más restrictivo)

---

### 2. `min_pool_size` (Tamaño Mínimo del Pool)

**Valor típico:** `5`

**Propósito:** Garantiza un número mínimo de proveedores en el pool, incluso si no hay suficientes calificados.

**Cómo funciona:**
- Si solo 2 proveedores alcanzan el `threshold`, el algoritmo añadirá 3 más de `fallback_vendors` (los mejores no calificados)
- Estos proveedores adicionales pueden tener `normalized_score < threshold` o incluso `0`
- El algoritmo siempre intenta alcanzar este mínimo, priorizando calidad sobre cantidad

**Recomendación:** Valores entre `3` (pequeñas licitaciones) y `10` (licitaciones grandes)

---

### 3. `hierarchy_factor` (Factor Jerárquico)

**Valor típico:** `0.5`

**Propósito:** Controla cómo afectan las relaciones jerárquicas de tags a la compatibilidad.

**Cómo funciona:**
- **Tags padres:** Reduce progresivamente el peso al ascender en la jerarquía
  - Ejemplo: Si un tag de la licitación es padre de otro, el peso del tag padre se multiplica por `hierarchy_factor`
  - En cada nivel superior, el peso se reduce: `peso × hierarchy_factor^nivel`
- **Tags hijos:** Mantiene el peso original del tag de la licitación
- **Límite de profundidad:** Máximo 3 niveles (`depth < 3`)

**Ejemplo:**
- Tag base (depth=1): peso = 100
- Tag padre nivel 1 (depth=2): peso = 100 × 0.5 = 50
- Tag padre nivel 2 (depth=3): peso = 50 × 0.5 = 25

**Recomendación:** Valores entre `0.3` (más permisivo) y `0.7` (más restrictivo)

---

## Ejemplo Práctico

### Configuración:
```json
{
  "threshold": 0.5,
  "min_pool_size": 5,
  "hierarchy_factor": 0.5
}
```

### Escenario:
- Licitación con 3 tags base
- 10 proveedores en el sistema
- 2 proveedores tienen `normalized_score >= 0.5` (calificados)
- 8 proveedores tienen `normalized_score < 0.5` (no calificados)

### Resultado:
1. **Fase 1 (Qualified):** Se incluyen los 2 proveedores calificados
2. **Fase 2 (Fallback):** Se añaden los 3 mejores no calificados para alcanzar `min_pool_size = 5`
3. **Pool final:** 5 proveedores (2 calificados + 3 fallback)

---

## Casos de Uso

### Licitación Especializada (Alto Threshold)
```sql
INSERT INTO business.vendor_pool_config (threshold, min_pool_size, hierarchy_factor, effective_from)
VALUES (0.7, 3, 0.3, NOW());
```
- Requiere alta compatibilidad (70%)
- Pool pequeño (3 proveedores)
- Expansión jerárquica más permisiva (0.3)

### Licitación General (Bajo Threshold)
```sql
INSERT INTO business.vendor_pool_config (threshold, min_pool_size, hierarchy_factor, effective_from)
VALUES (0.3, 10, 0.6, NOW());
```
- Requiere compatibilidad moderada (30%)
- Pool grande (10 proveedores)
- Expansión jerárquica más restrictiva (0.6)

---

## Notas de Implementación

- El algoritmo se ejecuta **dentro de una transacción** para garantizar atomicidad
- Los scores se **persisten** en `business.vendor_pool` para análisis posterior
- La configuración se obtiene **dinámicamente** en cada ejecución (permite cambios sin reiniciar)
- El algoritmo es **idempotente**: regenerar el pool produce el mismo resultado con la misma configuración
- Se usa **CTE recursivo** de PostgreSQL para eficiencia en la expansión jerárquica