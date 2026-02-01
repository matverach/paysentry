# PaySentry - Problem Statement

---

## 1. Contexto del Mercado

Los agentes de IA autónomos están pasando de experimentales a producción. OpenAI lanzó Operator (enero 2025), Google está desarrollando Project Jarvis, y Visa anunció Intelligent Commerce para pagos autónomos.

El cambio fundamental: **los agentes ya no solo consultan información, ahora ejecutan acciones** — incluyendo transacciones financieras.

**¿Por qué AHORA?**
- Los agentes IA tienen suficiente fiabilidad para tareas específicas
- Las fintechs buscan diferenciación con features "AI-powered"
- Los usuarios quieren automatización, pero no al costo de perder control

El problema es que **la infraestructura de autorización no existe**. OAuth resolvió "delegar acceso a datos". Nadie resolvió "delegar acceso a dinero" de forma granular y auditable.

---

## 2. El Problema Específico

### ¿Qué problema existe HOY?

**Los usuarios quieren delegar pagos a agentes IA, pero no confían en darles acceso total a sus fondos.**

### ¿Quién lo sufre?

1. **Fintechs y wallets** que quieren ofrecer "AI assistant para pagos"
   - No tienen infraestructura de autorización granular
   - Risk/Compliance no aprueba dar acceso total a un agente

2. **Usuarios finales** que ven valor en automatización pero:
   - No confían en que el agente no se equivoque
   - Quieren límites: "solo gastá hasta $X", "solo en merchant Y"
   - Necesitan poder revocar acceso inmediatamente

3. **Desarrolladores de agentes IA** que:
   - No tienen un estándar para "pedir permiso para pagar"
   - Cada integración es custom y sin trazabilidad

### ¿Cuánto les duele?

- **Oportunidad perdida:** Fintechs no pueden lanzar features de agentes IA → pierden competitividad
- **Riesgo regulatorio:** Dar acceso sin límites → problemas de compliance
- **Fricción de usuario:** Cada pago requiere confirmación manual → mata la propuesta de valor de "autónomo"

### ¿Cómo lo resuelven hoy?

**Workaround 1:** Dar credenciales completas al agente
- Problema: Sin límites, sin audit trail, sin revocación granular

**Workaround 2:** Confirmación manual en cada transacción
- Problema: Mata la autonomía, alta fricción

**Workaround 3:** No ofrecer la funcionalidad
- Problema: Pierden market share vs competidores que sí innovan

**¿Por qué son insuficientes?**
Porque nadie resolvió el problema de **confianza programable**. Queremos automatizar, pero con guardrails.

---

## 3. Propuesta de Solución

**PaySentry es un authorization gateway que permite a fintechs y wallets delegar pagos y transferencias a agentes IA con políticas granulares y trazabilidad completa, a diferencia de dar acceso directo a tarjetas que no tiene límites configurables ni audit trail.**

**Analogía:** OAuth, pero para dinero.

---

## 4. Actores del Sistema

### Actor 1: Usuario Final (owner del dinero)

**¿Quién es?**
Una persona que usa una fintech/wallet y quiere automatizar pagos recurrentes o delegarlos a un asistente IA.

**¿Qué necesita del sistema?**
- Definir políticas de gasto para el agente ("máximo $50k/mes", "solo en Edenor", "transferencias solo a mi CBU de ahorro")
- Ver historial completo de qué hizo el agente
- Revocar permisos inmediatamente si algo sale mal
- Aprobar manualmente transacciones que excedan ciertos thresholds

**¿Qué problema le resuelve PaySentry?**
Le da **control y visibilidad**. Puede automatizar sin miedo a que el agente se descontrole.

### Actor 2: Agente IA (delegado)

**¿Quién es?**
Un sistema autónomo (ej: asistente de pagos, bot de expensas) que ejecuta acciones financieras en nombre del usuario.

**¿Qué necesita del sistema?**
- Solicitar autorización para ejecutar un pago o transferencia
- Saber si fue aprobado/denegado y por qué
- Ejecutar el pago si tiene autorización
- Recibir confirmación de que el pago se procesó

**¿Qué problema le resuelve PaySentry?**
Le da **un protocolo estándar** para "pedir permiso". No necesita integrarse directamente con el procesador de pagos, solo con PaySentry.

### Actor 3: Fintech/Wallet (integrador)

**¿Quién es?**
La empresa que ofrece servicios financieros y quiere agregar capacidades de agentes IA.

**¿Qué necesita del sistema?**
- Infraestructura lista para ofrecer "AI-powered payments"
- Compliance: audit trail completo, políticas configurables
- No construir todo desde cero

**¿Qué problema le resuelve PaySentry?**
Les da **time-to-market rápido** y **compliance out-of-the-box**. En lugar de construir toda la lógica de autorización, políticas y audit, integran una API.

---

## 5. Alcance del MVP

### 5.1 QUÉ SÍ ENTRA (Must Have)

**Flujo mínimo end-to-end:**

1. **Registro de agente:** Usuario registra un agente y recibe un token de acceso
2. **Definición de política:** Usuario crea política con límite por transacción y límite diario
3. **Solicitud de autorización:** Agente pide autorización para transferencia de $X a CBU Y
4. **Evaluación automática:** Sistema valida contra políticas
5. **Aprobación manual (threshold):** Si monto > threshold, usuario debe aprobar manualmente
6. **Ejecución:** Sistema ejecuta la transferencia contra un Mock Adapter (simulado)
7. **Audit trail:** Todas las operaciones se registran en event log inmutable

