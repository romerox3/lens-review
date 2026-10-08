# Lente — security-and-injection

> Injection, authz, secrets, SSRF, path traversal, y **prompt-injection**. La lente del revisor `lens-review:lens-security` (Opus, Tier 2). Usa modelado adversarial concreto (heredado de `differential-review/adversarial.md`): no "esto podría ser inseguro", sino "este atacante, con este acceso, por este entry point, logra esto".

## Modelo de atacante (formúlalo primero)

Antes de buscar vulns, define:
- **QUIÉN**: usuario autenticado de otro tenant, anónimo, atacante con un token robado, agente LLM con input controlado.
- **QUÉ acceso/privilegios** tiene.
- **DÓNDE** interactúa: endpoint, WS action, campo de formulario, comentario de PR, contenido de archivo.

Un hallazgo de seguridad serio incluye **entry point + secuencia + prueba de alcanzabilidad**.

## Ejes a recorrer

### 1. AuthZ / control de acceso
- ¿Toda mutación verifica ownership/membership antes de actuar? (no solo authn — authz por recurso)
- IDOR: ¿se puede pasar el id de otro usuario/workspace y operar sobre él?
- En Teros: ¿el recurso es workspace-owned? ¿se verifica membership, no solo `userId`?
- Privilege escalation: ¿un rol bajo alcanza una acción de admin?

### 2. Injection clásica
- SQL/NoSQL: ¿queries parametrizadas? (`find({channelId: null})` matchea documentos sin el campo → data leak).
- Command/shell: input que llega a `exec`/`spawn` sin sanitizar.
- Path traversal: `../` en nombres de archivo que llegan a FS.
- XSS: input que llega a HTML/render sin escapar.

### 3. SSRF y egress
- URLs controladas por el usuario que el backend fetchea: ¿whitelist de protocolo (`http`/`https`, no `file:`/`gopher:`/`javascript:`)? ¿bloqueo de IPs internas/metadata (169.254.169.254)?
- Webhooks/callbacks con destino controlado.

### 4. Secrets y datos sensibles
- ¿Secrets hardcodeados, en logs, en mensajes de error, en el diff?
- ¿Tokens/keys con scope mínimo? ¿Se loguea PII?
- Defaults inseguros (fail-open): auth desactivable por env var, CORS abierto, debug en prod.

### 5. Prompt-injection (entradas a LLMs)
- ¿Texto controlado por el usuario (PR title/body, comentarios, contenido de archivo, mensajes) fluye a un prompt de LLM con herramientas de ejecución en el mismo runtime?
- Ataque conocido ("Comment and Control"): un comentario/título de PR crafteado secuestra un agente de revisión. Si el diff añade un agente que ingiere texto no confiable, **márcalo**.
- Mitigación esperada: least-privilege en las tools del agente (read-only), separación input/output, no ejecutar acciones derivadas de texto no confiable.

### 6. Crypto y datos en reposo
- ¿Algoritmos/modos correctos? ¿IV/nonce únicos? ¿comparación constant-time de secretos?
- ¿Qué se cifra y qué no? (en Teros: solo credenciales AES-256-GCM; mensajes/memoria en claro — no afirmar E2E donde no lo hay).

## Hallazgo típico

```
CRÍTICO · security · packages/backend/src/handlers/fetch-preview.ts:54 — la URL del usuario se fetchea sin validar protocolo ni IP destino → SSRF a 169.254.169.254 (metadata) · whitelist http/https + bloquear rangos privados/link-local · confianza: alta · atacante: usuario autenticado, entry point WS action link.preview
```

## Anti-patterns

- "Está detrás de auth, no hace falta authz por recurso" → IDOR clásico.
- Validar en el cliente y confiar en el backend.
- Sanitizar para SQL pero no para el shell (o viceversa).
- Reintentar un POST con retry automático sin idempotency (duplica + amplifica abuso).
- Agente LLM con `Bash`/`Write` que ingiere texto de PR no confiable.

## Disciplina

- Construye el exploit concreto (posición inicial → pasos → impacto). Sin alcanzabilidad demostrada, baja la confianza.
- Verifica el comportamiento real de la API/lib de seguridad (regex, opciones) con context7/WebSearch — no asumas.
