# PaySentry - Requirements Specification

---

## 1. Constraints (Restricciones)

> **Qué son:** Limitaciones impuestas externamente que no podés cambiar. Definen el espacio de soluciones posibles.

### 1.1 Técnicas

| ID | Restricción | Razón |
|----|-------------|-------|
| CON-T01 | Procesador de pagos: MercadoPago (Mock Adapter para MVP) | Mercado argentino, decisión ya tomada (ADR-001) |
| CON-T02 | Presupuesto cloud: $0-200/mes | Proyecto personal, no comercial |
| CON-T03 | Backend: Python (FastAPI) | Stack conocido, rápido para prototipar |
| CON-T04 | Base de datos: PostgreSQL | ACID requerido, ya decidido |

### 1.2 De Negocio

| ID | Restricción | Razón |
|----|-------------|-------|
| CON-B01 | No requiere licencia bancaria | Simplifica regulación, actúa como middleware |
| CON-B02 | Solo mercado argentino (ARS, MercadoPago) | Scope limitado para MVP |

### 1.3 De Tiempo

| ID | Restricción | Razón |
|----|-------------|-------|
| CON-TIME01 | Fase de diseño: ~6-8 semanas | Proyecto paralelo al trabajo full-time |
| CON-TIME02 | Implementación MVP: ~4-6 semanas | Solo funcionalidad core |

---

## 2. Casos de Uso Críticos

> **Por qué primero:** Todos los requisitos funcionales se derivan de estos casos de uso concretos.

### Caso de Uso 1: Pago Automático de Expensas (Happy Path)

**Actor:** Usuario (Martín) + Agente (Bot de Expensas)

**Flujo:**
1. Martín registra "Bot de Expensas" → recibe `agent_token`
2. Martín crea política: `{max_per_transaction: 60000, daily_limit: 100000, approval_threshold: 50000}`
3. Día 10 del mes: Bot solicita autorización para $45.000 a CBU del consorcio
4. PaySentry evalúa:
   - ✅ Monto < max_per_transaction (45k < 60k)
   - ✅ Monto < daily_limit (45k < 100k, sin gasto previo)
   - ✅ Monto < approval_threshold (45k < 50k) → **auto-aprobado**
5. PaySentry ejecuta pago contra Mock Adapter
6. Retorna `{status: "captured", payment_id: "..."}` al agente
7. Martín ve notificación: "Tu agente pagó $45.000 en expensas"

**Requisitos derivados:** FR-A01, FR-P01, FR-P02, FR-P03, FR-AUTH01, FR-AUTH02, FR-PAY01, FR-HIST01

---

### Caso de Uso 2: Monto Excede Threshold → Aprobación Manual

**Actor:** Usuario (Martín) + Agente (Bot de Expensas)

**Flujo:**
1. Mes siguiente: expensas suben a $55.000 (aumento de consorcio)
2. Bot solicita autorización por $55.000
3. PaySentry evalúa:
   - ✅ Monto < max_per_transaction (55k < 60k)
   - ✅ Monto < daily_limit (55k < 100k)
   - ⚠️ Monto > approval_threshold (55k > 50k) → **requiere aprobación manual**
4. PaySentry retorna `{status: "pending_approval", authorization_id: "auth_123"}`
5. Martín recibe notificación push (simulada con GET /authorizations?status=pending)
6. Martín revisa y aprueba: `POST /authorizations/auth_123/approve`
7. Bot hace capture: `POST /authorizations/auth_123/capture`
8. PaySentry ejecuta pago

**Requisitos derivados:** FR-P04, FR-APR01, FR-APR02, FR-AUTH03

---

### Caso de Uso 3: Violación de Política → Rechazo Automático

**Actor:** Agente (Bot de Expensas)

**Flujo:**
1. Bot solicita autorización por $70.000 (error en configuración o ataque)
2. PaySentry evalúa:
   - ❌ Monto > max_per_transaction (70k > 60k)
3. PaySentry retorna `{status: "denied", reason: "exceeded_max_transaction_limit"}`
4. Bot no puede capturar
5. Evento queda registrado en audit log

**Requisitos derivados:** FR-P02, FR-AUTH02, FR-HIST01

---

## 3. Requisitos No Funcionales (NFR)

> **Por qué antes de FRs:** Los NFRs definen la arquitectura. Un cambio aquí cambia TODO el diseño.

### 3.1 Performance

