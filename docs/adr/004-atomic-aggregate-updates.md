# ADR-004: Atomic Aggregate Updates

## Estado

Aceptado

## Contexto

PaySentry necesita validar límites de gasto diario (`daily_amount_limit`, `max_daily_transactions`) antes de autorizar cada pago. Esto requiere conocer cuánto ha gastado un agente en el día actual.

Existen dos estrategias para obtener este dato:

1. **Cálculo on-the-fly**: `SELECT SUM(amount) FROM authorizations WHERE agent_id = ? AND date = TODAY`
2. **Agregado materializado**: Tabla `agent_daily_stats` con contadores pre-calculados

La primera opción es O(n) donde n = transacciones del día. Con volúmenes altos (miles de transacciones por agente por día), esto impacta latencia de validación.

La segunda opción es O(1) pero introduce riesgo de inconsistencia: si el agregado no se actualiza correctamente, las políticas se evalúan contra datos incorrectos.

## Decisión

Usaremos **agregados materializados con actualización atómica**.

El agregado (`agent_daily_stats`) se actualiza en la **misma transacción de base de datos** que crea la authorization. Si la actualización del agregado falla, toda la operación se rechaza.

```
BEGIN TRANSACTION
  INSERT INTO authorizations (...)
  INSERT INTO events (...)
  UPDATE agent_daily_stats SET
    daily_spent = daily_spent + amount,
    daily_count = daily_count + 1
  WHERE agent_id = ? AND date = TODAY
COMMIT
```

Si cualquier operación falla, se hace rollback completo y el pago no se autoriza.

## Alternativas Consideradas

### Cálculo on-the-fly
- **Pro**: Siempre consistente, sin duplicación de datos
- **Contra**: O(n) en cada validación, no escala con volumen

### Agregado con actualización eventual
- **Pro**: Mejor disponibilidad, operaciones independientes
- **Contra**: Ventana de inconsistencia donde políticas podrían evaluarse mal

### Agregado con reconciliación periódica
- **Pro**: Permite detectar y corregir desincronizaciones
- **Contra**: Complejidad operacional, no previene el problema inicial

## Consecuencias

### Positivas
- Validación de límites en O(1)
- Garantía de consistencia entre transacciones y agregados
- Mismo patrón que ADR-002 (consistency > availability)

### Negativas
- La base de datos se convierte en punto único de fallo (SPOF)
- Write amplification: cada authorization requiere múltiples escrituras
- Si la tabla `agent_daily_stats` no tiene row para el día actual, debe crearse (upsert)

### Neutrales
- Necesidad de job de limpieza para stats de días anteriores (o retención indefinida para histórico)

## Referencias

- DDIA Cap 3: Storage and Retrieval (materialized views)
- DDIA Cap 7: Transactions (atomicity)
- ADR-002: Transactional Event Log (mismo patrón aplicado)
