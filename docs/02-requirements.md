# PaySentry - Requirements Specification

> **Instrucciones:** Este documento define QUÉ debe hacer el sistema, no CÓMO.
> Cada requisito debe ser verificable (podés probar si se cumple o no).
> Usá el formato "El sistema debe..." para requisitos funcionales.

---

## 1. Requisitos Funcionales (FR)

> **Qué son:** Comportamientos específicos del sistema. Qué hace cuando el usuario hace X.
> **Criterio de calidad:** Si no podés escribir un test case, no es un buen requisito.

### 1.1 Gestión de Usuarios

**Preguntas guía:**
- ¿Cómo se registra un usuario?
- ¿Cómo se autentica?
- ¿Qué datos mínimos necesitás del usuario?

| ID | Requisito | Prioridad | Criterio de Aceptación |
|----|-----------|-----------|------------------------|
| FR-U01 | El sistema debe permitir... | Must | Dado... Cuando... Entonces... |
| FR-U02 | | | |

<!-- 
Tip: Para MVP, probablemente:
- Registro con email/password
- Login con JWT
- NO necesitás: OAuth social, 2FA, etc.
-->

---

### 1.2 Gestión de Agentes

**Preguntas guía:**
- ¿Cómo registra un usuario a un agente?
- ¿Qué datos identifica a un agente?
- ¿Cómo se genera y entrega el token del agente?
- ¿Cómo se revoca un agente?

| ID | Requisito | Prioridad | Criterio de Aceptación |
|----|-----------|-----------|------------------------|
| FR-A01 | El sistema debe permitir a un usuario registrar un nuevo agente | Must | Dado un usuario autenticado, cuando envía POST /agents con {name}, entonces recibe {agent_id, agent_token} |
| FR-A02 | | | |
| FR-A03 | | | |

---

### 1.3 Gestión de Políticas

**Preguntas guía:**
- ¿Qué parámetros puede configurar una política?
- ¿Una política es por agente o global?
- ¿Cómo se actualizan las políticas? ¿Efecto inmediato?

**Parámetros de política a considerar:**
- Límite por transacción
- Límite diario/mensual
- Categorías permitidas/bloqueadas
- Merchants específicos bloqueados
- Threshold para aprobación manual
- Horarios permitidos (opcional)

| ID | Requisito | Prioridad | Criterio de Aceptación |
|----|-----------|-----------|------------------------|
| FR-P01 | El sistema debe permitir definir un límite máximo por transacción | Must | Dado policy.max_transaction=100, cuando agente pide auth por 150, entonces status=denied, reason=exceeded_limit |
| FR-P02 | | | |
| FR-P03 | | | |

---

### 1.4 Flujo de Autorización

**Preguntas guía:**
- ¿Qué datos envía el agente para pedir autorización?
- ¿Qué validaciones se hacen?
- ¿Cuáles son los posibles resultados?
- ¿Cuánto tiempo es válida una autorización?

**Estados posibles de una autorización:**
```
pending_approval → approved → captured
                → rejected
                → expired

approved → captured
        → voided (cancelada antes de captura)
        → expired
```

| ID | Requisito | Prioridad | Criterio de Aceptación |
|----|-----------|-----------|------------------------|
| FR-AUTH01 | El sistema debe validar el token del agente antes de procesar | Must | Dado token inválido, cuando POST /authorizations, entonces 401 Unauthorized |
| FR-AUTH02 | El sistema debe evaluar la política del agente | Must | ... |
| FR-AUTH03 | El sistema debe retornar status según resultado de evaluación | Must | Status ∈ {approved, denied, pending_approval} |
| FR-AUTH04 | | | |

---

### 1.5 Ejecución de Pagos

**Preguntas guía:**
- ¿Qué pasa cuando el agente hace capture?
- ¿Qué pasa si MercadoPago falla?
- ¿Cómo se maneja la idempotencia?
- ¿Qué información se retorna al agente?