| ID | Requisito | Target | Justificación | Método de Verificación |
|----|-----------|--------|---------------|------------------------|
| NFR-PERF01 | Latencia de autorización (p95) | < 2000ms | Límite aceptable para UX en flujos de pago. Balances responsiveness vs complejidad de implementación para MVP. | Manual testing, opcional load test con k6 |
| NFR-PERF02 | Throughput mínimo | > 1 req/s | Suficiente para validación de concepto con carga limitada | No requiere testing específico para MVP |

**Rationale:** Para MVP, el foco está en arquitectura correcta y confiabilidad sobre optimización prematura de performance. Targets escalan linealmente agregando recursos.

---

### 3.2 Reliability (Confiabilidad)

| ID | Requisito | Target | Justificación | Método de Verificación |
|----|-----------|--------|---------------|------------------------|
| NFR-REL01 | Cero doble cobros | 100% idempotencia garantizada | Crítico para sistemas financieros. Implementa at-least-once delivery + idempotencia = exactly-once semántico (DDIA Cap 7) | Integration tests con retries, manual testing con captures duplicados |
| NFR-REL02 | Audit trail completo | 100% operaciones logueadas | Compliance requirement. Implementa event sourcing para reconstrucción de estado (ADR-002) | Test: verificar que cada POST /authorizations genera evento en event_log |
| NFR-REL03 | Policy enforcement accuracy | 0% violaciones de política | Core value proposition del sistema - autorización confiable es requisito fundamental | Tests unitarios + e2e tests con diferentes políticas |

**Rationale:** Estos NFRs definen la arquitectura del sistema. Son medibles, tienen targets absolutos (no "mejor esfuerzo"), y son no-negociables para un gateway de autorización financiera.

---

### 3.3 Availability (Disponibilidad)

| ID | Requisito | Target | Justificación | Método de Verificación |
|----|-----------|--------|---------------|------------------------|
| NFR-AVAIL01 | Uptime durante horario de demo | Best-effort, sin SLA | Es un POC, downtime no tiene impacto comercial. Sin fallback ni HA para MVP. | Monitoring manual, UptimeRobot opcional |

**Rationale:** Para un MVP single-tenant, best-effort availability es aceptable. Arquitectura permite agregar redundancia cuando escale.

---

### 3.4 Security

| ID | Requisito | Target | Justificación | Método de Verificación |
|----|-----------|--------|---------------|------------------------|
| NFR-SEC01 | User tokens JWT con expiración | exp = 24h, firmados con HS256 | Tokens de sesión de corta duración, balance seguridad/UX | Code review, tests de validación |
| NFR-SEC02 | Agent tokens hasheados | Almacenamiento con bcrypt (ADR-003) | Credenciales de larga duración, protección contra dump de DB | Code review, verificar que GET /agents no expone tokens |
| NFR-SEC03 | No almacenar datos de tarjeta | 0 PAN/CVV stored | PCI-DSS compliance - MercadoPago maneja tokenización | Audit de schema DB |
| NFR-SEC04 | HTTPS en producción | TLS 1.2+ | Protección de tokens en tránsito | Verificar deploy en Railway/Render |

**Rationale:** Security debe ser "correcta por diseño", no agregada después. Estas prácticas son estándar en sistemas que manejan credenciales.

---

### 3.5 Maintainability

| ID | Requisito | Target | Justificación | Método de Verificación |
|----|-----------|--------|---------------|------------------------|
| NFR-MAINT01 | Cobertura de tests | > 70% para lógica core (policy engine, idempotencia) | Garantiza confiabilidad en componentes críticos del sistema | Coverage report en CI |
| NFR-MAINT02 | Logs estructurados | Todos los requests con correlation_id | Debuggabilidad y trazabilidad - crítico para sistemas financieros | Code review, verificar formato JSON |
| NFR-MAINT03 | Documentación de API | OpenAPI 3.0 completa con ejemplos | Facilita integración de clientes y mantenibilidad del contrato de API | Validar que openapi.yaml cubre todos los endpoints |

---

## 4. Requisitos Funcionales (FR)

> **Derivados de los 3 casos de uso críticos**

### 4.1 Gestión de Usuarios

