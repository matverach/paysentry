# ADR-001: Mock Adapter para Procesador de Pagos en MVP

## Status

**ACCEPTED** (2026-01-31)

---

## Context

PaySentry necesita ejecutar transferencias bancarias a CBU/Alias como parte del flujo de autorización de pagos. Para el MVP, necesitamos decidir qué procesador de pagos usar.

**Fuerzas en juego:**

1. **Caso de uso principal del MVP:** Transferencias salientes (ej: pago de expensas a CBU del consorcio)
2. **Restricción de tiempo:** MVP en 8-12 semanas, proyecto paralelo
3. **Restricción de presupuesto:** $0-200/mes
4. **Objetivo del proyecto:** Aprendizaje arquitectónico + portfolio de SA, no producto productivo inmediato
5. **Requisito de diseño:** El sistema debe demostrar arquitectura sólida, no solo features funcionales

**Investigación realizada:**

### Opción 1: MercadoPago
- **Pros:**
  - Sandbox gratuito
  - Documentación en español
  - Familiaridad con el ecosistema argentino
- **Cons:**
  - ❌ **No soporta transferencias salientes via API**
  - Solo permite cobros (payments IN), no envío de dinero (payouts OUT)
  - API limitada a: crear pagos, guardar tarjetas, reembolsos

### Opción 2: BIND APIBank
- **Pros:**
  - ✅ Soporta transferencias salientes (POST /transfers)
  - API bancaria completa (cuentas, CBU/CVU, transferencias inmediatas)
  - Pensado para fintechs
- **Cons:**
  - ❌ Requiere proceso de onboarding (KYC, compliance)
  - Sin sandbox público instantáneo
  - Tiempo de aprobación desconocido (días/semanas)

**Problema:**
El caso de uso principal del MVP (transferencias automáticas) no es soportado por la opción más accesible (MercadoPago), y la opción que sí lo soporta (BIND) introduce un bloqueante de tiempo indeterminado.

---

## Decision

**Usaremos un Mock Adapter para el procesador de pagos durante el MVP.**

El sistema implementará **Hexagonal Architecture (Ports & Adapters)** con:

1. **Puerto (Interface):** `PaymentProcessorPort`
   ```python
   class PaymentProcessorPort(ABC):
       @abstractmethod
       def execute_transfer(self,
                           amount: Decimal,
                           destination_cbu: str,
                           idempotency_key: str) -> TransferResult:
           pass
   ```

2. **Adaptador Mock (MVP):** `MockPaymentAdapter`
   - Simula transferencias exitosas/fallidas basándose en reglas configurables
   - Persiste operaciones en DB local para audit trail
   - Genera IDs de transacción realistas
   - Simula latencias de red (100-300ms)

3. **Adaptador Real (Futuro):** `BindPaymentAdapter`
   - Se implementará cuando se complete onboarding con BIND
   - **Cero cambios en la lógica de negocio** al cambiar adaptadores

---

## Consequences

### Positivas

- ✅ **Desbloquea el MVP inmediatamente** - No dependemos de aprobaciones externas
- ✅ **Demuestra arquitectura hexagonal** - Concepto clave para SA (dependency inversion, testability)
- ✅ **100% controlable para testing** - Podemos simular fallos de red, timeouts, duplicados
- ✅ **Permite iterar rápido en lógica de negocio** - El core (políticas, autorizaciones, audit) se puede desarrollar sin esperar integraciones
- ✅ **Bajo costo** - $0 en fees de procesador durante desarrollo

### Negativas

- ⚠️ **No valida asunciones sobre la API real** - Podríamos descubrir problemas de integración tarde
- ⚠️ **Requiere documentar asunciones explícitamente** - El mock debe basarse en cómo *asumimos* que funciona BIND
- ⚠️ **Riesgo de mock demasiado simplista** - Si no modelamos correctamente fallos, el código real podría ser frágil

### Neutras

- El mock debe mantenerse actualizado si aprendemos más sobre la API real
- Cuando migremos a BIND, necesitaremos tests de integración exhaustivos

---

## Alternatives Considered

| Alternativa | Pros | Cons | Razón de Descarte |
|-------------|------|------|-------------------|
| **MercadoPago** | Sandbox inmediato, docs buenas | ❌ No soporta transferencias salientes | El caso de uso core no es posible |
| **BIND APIBank** | Soporta el caso de uso real | ❌ Onboarding bloquea inicio del MVP | Indeterminismo en timeline (incompatible con 8-12 semanas) |
| **Usar pagos IN en lugar de transferencias OUT** | Desbloquea MercadoPago | ❌ Cambia el caso de uso completamente (ya no es "pagar expensas", sería "cobrar al usuario") | Invalida la propuesta de valor del problema |
| **Mock Adapter (elegida)** | Desbloquea MVP, demuestra arquitectura | No valida integración real | ✅ Es la única que cumple: tiempo, scope, y objetivo de aprendizaje |

---

## References

- **DDIA Cap 1 - Maintainability:** "Good abstractions can help keep different parts of the system decoupled, making it easier to change one part without affecting another."
- **Hexagonal Architecture (Alistair Cockburn):** https://alistair.cockburn.us/hexagonal-architecture/
- **BIND APIBank Docs:** https://docs.bind.com.ar/reference/transferencias
- **MercadoPago API Reference:** https://www.mercadopago.com.ar/developers/es/reference

---

## Notes

### Asunciones sobre la API real de BIND (a validar)

El mock se diseñará asumiendo que BIND:
- Acepta `idempotency_key` para evitar duplicados
- Retorna estados: `pending`, `completed`, `failed`
- Valida CBU/CVU antes de ejecutar
- Puede fallar por: fondos insuficientes, CBU inválido, timeout
- Latencia típica: 200-500ms

### Criterios para migrar del Mock a BIND

Migraremos cuando:
1. El onboarding con BIND esté completo
2. La lógica core (políticas, autorizaciones, audit) esté estable
3. Tengamos tests de integración que validen el contrato del puerto

### Implementación del Mock

El `MockPaymentAdapter` debe:
- Persistir cada operación en tabla `mock_transfers` (para audit)
- Simular latencia random (100-300ms)
- Configurar tasa de fallos simulados (ej: 5% de transfers fallan con "timeout")
- Generar transaction_id realista (ej: `MOCK-TX-{timestamp}-{random}`)
- Loguear todas las operaciones como si fueran reales

Esto garantiza que el código de negocio se comporte igual con el mock y con BIND.