| ID | Requisito | Prioridad | Criterio de Aceptación |
|----|-----------|-----------|------------------------|
| FR-PAY01 | El sistema debe ejecutar el pago solo si la autorización está approved | Must | Dado auth.status=denied, cuando POST /auth/{id}/capture, entonces 400 Bad Request |
| FR-PAY02 | El sistema debe ser idempotente en capture | Must | Dado capture ya ejecutado, cuando se repite con mismo idempotency_key, entonces retorna mismo resultado sin re-ejecutar |
| FR-PAY03 | | | |

---

### 1.6 Aprobación Manual

**Preguntas guía:**
- ¿Cómo se notifica al usuario que hay algo pendiente?
- ¿Cómo aprueba/rechaza?
- ¿Qué pasa si no responde? (timeout)

| ID | Requisito | Prioridad | Criterio de Aceptación |
|----|-----------|-----------|------------------------|
| FR-APR01 | El sistema debe notificar al usuario cuando una auth requiere aprobación | Should | ... |
| FR-APR02 | | | |

---

### 1.7 Historial y Auditoría

**Preguntas guía:**
- ¿Qué información debe poder consultar el usuario?
- ¿Qué filtros necesita?
- ¿Por cuánto tiempo se retiene el historial?

| ID | Requisito | Prioridad | Criterio de Aceptación |
|----|-----------|-----------|------------------------|
| FR-HIST01 | El sistema debe registrar cada intento de autorización | Must | Todo POST /authorizations genera un registro en event_log |
| FR-HIST02 | | | |

---

## 2. Requisitos No Funcionales (NFR)

> **Qué son:** Cualidades del sistema. Cómo de bien hace lo que hace.
> **Criterio de calidad:** Debe ser medible con un número.

### 2.1 Performance

**Preguntas guía (DDIA Cap 1):**
- ¿Cuál es la latencia aceptable para autorización?
- ¿Cuántas requests/segundo esperás?
- ¿Qué percentil importa? (p50, p95, p99)

| ID | Requisito | Target | Método de Verificación |
|----|-----------|--------|------------------------|
| NFR-PERF01 | Latencia de autorización (p99) | < ? ms | Load test con k6/locust |
| NFR-PERF02 | Throughput mínimo | ? req/s | Load test |
| NFR-PERF03 | | | |

<!-- 
Tip DDIA: 
- p50 = "típico", p99 = "peor caso razonable"
- Para pagos, <500ms es aceptable, <200ms es bueno
- Pensá en qué domina la latencia: red a MercadoPago, DB, etc.
-->

---

### 2.2 Availability (Disponibilidad)

**Preguntas guía:**
- ¿Qué uptime necesitás? (99% = 3.65 días down/año)
- ¿Qué pasa si el sistema está caído?
- ¿Tenés que operar 24/7 o hay ventanas de mantenimiento?

| ID | Requisito | Target | Método de Verificación |
|----|-----------|--------|------------------------|
| NFR-AVAIL01 | Uptime mensual | ? % | Monitoring (mejor servicio) |
| NFR-AVAIL02 | | | |

<!-- 
Tip DDIA:
- 99% = 7.2 horas down/mes
- 99.9% = 43 min down/mes
- 99.99% = 4.3 min down/mes (muy difícil de lograr)
Para MVP, 99% es realista.
-->

---

### 2.3 Reliability (Confiabilidad)

**Preguntas guía:**
- ¿Qué errores son críticos? (pérdida de dinero, doble cobro)
- ¿Qué garantías de entrega necesitás?
- ¿Cómo manejás fallos de MercadoPago?

| ID | Requisito | Target | Método de Verificación |
|----|-----------|--------|------------------------|
| NFR-REL01 | Tasa de errores en pagos | < ? % | Métricas de producción |
| NFR-REL02 | Cero doble cobros | 0 | Idempotency keys + reconciliación |
| NFR-REL03 | | | |

<!-- 
Tip DDIA Cap 7 (Transactions):
- At-least-once delivery con idempotencia = exactly-once semántico
- Pensá en: ¿qué pasa si el proceso muere entre cobrar y confirmar?
-->

---

### 2.4 Security

**Preguntas guía:**
- ¿Qué datos son sensibles?
- ¿Necesitás compliance específico? (PCI-DSS?)
- ¿Cómo protegés los tokens?