| ID | Requisito | Prioridad | Criterio de Aceptación |
|----|-----------|-----------|------------------------|
| FR-U01 | El sistema debe permitir registrar un usuario con email y password | Must | Dado email válido y password >= 8 chars, cuando POST /users/register, entonces retorna {user_id, user_token} y status 201 |
| FR-U02 | El sistema debe autenticar usuarios con email/password | Must | Dado credenciales válidas, cuando POST /auth/login, entonces retorna JWT con exp=24h y scopes={policies:*, agents:*, authorizations:*} |
| FR-U03 | El sistema debe rechazar autenticación con credenciales inválidas | Must | Dado password incorrecto, cuando POST /auth/login, entonces retorna 401 Unauthorized |
| FR-U04 | El sistema debe validar formato de email en registro | Should | Dado email inválido (sin @), cuando POST /users/register, entonces retorna 400 Bad Request con mensaje descriptivo |

---

### 4.2 Gestión de Agentes

| ID | Requisito | Prioridad | Criterio de Aceptación | Caso de Uso |
|----|-----------|-----------|------------------------|-------------|
| FR-A01 | El sistema debe permitir a un usuario registrar un nuevo agente | Must | Dado user_token válido, cuando POST /agents con {name, description}, entonces retorna {agent_id, agent_token} en plain text una sola vez | CU1, CU2, CU3 |
| FR-A02 | El sistema debe almacenar agent_token hasheado (ADR-003) | Must | Los agent_tokens se almacenan hasheados en DB con bcrypt, imposibilitando recuperación posterior | CU1 |
| FR-A03 | El sistema debe generar agent_tokens con formato identificable | Must | Tokens generados tienen formato "agt_" + 32 chars aleatorios (base62), scopes implícitos: {authorizations:create, authorizations:read} | CU1 |
| FR-A04 | El sistema debe permitir listar agentes del usuario | Should | Dado user_token válido, cuando GET /agents, entonces retorna lista con {id, name, created_at, status} sin exponer tokens | CU1 |
| FR-A05 | El sistema debe permitir revocar un agente | Must | Dado user_token válido, cuando DELETE /agents/{id}, entonces marca agent.status=revoked y rechaza autorizaciones futuras con ese token | CU3 |
| FR-A06 | El sistema debe validar ownership en revocación | Must | Dado user_token de usuario X, cuando DELETE /agents/{id} de usuario Y, entonces retorna 403 Forbidden | CU3 |

---

### 4.3 Gestión de Políticas

| ID | Requisito | Prioridad | Criterio de Aceptación | Caso de Uso |
|----|-----------|-----------|------------------------|-------------|
| FR-P01 | El sistema debe permitir crear una política asociada a un agente | Must | Dado user_token válido y agent_id existente, cuando POST /policies con {agent_id, max_amount_per_transaction, daily_limit, approval_threshold}, entonces retorna {policy_id} y status 201 | CU1, CU2 |
| FR-P02 | El sistema debe validar monto máximo por transacción | Must | Dado policy.max_amount_per_transaction=60000, cuando agente solicita auth por 70000, entonces status=denied, reason="exceeded_max_transaction_limit" | CU3 |
| FR-P03 | El sistema debe validar límite diario acumulado | Must | Dado policy.daily_limit=100000 y agente ya gastó 80000 hoy, cuando solicita auth por 30000, entonces status=denied, reason="exceeded_daily_limit" | CU1 |
| FR-P04 | El sistema debe permitir definir threshold para aprobación manual | Must | Dado policy.approval_threshold=50000, cuando agente solicita auth por 55000, entonces status=pending_approval (no denied automáticamente) | CU2 |
| FR-P05 | El sistema debe aplicar cambios de política inmediatamente | Should | Dado policy actualizada con nuevo max_amount, cuando siguiente auth llega <5 segundos después, entonces usa los nuevos límites | CU1 |
| FR-P06 | El sistema debe permitir actualizar una política existente | Must | Dado user_token válido y policy_id existente, cuando PUT /policies/{id} con nuevos valores, entonces actualiza y retorna 200 OK | CU2 |
| FR-P07 | El sistema debe validar que solo el owner puede modificar políticas | Must | Dado user_token de usuario X, cuando PUT /policies/{id} de agente de usuario Y, entonces 403 Forbidden | Seguridad |

---

### 4.4 Flujo de Autorización