**Features esenciales:**
1. Autenticación JWT con scopes separados (user_token, agent_token)
2. CRUD de políticas (límite por tx, límite diario, threshold de aprobación manual)
3. Evaluación de políticas en tiempo real
4. Estados de autorización (pending, approved, denied, captured)
5. Mock Adapter para simular procesador de pagos
6. Event store append-only para auditoría
7. API RESTful documentada (OpenAPI)

### 5.2 QUÉ NO ENTRA (Out of Scope)

**Diferido para v2:**
- Múltiples procesadores de pago (solo Mock para MVP)
- Políticas avanzadas (categorías de merchant, horarios, geolocalización)
- Webhooks para notificaciones async
- Dashboard web (solo API)
- Multi-currency (solo ARS)
- Reembolsos/chargebacks
- Rate limiting sofisticado
- Multi-tenancy

---

## 6. Métricas de Éxito

### 6.1 Métricas Técnicas

| Métrica | Target MVP | Cómo se mide |
|---------|------------|--------------|
| Latencia p99 autorización | < 500ms | Load test con k6 |
| Uptime | > 99% | Monitoring (UptimeRobot) |
| Tasa de error (errores del sistema, no denials) | < 1% | APM logs |
| Idempotencia | 100% captures duplicados retornan mismo resultado | Integration tests |

### 6.2 Métricas de Confianza (Trust Indicators)

> Estas métricas demuestran que el sistema está listo para generar confianza, sin necesidad de usuarios reales.

| Métrica | Target MVP | Por qué indica confianza |
|---------|------------|--------------------------|
| Audit trail completeness | 100% operaciones logueadas | Transparencia total = precondición de confianza. Si falta UNA operación del log, el sistema no es auditable. |
| Policy enforcement accuracy | 0% violaciones de política | Consistencia absoluta = el sistema hace lo que promete. Un solo caso de violación destruye credibilidad. |
| Policy update propagation | < 5 segundos | Control inmediato = usuario siente que tiene el poder. Latencia alta = frustración. |
| Idempotency guarantee | 100% duplicados manejados | Cero doble cobros = confianza en reliability financiera. No puede ser 99.9%, debe ser 100%. |
| Documentación completa | 100% endpoints con OpenAPI + ejemplos | Portfolio quality: demuestra profesionalismo y pensamiento arquitectónico. |

---

## 7. Riesgos Conocidos (Preview)

| Riesgo | Impacto | Probabilidad | Mitigación inicial |
|--------|---------|--------------|-------------------|
| Doble ejecución de pago (no idempotente) | Alto | Media | Idempotency keys en captures, tests de duplicación |
| Inconsistencia entre policy cache y DB | Medio | Media | TTL corto en cache (60s), invalidación explícita en updates |
| Mock Adapter oculta problemas reales de integración | Medio | Alta | Documentar asunciones sobre API real, diseño con Hexagonal Architecture |
| Scope creep (agregar features "fáciles") | Medio | Alta | Mantener lista explícita de Won't Have, revisión semanal |
| Estado inconsistente si proceso muere mid-flight | Alto | Baja | Usar transacciones DB, event sourcing para reconstruir estado |

---

## Caso de Uso Concreto: Pago Automático de Expensas

**Personaje:** Martín tiene expensas de $45.000/mes que vencen el día 10.

**Sin PaySentry:**
- Martín recibe notificación el día 10
- Debe abrir la app, ingresar CVV, confirmar pago
- Si se olvida → mora

**Con PaySentry:**
1. Martín registra "Agente de Expensas"
2. Define política:
   - Límite por transacción: $60.000
   - Solo transferencias a CBU de administración
   - Threshold aprobación manual: $50.000
3. El día 10, el agente:
   - Solicita autorización para $45.000 a CBU del consorcio
   - PaySentry valida: ✅ Dentro del límite, ✅ CBU permitido, ✅ No requiere aprobación manual
   - Aprueba automáticamente
   - Ejecuta transferencia
4. Martín recibe notificación: "Tu agente pagó $45.000 en expensas"

**Escenario de falla:**
- Mes siguiente, el agente pide $55.000 (aumento inesperado)
- PaySentry detecta: ⚠️ Supera threshold de $50k
- Status: `pending_approval`
- Martín recibe push: "Aprobar pago de $55k?"
- Martín revisa y aprueba manualmente

**Valor demostrado:**
- Automatización del caso feliz (sin fricción)
- Control en casos excepcionales (requiere aprobación)
- Audit trail completo (cada request logueado)

---

## Checklist de Completitud

- [x] Cada sección tiene contenido concreto (no placeholders)
- [x] El problema es específico y medible
- [x] El alcance MVP es realista para 8-12 semanas
- [x] Las métricas tienen números, no solo "alto" o "bueno"
- [x] Puedo explicar esto en 3 minutos a alguien no técnico

---

## Próximo Paso

**Artefacto siguiente:** `02-requirements.md` - Traducir este problema en requisitos funcionales y no funcionales verificables.