| ID | Requisito | Target | Método de Verificación |
|----|-----------|--------|------------------------|
| NFR-SEC01 | Tokens JWT firmados y con expiración | Exp < 24h | Code review, tests |
| NFR-SEC02 | No almacenar datos de tarjeta | 0 PAN stored | Audit |
| NFR-SEC03 | | | |

<!-- 
Nota: Al usar MercadoPago, ellos manejan PCI. 
Pero vos seguís siendo responsable de proteger tus tokens y datos de usuario.
-->

---

### 2.5 Scalability

**Preguntas guía (DDIA Cap 1):**
- ¿Cómo crece la carga? (lineal, picos?)
- ¿Cuál es tu estrategia de scaling? (vertical primero, horizontal después?)
- ¿Qué componente es el cuello de botella probable?

| ID | Requisito | Target | Método de Verificación |
|----|-----------|--------|------------------------|
| NFR-SCALE01 | Soportar crecimiento sin rediseño hasta | ? usuarios / ? req/s | Load test |
| NFR-SCALE02 | | | |

<!-- 
Tip: Para MVP, no over-enginear. 
"Make it work, make it right, make it fast" - en ese orden.
-->

---

### 2.6 Maintainability

**Preguntas guía (DDIA Cap 1):**
- ¿Qué tan fácil es entender el código?
- ¿Qué tan fácil es deployar cambios?
- ¿Qué tan fácil es debuggear problemas?

| ID | Requisito | Target | Método de Verificación |
|----|-----------|--------|------------------------|
| NFR-MAINT01 | Cobertura de tests | > ? % | CI pipeline |
| NFR-MAINT02 | Deploy automatizado | < ? min de commit a prod | CI/CD |
| NFR-MAINT03 | Logs estructurados | Todos los requests logueados con correlation_id | Code review |

---

## 3. Restricciones (Constraints)

> **Qué son:** Limitaciones impuestas externamente que no podés cambiar.

### 3.1 Técnicas

| ID | Restricción | Razón |
|----|-------------|-------|
| CON-T01 | Procesador de pagos: MercadoPago | Decisión ya tomada, mercado AR |
| CON-T02 | Presupuesto cloud: $0-200/mes | Restricción personal |
| CON-T03 | | |

### 3.2 De Negocio

| ID | Restricción | Razón |
|----|-------------|-------|
| CON-B01 | No requiere licencia bancaria | Simplifica regulación |
| CON-B02 | | |

### 3.3 De Tiempo

| ID | Restricción | Razón |
|----|-------------|-------|
| CON-TIME01 | MVP en ~8-12 semanas | Proyecto paralelo |
| CON-TIME02 | | |

---

## 4. Assumptions (Supuestos)

> **Qué son:** Cosas que asumís como ciertas pero que podrían no serlo.

| ID | Supuesto | Riesgo si es falso |
|----|----------|-------------------|
| ASM-01 | MercadoPago API es estable y bien documentada | Retrasos en integración |
| ASM-02 | Los agentes IA enviarán requests bien formados | Más validación necesaria |
| ASM-03 | | |

---

## 5. Priorización (MoSCoW)

**Resumí la priorización de todos los requisitos:**

### Must Have (MVP no funciona sin esto)
- FR-AUTH01, FR-AUTH02, FR-AUTH03
- FR-PAY01, FR-PAY02
- FR-P01
- ...

### Should Have (Importante pero no bloqueante)
- FR-APR01
- ...

### Could Have (Nice to have)
- ...

### Won't Have (Explícitamente fuera de scope)
- Multi-currency
- Múltiples procesadores de pago
- Mobile SDK
- ...

---

## Checklist de Completitud

- [ ] Cada FR tiene criterio de aceptación verificable
- [ ] Cada NFR tiene un número target
- [ ] Las prioridades están asignadas (Must/Should/Could/Won't)
- [ ] Las restricciones están documentadas
- [ ] Los supuestos están explícitos

---

## Próximo Paso

Cuando completes este documento:
1. `git commit -m "docs: add requirements specification v1"`
2. Pedí review: "Claude, revisá mis requirements"
3. Iterá basándote en feedback
4. Siguiente artefacto: `/docs/03-data-model.md`