| ID | Requisito | Prioridad | Criterio de Aceptación | Caso de Uso |
|----|-----------|-----------|------------------------|-------------|
| FR-AUTH01 | El sistema debe validar el token del agente antes de procesar | Must | Dado agent_token inválido o revocado, cuando POST /authorizations, entonces 401 Unauthorized | CU1, CU3 |
| FR-AUTH02 | El sistema debe evaluar la política del agente contra el monto solicitado | Must | Dado policy con límites y auth request con monto, cuando evalúa, entonces retorna status ∈ {approved, denied, pending_approval} según reglas de política | CU1, CU2, CU3 |
| FR-AUTH03 | El sistema debe retornar status específico según resultado de evaluación | Must | Status debe indicar claramente: approved (auto-aprobado), denied (violación de política), pending_approval (requiere intervención manual) | CU1, CU2, CU3 |
| FR-AUTH04 | El sistema debe generar authorization_id único para cada solicitud | Must | Cada POST /authorizations genera un authorization_id único usado para capture posterior | CU1, CU2 |
| FR-AUTH05 | El sistema debe validar estructura del request de autorización | Must | Dado request sin campos requeridos (amount, destination_cbu), entonces 400 Bad Request con detalle de campos faltantes | CU1 |

---

### 4.5 Ejecución de Pagos

| ID | Requisito | Prioridad | Criterio de Aceptación | Caso de Uso |
|----|-----------|-----------|------------------------|-------------|
| FR-PAY01 | El sistema debe ejecutar el pago solo si la autorización está approved | Must | Dado auth.status=denied o pending_approval, cuando POST /authorizations/{id}/capture, entonces 400 Bad Request | CU1, CU2 |
| FR-PAY02 | El sistema debe ser idempotente en capture (NFR-REL01) | Must | Dado capture ya ejecutado con authorization_id, cuando se repite POST /authorizations/{id}/capture, entonces retorna mismo resultado (mismo payment_id) sin re-ejecutar | CU1 |
| FR-PAY03 | El sistema debe ejecutar contra Mock Adapter en MVP | Must | Para MVP, POST /authorizations/{id}/capture simula ejecución y retorna payment_id fake sin integración real a MercadoPago (ADR-001) | CU1 |
| FR-PAY04 | El sistema debe actualizar daily_spent después de capture exitoso | Must | Dado capture exitoso, cuando actualiza estado, entonces incrementa policy.daily_spent para validaciones futuras | CU1, CU3 |

---

### 4.6 Aprobación Manual

| ID | Requisito | Prioridad | Criterio de Aceptación | Caso de Uso |
|----|-----------|-----------|------------------------|-------------|
| FR-APR01 | El sistema debe permitir al usuario aprobar una autorización pendiente | Must | Dado auth.status=pending_approval, cuando POST /authorizations/{id}/approve con user_token válido, entonces auth.status → approved | CU2 |
| FR-APR02 | El sistema debe permitir al usuario rechazar una autorización pendiente | Should | Dado auth.status=pending_approval, cuando POST /authorizations/{id}/reject con user_token válido, entonces auth.status → rejected | CU2 |
| FR-APR03 | El sistema debe permitir listar autorizaciones pendientes | Should | Dado user_token válido, cuando GET /authorizations?status=pending_approval, entonces retorna lista de auths que requieren aprobación | CU2 |

---

### 4.7 Historial y Auditoría

| ID | Requisito | Prioridad | Criterio de Aceptación | Caso de Uso |
|----|-----------|-----------|------------------------|-------------|
| FR-HIST01 | El sistema debe registrar cada intento de autorización en event log (NFR-REL02) | Must | Todo POST /authorizations genera un registro inmutable en event_log con timestamp, agent_id, amount, result | CU1, CU2, CU3 |
| FR-HIST02 | El sistema debe permitir consultar historial de transacciones | Should | Dado user_token válido, cuando GET /transactions, entonces retorna lista de autorizaciones con filtros por date_range, agent_id, status | CU1 |
| FR-HIST03 | El sistema debe registrar cambios de estado de autorizaciones | Must | Cada transición de estado (pending → approved → captured) genera evento en log | CU2 |

---

## 5. Priorización (MoSCoW)

### Must Have (MVP no funciona sin esto)

**Core flow (CU1 - Happy path):**
- FR-U01, FR-U02, FR-U03
- FR-A01, FR-A02, FR-A03, FR-A05
- FR-P01, FR-P02, FR-P03, FR-P04, FR-P06
- FR-AUTH01, FR-AUTH02, FR-AUTH03, FR-AUTH04
- FR-PAY01, FR-PAY02, FR-PAY03, FR-PAY04
- FR-APR01
- FR-HIST01, FR-HIST03

