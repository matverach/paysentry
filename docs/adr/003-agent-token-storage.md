# ADR-003: Agent Token Storage Strategy

## Status

**ACCEPTED**

---

## Context

Los agent tokens son credenciales de larga duración que permiten a los agentes IA ejecutar autorizaciones de pago en nombre del usuario. A diferencia de los user tokens (JWT con exp=24h), los agent tokens no tienen expiración automática y solo se invalidan cuando el usuario revoca el agente.

**El problema:** ¿Cómo almacenamos estos tokens en la base de datos para balancear seguridad y usabilidad?

**Fuerzas en juego:**
- **Seguridad:** Si la DB es comprometida (SQL injection, dump accidental), queremos minimizar el daño
- **Usabilidad:** Los usuarios pueden necesitar ver/copiar el token para reconfigurarlo
- **Simplicidad:** Evitar infraestructura compleja de key management para un MVP
- **Compliance:** Aunque no manejamos datos de tarjetas (PCI-DSS), sí manejamos credenciales de acceso

**Por qué ahora:** Esta decisión impacta el diseño del endpoint POST /agents y el schema de la tabla `agents`. Debe tomarse antes de implementar.

---

## Decision

**Almacenaremos agent tokens hasheados en la base de datos usando bcrypt, y los mostraremos en plain text solo una vez en la respuesta de creación.**

Esto significa:
1. Cuando el usuario crea un agente → generamos token con formato `agt_{32_random_chars}`
2. Retornamos `{agent_id, agent_token}` en la respuesta HTTP (una sola vez)
3. Almacenamos `bcrypt(agent_token)` en `agents.token_hash`
4. **No hay endpoint para recuperar el token después** — si se pierde, debe crear un nuevo agente

**Patrón:** Exactly-once token delivery

---

## Consequences

### Positivas
- **Seguridad:** Dump de DB no expone tokens utilizables → mitigación de riesgo crítico
- **Industry standard:** Mismo patrón que GitHub Personal Access Tokens, Stripe API Keys, AWS Access Keys
- **Simplicidad:** No requiere encryption key management ni infraestructura adicional
- **Auditabilidad:** Logs de autenticación pueden verificar uso del token sin exponerlo

### Negativas
- **UX friction:** Si el usuario pierde el token, debe revocar el agente viejo y crear uno nuevo
- **No recuperable:** No hay forma de "resetear" o "ver mi token" — esto puede generar soporte adicional
- **Educación:** Requiere documentación clara en POST /agents explicando "copiar ahora o perderlo"

### Neutras
- La creación de agentes es relativamente infrecuente (1-5 agentes por usuario típicamente)
- El costo de bcrypt en validación es negligible comparado con latencia de red

---

## Alternatives Considered

| Alternativa | Pros | Cons | Razón de Descarte |
|-------------|------|------|-------------------|
| Plain text storage | Simple, recuperable, podés mostrar tokens después | Dump de DB = compromiso total de todos los tokens | Riesgo de seguridad inaceptable |
| Encrypted storage | Tokens recuperables, más seguro que plain text | Requiere key management, encryption keys en memoria/config, si comprometen la key = mismo que plain text | Complejidad injustificada para MVP, no elimina riesgo fundamental |
| Hasheo con bcrypt (elegida) | Seguro, industry standard, simple | No recuperable después de creación | N/A — el trade-off de UX es aceptable para ganar seguridad |
| JWT con refresh tokens | Tokens cortos + refresh, patrón conocido | Agentes necesitan lógica de refresh, complejidad en gestión de sesiones | Over-engineering para MVP — agentes no son users interactivos |

---

## References

- [OWASP: Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html) - Aplica también a tokens de larga duración
- GitHub Personal Access Tokens - Mismo patrón de "mostrar una vez"
- Stripe API Keys - Prefijos identificables (`sk_`, `pk_`) + almacenamiento hasheado

---

## Notes

**Formato del token:** `agt_{32_chars_base62}`
- El prefijo `agt_` permite identificar el tipo de token en logs (vs `usr_` para user tokens)
- Base62 (0-9, a-z, A-Z) evita caracteres ambiguos y es URL-safe
- 32 chars = 190 bits de entropía (más que suficiente para prevenir brute force)

**Implementación de validación:**
```python
# En cada request con agent_token
def validate_agent_token(token: str) -> Agent | None:
    # Extraer los primeros 8 chars como hint para búsqueda (opcional, optimización)
    # Hacer bcrypt.checkpw(token, agent.token_hash) para cada agente activo del usuario
    # Si match → retornar agente
    # Si no match → retornar None (401 Unauthorized)
```

**Consideración futura (post-MVP):** Si el volumen de autenticaciones se vuelve problema (bcrypt es CPU-intensive), podemos:
1. Agregar cache de tokens validados (Redis con TTL corto)
2. Usar argon2 en lugar de bcrypt (más moderno, mejor perf)
3. Implementar token rotation periódica con notificaciones

Pero para MVP con <10 agentes por usuario, bcrypt directo es suficiente.