**NFRs críticos:**
- NFR-REL01, NFR-REL02, NFR-REL03
- NFR-SEC01, NFR-SEC02, NFR-SEC03

---

### Should Have (Importante pero no bloqueante)

- FR-U04 (validación de email - nice to have)
- FR-A04 (listar agentes - útil pero no crítico)
- FR-P05 (propagación inmediata - puede tener latencia aceptable)
- FR-APR02, FR-APR03 (reject + listing - complement del flujo)
- FR-HIST02 (consulta de historial - demo value)

---

### Could Have (Nice to have para post-MVP)

- Paginación en GET /transactions
- Filtros avanzados en historial
- Webhooks para notificaciones
- Métricas de uso por agente

---

### Won't Have (Explícitamente fuera de scope para MVP)

**Funcionalidad:**
- Multi-currency (solo ARS)
- Múltiples procesadores de pago (solo Mock Adapter)
- Categorías de merchant en políticas
- Horarios permitidos en políticas
- Geolocalización
- Reembolsos/chargebacks
- Rate limiting sofisticado (más allá de validación básica)
- Notificaciones push reales (solo polling con GET)
- Dashboard web (solo API + CLI/Postman)

**Arquitectura:**
- Multi-tenancy (solo single-tenant para POC)
- Horizontal scaling (single instance suficiente)
- Multi-region deployment
- Cache distribuido (Redis solo para deduplicación si necesario)
- Queue distribuida (procesamiento síncrono suficiente)

**Ops:**
- Monitoring avanzado (solo logs + UptimeRobot opcional)
- Alerting automático
- Auto-scaling
- Disaster recovery
- Backup automatizado

---

## 6. Assumptions (Supuestos)

> **Qué asumimos como cierto, pero podría ser falso**

| ID | Supuesto | Riesgo si es falso | Mitigación |
|----|----------|-------------------|------------|
| ASM-01 | MercadoPago API es estable y bien documentada | Retrasos en integración post-MVP | Usar Mock Adapter en MVP (ADR-001), diseño con Hexagonal Architecture permite cambiar procesador |
| ASM-02 | Los agentes IA enviarán requests bien formados | Más validación necesaria, errores frecuentes | Validación estricta en API, buenos mensajes de error |
| ASM-03 | Bcrypt performance es aceptable para validación de tokens | Latencia alta en auth si muchos agentes | Cache de validaciones recientes (post-MVP), o cambiar a argon2 |
| ASM-04 | PostgreSQL en free/low-tier hosting soporta carga esperada del MVP | Downtime o throttling | Acceptable para MVP, plan de upgrade documentado para producción |
| ASM-05 | Carga inicial limitada (single-tenant, baja concurrencia) | No necesita multi-tenancy, scaling, HA en MVP | Arquitectura permite agregar multi-tenancy después si proyecto escala |

---

## 7. Success Criteria (Criterio de "Done")

El MVP está completo cuando:

### Funcional
- ✅ Los 3 casos de uso (CU1, CU2, CU3) funcionan end-to-end
- ✅ E2E test demuestra flujo completo: registro → política → autorización → capture
- ✅ Demo script ejecutable muestra sistema funcionando

### No Funcional
- ✅ NFR-REL01: Test con captures duplicados retorna mismo payment_id
- ✅ NFR-REL02: Audit log tiene 100% de operaciones
- ✅ NFR-REL03: Tests demuestran enforcement de políticas

### Documentación
- ✅ OpenAPI spec completa y validada
- ✅ README permite a otro dev correr el proyecto localmente
- ✅ ADRs documentan decisiones mayores (ADR-001, ADR-002, ADR-003)

### Deployment
- ✅ Sistema deployado y accesible vía URL pública
- ✅ Health check endpoint funcional
- ✅ Logs agregados y accesibles

---

## References

- **DDIA Cap 1:** Reliability, Scalability, Maintainability - Base para NFRs
- **DDIA Cap 7:** Transactions - Idempotencia y exactly-once semantics
- **DDIA Cap 9:** Consistency and Consensus - Read-your-writes para políticas
- **DDIA Cap 11:** Stream Processing - Event sourcing para audit trail
- **ADR-001:** Mock Adapter for MVP
- **ADR-002:** Transactional Event Log
- **ADR-003:** Agent Token Storage Strategy
