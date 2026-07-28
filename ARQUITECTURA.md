# Arquitectura — Línea Correcta

## 1. Visión general

Backend en **Rust** (Axum o Actix-web), expuesto como **API REST**, con app móvil y web como clientes. El sistema gestiona expedientes, actas de veredicto firmadas (Ed25519) y su anclaje criptográfico (merkle root) en una cadena pública. Se organiza siguiendo **arquitectura hexagonal (puertos y adaptadores)**, principios **SOLID**, **DRY**, buenas prácticas de diseño de **API REST** y el checklist **OWASP** (API Security Top 10 / ASVS).

## 2. Arquitectura hexagonal (puertos y adaptadores)

El dominio no debe conocer detalles de framework HTTP, base de datos ni infraestructura externa. Todo acceso externo pasa por **puertos** (interfaces/traits) implementados por **adaptadores**.

```
                     ┌─────────────────────────────┐
                     │        Adaptadores IN        │
                     │  HTTP (Axum) · CLI · Webhooks│
                     └──────────────┬───────────────┘
                                    │ implementan
                     ┌──────────────▼───────────────┐
                     │      Puertos de entrada        │
                     │  (casos de uso / traits)       │
                     ├─────────────────────────────────┤
                     │           DOMINIO (core)        │
                     │  Entidades: Expediente, Acta,    │
                     │  Revisor, Firma, Veredicto        │
                     │  Reglas de negocio puras          │
                     ├─────────────────────────────────┤
                     │      Puertos de salida          │
                     │  (repos, firmante, ancla, etc.)  │
                     └──────────────┬───────────────┘
                                    │ implementan
                     ┌──────────────▼───────────────┐
                     │        Adaptadores OUT        │
                     │ Postgres · Ed25519 signer ·    │
                     │ Blockchain anchor · S3 · Email │
                     └─────────────────────────────────┘
```

### Capas y responsabilidades

Cada capa vive en su propio crate de Rust dentro del workspace (convención de nombres con guion, ver estructura completa en §16: `domain`, `application`, `adapters-http`, `adapters-db`, `adapters-crypto`).

- **`domain`** — entidades y reglas de negocio puras (sin `async`, sin dependencias de framework ni de I/O). Ej.: `Expediente`, `Acta`, `Veredicto` (enum `Validado | NoConcluyente | Desvirtuado`), invariantes como "un acta requiere quórum mínimo de firmas".
- **`application`** (puertos + casos de uso) — orquesta el dominio. Define traits como `ExpedienteRepository`, `FirmaService`, `AnclajeService`. Los casos de uso (`CrearExpediente`, `FirmarActa`, `ConsultarHistorialCargo`) dependen solo de estos traits, nunca de implementaciones concretas.
- **`adapters-http`** (adaptador de entrada) — controladores Axum: deserializan requests, invocan casos de uso, serializan respuestas. Sin lógica de negocio.
- **`adapters-db`, `adapters-crypto`** (adaptadores de salida) — implementaciones concretas: `PostgresExpedienteRepository`, `Ed25519FirmaService`, `ChainAnclajeService`. Intercambiables sin tocar el dominio (clave para tests con dobles/fakes).

**Regla de dependencia:** las flechas de dependencia siempre apuntan hacia el dominio. `adapters` depende de `application`, `application` depende de `domain`; nunca al revés.

## 3. Principios SOLID (aplicados a Rust)

- **S — Responsabilidad única**: cada struct/módulo tiene un solo motivo de cambio. Un `AuditoriaService` no debe también enviar notificaciones; eso va en `NotificacionService`.
- **O — Abierto/cerrado**: nuevas fuentes de evidencia (SECOP, SIGEP, sentencia judicial) se agregan implementando un trait `FuenteEvidencia`, sin modificar el caso de uso que las consume.
- **L — Sustitución de Liskov**: cualquier implementación de un trait (`ExpedienteRepository`) debe ser sustituible por otra (Postgres real vs. fake en memoria) sin romper el comportamiento esperado por quien la consume.
- **I — Segregación de interfaces**: traits pequeños y específicos (`FirmaVerifier`, `FirmaSigner`) en vez de un `CryptoService` monolítico que obligue a implementar métodos que no se usan.
- **D — Inversión de dependencias**: los casos de uso reciben sus dependencias vía traits inyectados (constructor injection), no instancian adaptadores concretos directamente. Facilita testear con mocks/fakes.

## 4. DRY (Don't Repeat Yourself)

- Validaciones repetidas (ej. formato de cédula, verificación de firma Ed25519) viven en un único módulo `domain::validation`, no duplicadas por handler.
- Lógica de mapeo DTO↔dominio centralizada por tipo, no copiada en cada endpoint.
- Errores de dominio (`DomainError`) mapeados a códigos HTTP en **un solo lugar** (middleware/`From<DomainError> for ApiError`), no repetido en cada controlador.
- Lógica compartida entre binarios (API, `lc-worker-anclaje`, `lc-worker-outbox`, ver §16) vive en los crates de `domain`/`application`, nunca copiada dentro de cada binario.
- DRY no es absoluto: preferir una pequeña duplicación explícita sobre una abstracción prematura que acople módulos que deberían evolucionar por separado.

## 5. Buenas prácticas de API REST

- **Recursos, no verbos**: `POST /expedientes`, `GET /expedientes/{id}`, `POST /expedientes/{id}/firmas` — no `/crearExpediente`.
- **Sustantivos en plural** y anidación coherente con la jerarquía del dominio (`/expedientes/{id}/actas`).
- **Códigos de estado correctos**: `201` al crear + header `Location`; `200` en lecturas; `204` en borrados; `400` validación; `401`/`403` auth; `404` no existe; `409` conflicto (ej. firma duplicada); `422` semánticamente inválido; `429` rate limit.
- **Versionado explícito**: `/api/v1/...` desde el día uno.
- **Paginación** basada en cursor para listados grandes (`/expedientes?cursor=...&limit=50`), no offset sobre tablas que crecen sin límite.
- **Idempotencia** en operaciones de escritura sensibles (firmar un acta) vía `Idempotency-Key`, para evitar dobles firmas por reintentos de red.
- **HATEOAS ligero opcional**: enlaces a acciones relacionadas (`self`, `firmas`, `replica`) cuando aporte navegabilidad real.
- **Contratos versionados y documentados**: OpenAPI/Swagger generado desde el código (`utoipa` en Rust), fuente única de verdad para clientes móvil/web.
- **Respuestas de error consistentes**: formato único tipo [RFC 9457 Problem Details](https://www.rfc-editor.org/rfc/rfc9457) (`type`, `title`, `status`, `detail`, `instance`).
- **Filtrado de campos y `Content-Type` estricto** (`application/json` siempre, rechazar otros con `415`).

## 6. Seguridad — OWASP

### OWASP API Security Top 10 (prioritario, dado que el sistema es API-first)

| Riesgo | Mitigación en Línea Correcta |
|---|---|
| API1 Broken Object Level Authorization | Todo acceso a `/expedientes/{id}` valida que el solicitante tenga permiso sobre ese recurso específico, no solo autenticación general |
| API2 Broken Authentication | Verificación de identidad real para aportantes/revisores; tokens de vida corta + refresh; nunca credenciales en query string |
| API3 Broken Object Property Level Authorization | Campos sensibles (ej. datos del aludido antes de publicación) filtrados según rol en el serializer, no en el cliente |
| API4 Unrestricted Resource Consumption | Rate limiting por IP/usuario, límites de tamaño de payload y de adjuntos, timeouts en operaciones de anclaje |
| API5 Broken Function Level Authorization | Endpoints de Areópago (firmar acta) exigen rol `revisor_acreditado` verificado en middleware, no solo en frontend |
| API6 Unrestricted Access to Sensitive Business Flows | Throttling + verificación anti-bot con **HumanGuard** (§15.2) en flujos críticos (registro, aporte de evidencia, creación masiva de expedientes) para prevenir abuso automatizado |
| API7 Server Side Request Forgery | Cualquier URL externa (enlaces de evidencia) se valida contra allowlist de dominios antes de fetch |
| API8 Security Misconfiguration | Headers de seguridad (`Strict-Transport-Security`, `X-Content-Type-Options`), CORS restrictivo, sin `debug mode` en prod |
| API9 Improper Inventory Management | Un solo esquema OpenAPI versionado; endpoints deprecados se retiran, no quedan huérfanos sin documentar |
| API10 Unsafe Consumption of APIs | Datos de SECOP/SIGEP/terceros se validan y sanitizan antes de persistir, tratados como input no confiable |

### Prácticas transversales

- **Autenticación/autorización**: JWT firmado o sesiones opacas + verificación de rol en cada caso de uso (defensa en profundidad, no solo en el gateway).
- **Validación de entrada estricta** en el borde (adaptador HTTP), tipado fuerte de Rust como primera línea de defensa.
- **Prepared statements / ORM parametrizado** (`sqlx`) — cero SQL concatenado.
- **Secrets** fuera del repo (vault/env), rotación periódica de llaves de firma.
- **Logging sin datos sensibles**, con trazabilidad de quién accedió a qué expediente (auditoría, coherente con el propio principio de la plataforma).
- **Dependencias auditadas**: `cargo audit` / `cargo deny` en CI contra CVEs conocidos.
- **HTTPS obligatorio**, HSTS, TLS 1.2+.

## 7. Arquitectura para funcionalidades sociales (comentarios y "me gusta")

El sistema tiene dos naturalezas de datos muy distintas que **no deben vivir en el mismo modelo ni con las mismas garantías**:

| | Núcleo de evidencia | Capa social |
|---|---|---|
| Ejemplos | Expediente, Acta, Firma, Documento | Comentario, Reacción/"me gusta", Seguimiento de caso |
| Mutabilidad | **Append-only, inmutable** una vez sellada el acta | Mutable: se puede editar/borrar (soft-delete) |
| Integridad | Anclada criptográficamente (ver §8) | No requiere anclaje |
| Moderación | Revisión por Areópago (quórum acreditado) | Moderación comunitaria + automática (spam/abuso) |

### Separación en bounded contexts

Se modelan como **dos contextos delimitados independientes** dentro del mismo workspace, comunicados por eventos de dominio, no por acceso directo a tablas:

```
lc-core-evidencia (expedientes, actas, firmas)  --evento: ActaSellada-->  lc-social (comentarios, reacciones, feed)
```

- `lc-social` referencia expedientes/actas **solo por su ID inmutable**; nunca los modifica ni los versiona.
- El feed de actividad y contadores de "me gusta" se construyen con un modelo de lectura separado (proyección/CQRS ligero) para no acoplar el rendimiento de un feed social de alto tráfico con las escrituras críticas de evidencia.
- Los comentarios solo se habilitan **después** de que el acta está sellada (regla de negocio ya definida en la landing: "solo entonces se abre la conversación"), evitando presión social sobre un caso en revisión.

### Consideraciones de diseño específicas

- **Rate limiting y anti-abuso** en likes/comentarios (por IP + por cuenta verificada), reforzado con **HumanGuard** (verificación anti-bot en el momento de comentar/reaccionar, ver §15.2) para prevenir brigading o manipulación de percepción sobre un servidor público mediante cuentas automatizadas.
- **Soft-delete con auditoría**: un comentario borrado se marca `eliminado_en`, no se destruye físicamente — necesario para investigaciones de acoso o difamación cruzada.
- **Umbral de reporte comunitario** que escala a moderación humana, nunca a borrado automático de contenido que mencione hechos (riesgo de censurar denuncias legítimas).
- **Namespacing de autorización**: poder comentar/reaccionar es un permiso de la capa social; poder firmar un acta es un permiso del núcleo — no deben compartir el mismo rol ni middleware de autorización.

## 8. Cadena de evidencia inmutable (blockchain)

Objetivo: que ningún expediente, documento o acta pueda alterarse o desaparecer sin que quede matemáticamente demostrado — incluida la propia plataforma como potencial atacante ("la prueba de que nada fue alterado queda fuera de nuestro alcance", ya declarado en el sitio).

### Diseño en capas

1. **Almacenamiento de contenido direccionado por hash (CAS)**: cada documento adjunto (PDF, sentencia, contrato SECOP) se guarda con su hash SHA-256 como identificador. Igual contenido → igual hash → detección trivial de duplicados o alteraciones.
2. **Event store append-only**: cada cambio de estado de un expediente (creado, evidencia adjunta, firmado, veredicto emitido) se escribe como un evento inmutable en una tabla/log que **nunca se actualiza ni se borra** (a nivel de base de datos: sin `UPDATE`/`DELETE`, permisos revocados para esas operaciones sobre esa tabla).
3. **Árbol de Merkle por lote**: periódicamente (ej. cada N actas u horas) se construye un árbol de Merkle sobre los hashes de los eventos del lote, produciendo un `merkle_root` único.
4. **Anclaje en cadena pública**: ese `merkle_root` se publica en una blockchain pública (ej. vía OpenTimestamps sobre Bitcoin, o un contrato simple en una red de bajo costo tipo Polygon) — esto es lo único que sale a una blockchain, **no los documentos ni datos personales** (evita el problema irreversible de poner datos personales en un registro que no admite "derecho al olvido").
5. **Verificación pública**: cualquiera puede recalcular el hash de un documento/evento, reconstruir su prueba de Merkle contra el `merkle_root` anclado, y confirmar la integridad sin confiar en Línea Correcta.

### Por qué NO todo va a blockchain

- Guardar datos personales o el contenido de expedientes on-chain sería **irreversible y en conflicto directo con la Ley 1581 de 2012** (derecho a rectificación/supresión de datos, §10) — de ahí que solo se ancla el *resumen criptográfico* (hash), nunca el dato.
- El estado transaccional operativo (borradores, comentarios, sesiones) sigue en Postgres; blockchain se usa exclusivamente como **notario de integridad de lo ya sellado**, no como base de datos primaria.

### Puertos involucrados (coherente con §2)

- `trait AnclajeService { fn anclar(&self, merkle_root: Hash) -> AnclaTx; fn verificar(&self, tx: AnclaTx) -> bool; }` — el dominio depende de esta interfaz, no del proveedor de blockchain concreto, permitiendo cambiar de red de anclaje sin tocar casos de uso.

## 9. Testing

Estrategia de pruebas alineada con las capas hexagonales: cada capa se prueba con la herramienta adecuada a su nivel de acoplamiento. Dada la naturaleza del producto (evidencia legal, criptografía, datos personales sensibles), la pirámide de pruebas estándar no basta — se complementa con mutación, fuzzing, caos y pruebas de cumplimiento como ciudadanos de primera clase, no como extras opcionales.

### 9.1 Pirámide base

| Capa | Tipo de prueba | Herramienta / enfoque | Qué cubre |
|---|---|---|---|
| `domain/` | **Unitarias puras** | `#[test]` estándar de Rust, sin mocks | Reglas de negocio: quórum mínimo de firmas, transiciones válidas de veredicto (§16.1), invariantes de `Acta` |
| `domain/` (crypto) | **Property-based testing** | `proptest` | Firma/verificación Ed25519, construcción de Merkle tree — propiedades como "verificar(firmar(x)) siempre es true" para cualquier input |
| `application/` | **Unitarias con dobles** | Fakes/mocks de los traits (`FakeExpedienteRepository` en memoria) | Casos de uso orquestando el dominio correctamente, sin tocar red ni DB |
| `adapters-db/` | **Integración** | `testcontainers` (Postgres real en Docker) | Queries reales, migraciones, constraints de integridad referencial |
| `adapters-http/` | **Contract testing** | Generación desde OpenAPI (`utoipa` + `schemathesis` o similar) | La API cumple su propio contrato documentado; detecta breaking changes antes de que rompan al cliente móvil/web |
| API completa | **E2E** | Cliente HTTP contra el binario levantado en CI (`reqwest` + entorno docker-compose) | Flujos completos: crear expediente → adjuntar evidencia → firmar → sellar acta → verificar anclaje → generar queja disciplinaria (§16.6) |

### 9.2 Seguridad y autorización (más allá del checklist OWASP de §6)

- **Tests de autorización negativos explícitos** por cada regla de §6 (ej. "usuario sin rol revisor no puede firmar", "usuario A no puede leer datos de propiedad-nivel de usuario B") — API1/API3/API5 del OWASP API Top 10; tan obligatorios como los tests funcionales, no un anexo.
- **Fuzzing** (`cargo-fuzz` / `afl.rs`) sobre cualquier parser de entrada no confiable: payloads JSON de eventos, documentos SECOP/SIGEP ingeridos (§16.2), y muy especialmente la deserialización de firmas y estructuras criptográficas — es la superficie más común de vulnerabilidades de memoria/lógica incluso en Rust (parsers con lógica manual siguen siendo falibles aunque el lenguaje evite corrupción de memoria).
- **Tests de la máquina de estados del `Expediente`** (§16.1): además de los unitarios, un test exhaustivo que enumera todas las transiciones posibles del enum y confirma que las no listadas explícitamente **no compilan** — convierte la garantía legal en una prueba de tipo, no solo de comportamiento en runtime.

### 9.3 Mutation testing — probar que las pruebas realmente prueban algo

- **`cargo-mutants`** sobre `domain/` y `adapters-crypto/`: introduce mutaciones automáticas al código (invertir un `if`, cambiar `<` por `<=`, eliminar una línea) y verifica que al menos un test falle. Una alta cobertura de líneas (§9.5) no garantiza que los tests detecten errores reales — mutation testing sí lo mide directamente.
- Aplicado con prioridad a la lógica de verificación de firmas y construcción del Merkle tree: es exactamente el código donde un test "que pasa pero no prueba nada" tendría el mayor costo posible (una firma inválida aceptada como válida).

### 9.4 Chaos engineering y resiliencia

- **Inyección de fallos controlada** en staging: caída súbita del worker de anclaje (§8, §13.1) a mitad de un lote, pérdida de conexión a Postgres durante una escritura de evento, timeout del proveedor de blockchain — confirma que el patrón outbox/at-least-once (§13.1) realmente recupera sin duplicar ni perder eventos.
- **Simulacros de restauración de backup** (no solo verificar que el backup se generó, sino restaurarlo completo en un entorno aislado y validar integridad) con cadencia trimestral — un backup nunca probado no es un backup, es una suposición.
- **Prueba de "servidor comprometido"**: ejercicio periódico que confirma en un entorno de staging que, aun con acceso total a la base de datos, no es posible forjar una firma válida — validación práctica y recurrente de la garantía de §15.1, no solo diseño en papel.

### 9.5 Cobertura, calidad y accesibilidad de clientes

- El dominio (`domain/`) exige **cobertura cercana al 100%** por ser lógica crítica y sin dependencias externas que dificulten testear.
- Los casos de uso (`application/`) se prueban con fakes, no con la base de datos real — mantiene el test suite rápido (segundos, no minutos) y ejecutable en cada commit.
- **Pruebas de accesibilidad (WCAG 2.1 AA)** en web y móvil — `axe-core` integrado en los tests E2E de frontend, y verificación manual con lector de pantalla en los flujos críticos (consultar expediente, aportar evidencia, firmar réplica). Es una plataforma de participación ciudadana; excluir usuarios con discapacidad visual o motora contradice su propósito, no es un "nice to have" de UI.
- **Pruebas de regresión visual** (Playwright/Percy) específicamente sobre la vista pública de un expediente y su corrección/retractación (§16.3) — la regla de "misma prominencia visual" de §10.3 solo se puede garantizar en el tiempo con una prueba automática que falle si alguien reduce accidentalmente el tamaño del bloque de corrección en un refactor de CSS.

### 9.6 Datos de prueba seguros para privacidad (Ley 1581)

- **Nunca** usar datos reales de SIGEP/SECOP ni de usuarios reales en entornos de staging/desarrollo — fixtures sintéticas generadas (nombres, cédulas y cargos ficticios pero con la misma forma/distribución estadística que los datos reales) para que los tests de clasificación de campos (§16.2) sean representativos sin exponer datos personales fuera de producción.
- Entornos de staging con la misma política de acceso restringido que producción para cualquier dato que sí provenga de una fuente real (ej. muestras anonimizadas para depurar un bug reportado) — un incidente de privacidad en "solo staging" sigue siendo un incidente bajo la Ley 1581.

### 9.7 Pruebas de cumplimiento legal como suite propia

Un conjunto de tests explícitamente vinculado a §16, no mezclado con los tests funcionales genéricos, para que una revisión legal pueda auditar "qué prueba cada garantía":

- Un expediente con `Sensibilidad::Sensible` en cualquier campo **nunca** aparece en la proyección de lectura pública (10.1/16.2).
- Una corrección (`ActaRetractada`) se sirve por el mismo endpoint y con el mismo peso de payload/template que el señalamiento original (10.3/16.3).
- Un hecho marcado `HechoExcluidoPorCosaJuzgada` sin interés legítimo documentado no aparece en ninguna proyección (10.3/16.3).
- El hash recalculado de un evento con serialización canónica (§16.7) coincide siempre con el `hash_evento` almacenado — test de reproducibilidad que un perito judicial podría replicar literalmente.
- El worker de anonimización (§16.9) desindexa el contenido personal de las proyecciones públicas sin alterar el `merkle_root` ya anclado — prueba directa de que "restricción de circulación con conservación interna" (T-125/25) funciona como se prometió.

### 9.8 Pruebas continuas en producción

- **Monitoreo sintético**: transacciones canario ejecutadas cada pocos minutos contra producción (crear expediente de prueba en un tenant de test, firmarlo, verificar que se ancla) — detecta degradación real antes de que un usuario la reporte.
- **Despliegues canario/progresivos** (§12.3): un cambio se expone primero a un porcentaje pequeño de tráfico con monitoreo activo de tasa de error antes de completar el rollout — especialmente en el core de firmas/anclaje, donde un bug en producción tiene costo legal, no solo técnico.
- **Feature flags** (ya definidos en §12.3) permiten desactivar una funcionalidad social defectuosa sin un rollback completo del binario.

### 9.9 Reglas de calidad en CI

- CI bloquea merge si: fallan tests, `cargo audit`/`cargo deny` reporta CVE crítico sin mitigar, cobertura de `domain/` cae por debajo del umbral fijado, o el score de mutation testing (§9.3) sobre `adapters-crypto/` baja del umbral acordado.
- Los tests de integración/E2E corren en CI con contenedores efímeros, nunca contra ambientes compartidos.
- La suite de cumplimiento legal (§9.7) corre en cada PR que toque `domain/`, `adapters-crypto/` o las proyecciones de lectura — no solo en un pipeline nocturno, porque una regresión ahí es simultáneamente un bug y una exposición legal.

## 10. Marco legal aplicable (Colombia)

Investigación normativa profundizada (cuatro líneas de investigación independientes: habeas data, responsabilidad por contenido de terceros, valor probatorio de blockchain/firma electrónica, y transparencia/veedurías) que condiciona directamente decisiones de arquitectura y producto. No sustituye asesoría legal formal antes de lanzar a producción — varios puntos aquí (p. ej. si constituirse como veeduría formal, o contratar una entidad de certificación) son decisiones de negocio con implicación legal que un abogado debe validar.

### 10.1 Habeas data — dato público del cargo vs. dato reputacional generado por la plataforma

- **Decreto 1377/2013, art. 3**: el hecho de ser servidor público (cargo, entidad, dependencia) **es un dato público** de tratamiento libre. Pero las denuncias, evidencia y calificaciones de conducta que Línea Correcta produce **no son "datos públicos por naturaleza"** — son datos reputacionales nuevos generados por la plataforma, y requieren su propia base de legitimación (no autorización del denunciado, sino interés legítimo de veeduría + garantías de veracidad).
- **Precedente directo y grave**: la SIC sancionó en 2024 (Resolución 43121) a una plataforma de "listas negras" de contratistas con **multa de $190.547.400 COP y suspensión de 6 meses** por circular datos sin autorización, no garantizar veracidad de lo reportado por usuarios, y no permitir a los afectados conocer/suprimir su información — **la SIC no reconoció excepción de interés público**. Este es el precedente más cercano al modelo de negocio de Línea Correcta y debe leerse como advertencia directa, no como referencia lejana.
- **Excepción periodística (art. 2 Ley 1581)**: bases de datos con fines editoriales/periodísticos quedan fuera del ámbito general de la ley, pero **los principios (veracidad, acceso, circulación restringida) igual aplican**. Vale la pena que la plataforma defina y documente explícitamente si se posiciona como medio editorial para efectos de esta excepción — con asesoría legal, dado que esa calificación también pesa en el análisis de "medio de comunicación" del punto 10.2.
- **Reconciliación inmutabilidad ↔ supresión (Sentencia T-125/25)**: la Corte distingue **supresión total** de **restricción de circulación con conservación interna** (anonimización/desindexación pública, manteniendo archivo interno). El criterio no es solo el paso del tiempo sino la "**caducidad funcional**" — cuando el dato deja de cumplir su finalidad. Esto valida directamente el diseño ya definido en §8: anclar solo el hash (nunca el dato personal) permite desindexar/anonimizar el contenido visible sin romper la cadena de integridad.
- **Registro Nacional de Bases de Datos (RNBD)**, Decreto 1074/2015 cap. 26: obligatorio para toda entidad de naturaleza pública y para privadas con activos > 100.000 UVT — Línea Correcta probablemente califica. Plazo: registrar la base dentro de los 2 meses de creada, y actualizar cada año entre el 2 de enero y el 31 de marzo.

**Implicaciones de arquitectura:**
- Pipeline de ingesta con **clasificación de campos en el origen** (no como moderación reactiva): dato funcional del cargo (libre) vs. dato evaluativo/denuncia (requiere flujo de verificación del Areópago antes de publicarse) vs. dato sensible (art. 18 Ley 1712, bloqueado — ver 10.4).
- Módulo de **verificación de veracidad obligatorio** antes de publicar cualquier señalamiento — no opcional; es precisamente lo que la SIC sancionó por ausencia.
- Mecanismo de acceso/corrección/supresión para el servidor señalado, con SLA de respuesta, y política de "caducidad funcional" que dispare anonimización automática (ej. tras resolución del caso o prescripción disciplinaria) — aunque el hash permanezca anclado.
- Registro en el RNBD desde el lanzamiento, con recordatorio de actualización anual (ene-mar) en el calendario operativo del equipo.

### 10.2 Responsabilidad por comentarios de terceros (capa social, §7)

- Colombia **no tiene un safe harbor legislado**; el régimen es jurisprudencial (Corte Constitucional, sentencias **T-241/23, T-289/23, T-061/24, T-256/25**). Regla central: la plataforma **no responde** por contenido de terceros salvo que (a) intervenga directamente en su creación/edición, o (b) desatienda una **orden judicial expresa** de retiro. En T-241/23 la Corte exoneró a Meta precisamente porque existían mecanismos de reporte y la plataforma no editorializó el contenido.
- **Tutela contra particulares (Decreto 2591/1991, art. 42.7)**: procede contra medios de comunicación cuando el afectado ya solicitó rectificación sin respuesta en condiciones de equidad. La jurisprudencia (T-546/10, T-004/22, T-593/17) ha extendido este tratamiento a plataformas digitales. **Línea Correcta, al publicar expedientes de conducta con curaduría editorial (el Areópago verifica y sella), puede ser calificada como "medio de comunicación"** para este efecto — cuanto más editorial sea la curaduría, mayor la exposición directa a tutela.

**Implicaciones de arquitectura:**
- **Canal de reporte/denuncia interno visible con SLA de respuesta** — es el hecho concreto que exoneró a Meta en T-241/23; su ausencia es el principal factor de riesgo.
- **No editorializar comentarios de usuarios** más allá de moderación por políticas neutrales (spam, amenazas) — intervenir sustancialmente en el contenido rompe la exención jurisprudencial.
- **Flujo técnico de retiro rápido ante orden judicial**, documentado y con dueño operativo claro — es el único supuesto claro donde la plataforma sí responde por inacción.
- El flujo de **"réplica del servidor" debe operar también como canal formal de rectificación previa** (art. 42.7): ofrecer al funcionario la oportunidad de corregir antes de que pueda tutelar, y dejar registro auditable de que se le dio esa oportunidad — blindaje procesal directo.

### 10.3 Injuria, calumnia y defensas del Código Penal (arts. 220-225, Ley 599/2000)

- **Art. 224 — exceptio veritatis**: no hay responsabilidad si se prueba la veracidad de lo imputado. Límite: no puede probarse verdad sobre hechos ya juzgados con sentencia absolutoria ejecutoriada, salvo interés legítimo documentado.
- **Art. 225 — retractación**: exime de responsabilidad si el autor se retracta voluntariamente **antes de sentencia de primera instancia**, publicada con visibilidad equivalente a la imputación original.
- Ambos artículos, junto con los 220-221 ya declarados constitucionales en **C-487/2023**, son la base legal directa del diseño de "acta sellada" + "réplica" ya definido en el producto — no son solo antecedente, son **el fundamento de la defensa legal de la plataforma**.

**Implicaciones de arquitectura:**
- El **"acta sellada" debe diseñarse explícitamente como repositorio probatorio reutilizable en juicio** para sostener la exceptio veritatis: fecha, fuente, cadena de custodia verificable, capacidad de excluir automáticamente hechos ya juzgados con sentencia absolutoria salvo interés legítimo documentado por el Areópago.
- Implementar un **mecanismo de retractación editorial real** que replique el efecto del art. 225: si un expediente resulta inexacto, la corrección se publica con la **misma prominencia visual** que la publicación original (coherente con la jurisprudencia de rectificación en equidad ya citada) — y antes de cualquier escalamiento judicial, no después.

### 10.4 Transparencia, datos abiertos y veedurías (Ley 1712/2014, Ley 850/2003, Ley 1474/2011, CGD)

- **Ley 1712/2014**: la información funcional del cargo (cargo, entidad, contratos, sanciones ejecutoriadas) es pública por defecto (principio de máxima publicidad, art. 3). Pero hay **dos excepciones distintas y no intercambiables**: **art. 18** (información clasificada — daño a terceros: datos sensibles no ligados a la función) y **art. 19** (información reservada — daño al interés público: investigaciones disciplinarias/penales *en curso* sin medida de aseguramiento en firme).
- **Licencia de datos abiertos** de SIGEP II/SECOP II/datos.gov.co: tipo atribución, permite uso comercial implícito, pero exige **citar fuente y fecha de la versión consultada**, y traslada al reutilizador la responsabilidad por el uso del dato (la entidad no garantiza vigencia/calidad).
- **Ley 850/2003 (veedurías ciudadanas)**: da legitimación reforzada (derechos de petición prioritarios, legitimación para acciones constitucionales) a veedurías formalmente constituidas — no es obligatoria para operar, pero es una decisión de negocio a evaluar: constituir una red de veedurías territoriales asociada a la plataforma refuerza la posición legal sin convertir la plataforma tecnológica misma en el sujeto colectivo formal.
- **Código General Disciplinario (Ley 1952/2019), art. 86**: la acción disciplinaria se activa también por **queja de cualquier persona**, sin formalidad especial — un expediente de Línea Correcta puede convertirse directamente en insumo de queja disciplinaria ante Procuraduría/Personerías.

**Implicaciones de arquitectura:**
- Motor de clasificación de campos por tipo (funcional / sensible art. 18 / en investigación no ejecutoriada art. 19, con disclaimer de presunción de inocencia) — mismo pipeline de ingesta de 10.1, un solo sistema de clasificación cubre habeas data y transparencia a la vez.
- **Atribución obligatoria y versionado con fecha de extracción** en cada registro republicado (cumple la licencia abierta y sustenta la defensa de "dato tomado de fuente oficial en su momento").
- Botón/flujo de **"generar queja disciplinaria"** que estructura la evidencia en formato compatible con el art. 86 CGD para envío directo a Procuraduría/Personería — funcionalidad separada del "veredicto" propio de la plataforma, y de bajo costo relativo de implementar con alto valor legal/de producto.
- Separación visual estricta entre **dato oficial verificado** y **comentario/opinión de usuario** — mezclar ambos vicia la presunción de veracidad de la fuente pública y aumenta el riesgo del punto 10.2.

### 10.5 Valor probatorio de firmas Ed25519 y anclaje blockchain

- **Ley 527/1999, art. 7 vs. art. 28-29**: una firma Ed25519 sin entidad certificadora acreditada **sí es válida** como "firma electrónica" (art. 7, no requiere certificadora), pero no alcanza el estatus de "firma digital" reforzada (art. 28-29) que otorga **presunción legal de autoría e integridad**. Sin esa presunción, la fuerza probatoria queda a valoración libre del juez.
- **CGP arts. 244-247, 269-270**: los mensajes de datos se presumen auténticos, pero deben aportarse **en su formato nativo de generación** (no solo el PDF renderizado) para valoración plena (art. 247); la contraparte puede "tachar de falso" el acta, y sin certificación reforzada la carga de sustentar técnicamente la integridad (vía peritaje) recae en Línea Correcta.
- **Ley 2213/2022**: extiende el uso permanente de TIC en actuaciones judiciales, pero es procedimental — no crea un régimen probatorio propio para blockchain, remite al mismo estándar del art. 11 Ley 527.
- **No hay jurisprudencia colombiana consolidada sobre blockchain como prueba** (tema "en desarrollo" según doctrina 2022-2024). Por analogía, jurisprudencia española (STS 326/2019, SAP Vitoria-Gasteiz 1302/2021) sí ha admitido blockchain como prueba documental, siempre que la integridad se acredite mediante **peritaje técnico** — la "inmutabilidad" no se presume automáticamente ante un juez.
- **9 entidades de certificación acreditadas por ONAC/SIC existen en Colombia** (Certicámara, Camerfirma Colombia, GSE, Andes SCD, entre otras).

**Implicaciones de arquitectura:**
- Conservar y exponer el acta en su **formato nativo firmado** (JSON estructurado), no solo como PDF — necesario para valoración plena bajo art. 247 CGP.
- **Documentar y hacer reproducible el proceso de hashing/Merkle** (algoritmo exacto, orden de hojas, librería usada) para poder sostener una tacha de falsedad sin depender solo de testimonio.
- Registrar **metadatos de "generación confiable"** por acta: timestamp del sistema, identidad verificada del revisor, cadena de custodia — insumos que exige directamente el art. 11 Ley 527.
- **Evaluar contratar una entidad de certificación acreditada (ONAC/SIC)** para los revisores del Areópago — migrar sus firmas del régimen de "firma electrónica simple" (sin presunción legal) al de "firma digital" certificada (con presunción de autoría e integridad) reduce sustancialmente la carga probatoria en un eventual proceso judicial o disciplinario. Es la mejora de mayor apalancamiento legal disponible sobre el diseño actual.
- **Preparar un dictamen pericial estándar reutilizable** (o convenio con perito informático) que explique el anclaje blockchain ante un juez, y **exponer un verificador público independiente** (nodo/explorador de bloques accesible a terceros) — refuerza el estándar de "confiabilidad" exigido por el art. 11 Ley 527.

### 10.6 Resumen de acciones legales de mayor apalancamiento

| Prioridad | Acción | Sustento |
|---|---|---|
| Crítica | Módulo de verificación de veracidad antes de publicar (ya cubierto en diseño, pero debe ser innegociable) | Precedente SIC Res. 43121/2024 (sanción directa a plataforma similar) |
| Crítica | Canal de reporte interno + réplica como rectificación previa formal | T-241/23, art. 42.7 Decreto 2591/1991 — reduce exposición a tutela |
| Alta | Registrar la base de datos en el RNBD desde el lanzamiento | Decreto 1074/2015, obligación con plazo de 2 meses |
| Alta | Evaluar entidad de certificación acreditada para firmas del Areópago | Máximo apalancamiento probatorio disponible (art. 28-29 Ley 527) |
| Media | Botón de "generar queja disciplinaria" (art. 86 CGD) | Bajo costo de implementación, alto valor legal y de producto |
| Media | Motor único de clasificación de campos (funcional/sensible/reservado) | Reconcilia Ley 1581 (10.1) y Ley 1712 (10.4) con un solo pipeline |
| Continua | Mantener asesoría legal externa activa, no solo este documento | Varios puntos (veeduría formal, calificación como medio editorial) son decisiones de negocio con efecto jurídico |

## 11. Modelo de datos y eventos

Decisión central: **event sourcing solo en el núcleo de evidencia**; CRUD convencional (con outbox de eventos) en la capa social. No se aplica event sourcing a todo el sistema — sería sobre-ingeniería para comentarios y "me gusta", que no necesitan reconstrucción histórica ni auditoría legal.

### 11.1 Núcleo de evidencia: event sourcing real

El estado de un `Expediente` no se guarda como una fila mutable; se **deriva** de la secuencia de eventos que le ocurrieron. Esto da la inmutabilidad y trazabilidad que exige §8 (cadena de evidencia) y §10 (valor probatorio) de forma nativa, no como un añadido.

```sql
-- Tabla append-only. Sin UPDATE ni DELETE: se revocan esos privilegios a nivel de rol de DB.
CREATE TABLE evento_expediente (
    id              BIGSERIAL PRIMARY KEY,
    expediente_id   UUID NOT NULL,
    tipo            TEXT NOT NULL,        -- 'ExpedienteCreado' | 'EvidenciaAdjuntada' | 'FirmaAplicada' | 'ActaSellada'
    version         INT  NOT NULL,        -- versión secuencial del agregado, para optimistic concurrency
    payload         JSONB NOT NULL,       -- datos del evento, tipado en Rust vía enum serde
    hash_evento     BYTEA NOT NULL,       -- sha256(payload || hash_evento_anterior) — encadenamiento tipo blockchain local
    hash_anterior   BYTEA,                -- referencia al evento previo del mismo expediente
    actor_id        UUID NOT NULL,        -- quién generó el evento (ciudadano, revisor, sistema)
    ocurrido_en     TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (expediente_id, version)
);
CREATE INDEX idx_evento_expediente_id ON evento_expediente (expediente_id, version);
```

- **Encadenamiento local previo al anclaje**: cada evento incluye el hash del evento anterior del mismo expediente (patrón hash-chain, igual que un mini-blockchain interno). El `merkle_root` de §8 se construye sobre lotes de estos hashes ya encadenados — así hay dos capas de integridad: intra-expediente (hash chain) e inter-lote (Merkle + anclaje externo).
- **Reconstrucción del estado actual**: el agregado `Expediente` (dominio) se reconstruye con un `fold` sobre sus eventos (`eventos.iter().fold(Expediente::vacio(), Expediente::aplicar)`), función pura, 100% testeable sin DB (coherente con §9).
- **Proyecciones de lectura** (read models) materializadas en tablas normales (`expediente_actual`, `historial_cargo`) que se actualizan de forma asíncrona al aplicar cada evento — evita reconstruir el fold completo en cada `GET`.
- **Optimistic concurrency**: escribir un evento exige conocer la `version` esperada; un conflicto (dos firmas simultáneas) se resuelve con reintento a nivel de aplicación, no con locks pesimistas de DB.

### 11.2 Capa social: CRUD + outbox

Comentarios y reacciones son de alto volumen y bajo riesgo legal individual — usar event sourcing ahí sería complejidad sin beneficio. Se modelan como tablas mutables normales:

```sql
CREATE TABLE comentario (
    id              UUID PRIMARY KEY,
    expediente_id   UUID NOT NULL,         -- referencia al agregado del núcleo, solo por ID
    autor_id        UUID NOT NULL,
    contenido       TEXT NOT NULL,
    creado_en       TIMESTAMPTZ NOT NULL DEFAULT now(),
    eliminado_en    TIMESTAMPTZ            -- soft-delete, nunca DELETE físico (§7)
);

CREATE TABLE reaccion (
    expediente_id   UUID NOT NULL,
    usuario_id      UUID NOT NULL,
    tipo            TEXT NOT NULL DEFAULT 'me_gusta',
    creado_en       TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (expediente_id, usuario_id)  -- un usuario, una reacción por expediente: evita doble conteo
);
```

- **Contadores denormalizados** (`expediente_contadores.likes_count`) actualizados por trigger o worker async — evita `COUNT(*)` costoso en cada render del feed.
- **Patrón outbox** para publicar `ActaSellada` desde el núcleo hacia la capa social (habilita comentarios) sin acoplar una transacción distribuida: el evento se escribe en la misma transacción SQL que sella el acta, y un worker separado lo publica a un bus (ver §13.1) con reintento garantizado ("at-least-once").

### 11.3 Por qué no event sourcing puro en todo el sistema

Event sourcing total añadiría complejidad operativa (proyecciones, replay, versionado de esquemas de evento) donde el negocio no la necesita. La regla aplicada: **event sourcing donde la inmutabilidad y la auditoría son requisito legal/de producto** (expedientes, actas, firmas); **CRUD donde el dato es efímero o de bajo impacto probatorio** (comentarios, likes, sesiones, preferencias de notificación).

## 12. Infraestructura y despliegue

### 12.1 Base de datos

- **Postgres** como almacén primario (event store del núcleo + tablas CRUD de la capa social). Justificación: soporta JSONB para el `payload` de eventos, transacciones ACID reales (críticas para el patrón outbox de §11.2), y es sobradamente suficiente para el volumen esperado — no se justifica un motor especializado de event sourcing (EventStoreDB) para el tamaño de este proyecto.
- **Réplica de solo lectura** para separar tráfico de lectura pesado (feed social, listados públicos) del tráfico de escritura crítico (firmar actas), sin necesidad de un motor CQRS completo.
- **Backups**: `pg_dump` lógico diario + WAL continuo (point-in-time recovery). Retención mínima acorde a obligaciones de auditoría — a definir con el marco legal de §10, pero nunca inferior al tiempo típico de un proceso disciplinario/judicial.
- **Migraciones versionadas** (`sqlx migrate`), aplicadas solo vía CI, nunca manualmente en producción.

### 12.2 Hosting y contenedores

- **Contenedores Docker** para cada binario (`lc-api`, worker de anclaje, worker de outbox) — build reproducible, mismo artefacto de staging a producción.
- **Orquestación**: para el tamaño inicial del proyecto, un PaaS gestionado (Fly.io / Railway / un solo clúster K8s pequeño gestionado) es más pragmático que operar Kubernetes propio desde el día uno. Migrar a K8s cuando el equipo de plataforma y el tráfico lo justifiquen — YAGNI aplicado a infraestructura.
- **Separación de procesos**: la API HTTP, el worker de anclaje a blockchain (§8) y el worker de outbox (§11.2) corren como servicios independientes y escalan por separado — un pico de tráfico social no debe competir por recursos con el proceso que firma/ancla actas.

### 12.3 CI/CD

```
push → lint (clippy) + fmt check → tests (§9) → cargo audit → build imagen →
  → deploy automático a staging → smoke tests E2E → aprobación manual → deploy a producción
```

- **Ningún deploy a producción sin aprobación humana explícita**, dado que el sistema publica señalamientos sobre personas reales (riesgo legal de §10).
- **Feature flags** para activar funciones sociales (comentarios, likes) de forma gradual/por cohortes, separado del deploy del núcleo de evidencia.
- **Migraciones de esquema** como paso propio del pipeline, con verificación de reversibilidad antes de aplicarse.

### 12.4 Observabilidad

- **Logs estructurados** (JSON, `tracing` crate en Rust) correlacionados por `request_id` y, cuando aplique, `expediente_id` — permite reconstruir "quién hizo qué sobre qué expediente" sin exponer datos sensibles en el log mismo (solo IDs, nunca contenido de evidencia o datos personales, coherente con Ley 1581).
- **Métricas** (Prometheus/OpenTelemetry): latencia por endpoint, tasa de error, profundidad de la cola del worker de anclaje, tiempo entre `ActaSellada` y su anclaje efectivo en blockchain (SLA de integridad).
- **Trazas distribuidas** entre API → worker de outbox → worker de anclaje, para diagnosticar cuellos de botella en el pipeline de sellado.
- **Alertas** dedicadas a: fallos de anclaje (integridad de evidencia en riesgo), picos de reportes/moderación en la capa social, intentos de autorización fallidos repetidos (posible ataque, ver OWASP API4/API5 en §6).

## 13. Escalabilidad y rendimiento

El principio general: **el núcleo de evidencia prioriza corrección sobre velocidad; la capa social prioriza velocidad y disponibilidad sobre consistencia estricta**. Son perfiles de carga opuestos y se diseñan por separado.

### 13.1 Colas y mensajería asíncrona

- Bus de eventos ligero (Postgres `LISTEN/NOTIFY` + tabla outbox al inicio; migrar a NATS/Redis Streams si el volumen lo exige) para propagar eventos del núcleo (`ActaSellada`, `EvidenciaAdjuntada`) hacia consumidores: worker de anclaje, worker de proyecciones de lectura, worker de notificaciones.
- **At-least-once delivery** con consumidores idempotentes (cada handler verifica si ya procesó ese `evento.id` antes de aplicar efectos) — evita duplicar anclajes o notificaciones ante reintentos.
- El anclaje a blockchain (§8) es intencionalmente **asíncrono y por lotes**: no bloquea la respuesta HTTP de sellar un acta; el usuario ve "acta sellada" de inmediato y el anclaje se confirma minutos/horas después, comunicado como "verificable" cuando complete.

### 13.2 Caching

- **Cache de lectura** (Redis o in-memory con TTL corto) para: listados de catálogo de cargos (§ landing, casi estático), contadores de likes/comentarios (tolerantes a segundos de desfase), expedientes ya sellados (inmutables → cacheables agresivamente, invalidación solo por ID nuevo, nunca por actualización).
- **Nunca cachear** el estado de un expediente en proceso de firma (evitar mostrar un veredicto que aún puede cambiar) ni nada relacionado con autorización/permisos.
- Los expedientes sellados son un caso ideal de **cache-first con invalidación trivial**: al ser inmutables, no hay problema de coherencia — se cachean por tiempo largo o indefinidamente detrás de un CDN.

### 13.3 CQRS ligero para el feed social

- Comandos (crear comentario, dar like) van directo a Postgres transaccional.
- Consultas de alto tráfico (feed de un expediente, ranking de "más comentados") se sirven desde **proyecciones materializadas** actualizadas de forma asíncrona por los workers de §13.1, no calculadas en vivo con `JOIN`s pesados en cada request.
- Esto es CQRS *ligero* (separación de modelos de lectura/escritura dentro de la misma base de datos), no CQRS con bases de datos distintas — evitar esa complejidad hasta que el volumen real la justifique (mismo principio YAGNI de §12.2).

### 13.4 Manejo de picos ("actas virales")

- Un expediente de alto interés público puede recibir miles de likes/comentarios en minutos. Mitigaciones:
  - Rate limiting por usuario + verificación HumanGuard (§7, §15.2) evita que el pico sea artificial (bots), distinguiéndolo de un pico real de interés ciudadano legítimo.
  - Contadores con incrementos atómicos en cache (Redis `INCR`) sincronizados a Postgres de forma batched, en vez de un `UPDATE` por cada like.
  - CDN + cache de borde para la vista pública del expediente (contenido inmutable, ver 13.2) absorbe la mayor parte del tráfico de lectura sin tocar el backend.
- El tráfico social nunca debe poder saturar la capacidad de cómputo del núcleo de evidencia — reforzado por la separación de procesos de §12.2 (escalan independientemente) y por límites de recursos (cuotas de CPU/memoria) por servicio en el orquestador.

### 13.5 Cuándo NO optimizar todavía

- No introducir sharding de base de datos, microservicios adicionales, ni un bus de mensajería pesado (Kafka) mientras el tráfico proyectado no lo requiera — cada pieza de infraestructura extra es superficie de mantenimiento y de ataque (relevante para OWASP §6). Escalar verticalmente y con las optimizaciones de esta sección cubre razonablemente varios órdenes de magnitud de crecimiento antes de necesitar arquitectura distribuida compleja.

## 14. Decisiones arquitectónicas clave (ADR resumido)

| # | Decisión | Alternativas consideradas | Por qué esta opción |
|---|---|---|---|
| ADR-1 | Arquitectura hexagonal (puertos y adaptadores) | Arquitectura en capas tradicional (MVC monolítico) | El dominio (reglas de un "Poder Moral" ciudadano) debe sobrevivir cambios de framework HTTP o de proveedor de blockchain sin reescribirse; el desacople es requisito, no preferencia estética |
| ADR-2 | Event sourcing solo en el núcleo de evidencia, CRUD en la capa social | Event sourcing global / CRUD global | El núcleo necesita inmutabilidad y auditoría legal (§10); la capa social necesita velocidad y simplicidad. Aplicar una sola estrategia a todo habría sido sobre-ingeniería en un lado o insuficiente en el otro |
| ADR-3 | Solo el `merkle_root` se ancla en blockchain pública; documentos y datos personales quedan en Postgres | Anclar todo el expediente on-chain | Ley 1581/2012 exige poder rectificar/suprimir datos personales — incompatible con un registro verdaderamente inmutable; el hash preserva verificabilidad sin ese conflicto |
| ADR-4 | Bounded contexts separados para evidencia y capa social, comunicados por eventos (outbox) | Un solo modelo de datos compartido | Evita que el alto volumen de likes/comentarios degrade o arriesgue la integridad del núcleo; permite escalar y desplegar cada uno por separado (§12.2, §13) |
| ADR-5 | Postgres único (con réplica de lectura) en vez de motor de event sourcing dedicado o bases separadas por contexto | EventStoreDB, bases de datos poliglotas | El volumen esperado no justifica la complejidad operativa adicional; Postgres cubre event store (JSONB + ACID) y CRUD social igual de bien a esta escala (YAGNI) |
| ADR-6 | PaaS gestionado en vez de Kubernetes propio para el lanzamiento inicial | K8s autogestionado desde el día uno | Menor carga operativa mientras el equipo es pequeño; migrar cuando el tráfico o la organización lo exijan, no antes |
| ADR-7 | Ningún deploy a producción sin aprobación humana | Deploy 100% automático (continuous deployment) | El sistema publica señalamientos sobre personas reales; el riesgo legal (§10, injuria/calumnia) exige un punto de control humano antes de exponer cambios |
| ADR-8 | CQRS ligero (proyecciones dentro de la misma base) para el feed social, no CQRS con bases separadas | CQRS completo con almacenes de lectura/escritura distintos | Resuelve el problema real (lecturas rápidas de feed) sin la complejidad de sincronización entre dos bases de datos distintas — se puede evolucionar a eso después si hace falta |

**Principio transversal que ata todas las decisiones**: cada elección prioriza primero **corrección e integridad legal** en el núcleo de evidencia, y solo después **simplicidad operativa** — nunca al revés. Cuando una decisión de infraestructura entra en conflicto con la garantía de que "nada puede modificarse ni borrarse" en un expediente sellado, gana la garantía.

## 15. Modelo de amenazas y seguridad avanzada

El §6 (OWASP API Top 10) cubre la superficie estándar de una API. Esta sección cubre lo que es **específico del modelo de amenaza de Línea Correcta**: una plataforma que publica señalamientos verificados sobre personas con poder real —incluida fuerza pública y altos cargos— y que por diseño no puede permitirse que un solo servidor comprometido falsifique un veredicto.

### 15.1 Gestión de llaves de firma (Ed25519) — el activo más crítico del sistema

La llave privada de un revisor del Areópago es, en la práctica, más sensible que cualquier dato en la base de datos: quien la controle puede forjar un acta oficial.

- **La plataforma nunca custodia llaves privadas de revisores.** El firmado ocurre en el dispositivo del revisor (hardware wallet, HSM personal, o al menos un enclave seguro del dispositivo) — el backend solo recibe y verifica la firma, nunca la genera ni la almacena. Si un servidor de Línea Correcta se compromete, un atacante no debe poder firmar nada.
- **Ceremonia de generación de llaves** documentada y reproducible (idealmente presencial o con testigos para revisores fundadores), con registro público de la llave pública asociada a cada identidad de revisor desde el día uno.
- **Rotación y revocación**: proceso definido para invalidar la llave de un revisor comprometido, coaccionado o que deja el Areópago — incluye lista de revocación pública, consistente con el registro abierto de revisores ya descrito en el sitio.
- **MFA obligatorio** en la cuenta de plataforma de cada revisor, independiente y más estricto que el login de un ciudadano consultante — comprometer esa cuenta no debe bastar para firmar (la firma criptográfica en el dispositivo del revisor es la segunda barrera real).

### 15.2 Adversario con recursos, no solo atacante oportunista

El modelo de amenaza no es "hacker aleatorio buscando una vulnerabilidad genérica" — incluye actores con incentivo directo para silenciar o manipular la plataforma: el propio funcionario señalado, redes organizadas, o en el peor caso un actor con recursos estatales.

- **Protección de borde en dos capas independientes, no una sola herramienta:**
  - **Capa de red (L3/L4) — DDoS volumétrico y CDN**: sigue siendo necesario un proveedor de borde tipo Cloudflare (o equivalente) delante de toda la infraestructura, para absorber inundaciones de tráfico y servir contenido estático/cacheado cerca del usuario. HumanGuard **no cubre esta capa** — es un servicio de aplicación (L7), no un proveedor de red perimetral.
  - **Capa de aplicación (L7) — detección de bots y abuso automatizado**: aquí se usa **HumanGuard** (producto propio, `api.humanguard.app`) en lugar de un CAPTCHA tradicional, integrado en los puntos de fricción críticos: registro de cuenta, aporte de evidencia, comentarios y "me gusta" (mitiga directamente el riesgo de brigading/manipulación de percepción de §7 y el abuso de flujos de negocio de OWASP API6 en §6). Funciona con challenges gamificados de 3-5s + análisis de comportamiento por IA (ensemble XGBoost+LSTM), sin fricción tipo CAPTCHA clásico, y ya está en producción y validado (Accuracy 99.06%, Recall 100%, FPR 2.4% sobre datos reales).
  - El flujo de verificación: SDK web/Android de HumanGuard emite un `result_token` firmado tras el challenge → el backend de Línea Correcta lo valida vía `POST /v1/verify-token` contra la API de HumanGuard antes de aceptar la acción (registro, comentario, like, aporte de evidencia) — se integra como un adaptador más en `adapters-http/` (§2), detrás de un trait `BotVerificationService`, para poder sustituirlo si hiciera falta sin tocar los casos de uso.
  - Las dos capas son complementarias, no sustitutas: **la capa de red protege la disponibilidad de la infraestructura; HumanGuard protege la integridad del comportamiento humano de la capa social.** Eliminar Cloudflare (o equivalente) sin reemplazo dejaría la plataforma expuesta a DDoS volumétrico, que ningún detector de bots a nivel de aplicación puede mitigar por sí solo.
- **Protección de metadatos de quien aporta evidencia o comenta**: IPs y datos de sesión no deben ser accesibles a todo el equipo interno ni quedar en logs de aplicación — solo a un proceso de seguridad estrictamente necesario, con acceso auditado (principio de mínimo privilegio aplicado también hacia adentro del propio equipo).
- **Política de respuesta a solicitudes legales/gubernamentales de identificación de usuarios**, definida y publicada *antes* de que ocurra una solicitud real — qué se entrega, bajo qué orden judicial, y qué se notifica al usuario afectado si la ley lo permite.
- **Protección reforzada para reporteros/aportantes en riesgo documentado**, coherente con lo ya declarado en el sitio ("la atribución individual puede divulgarse en diferido") — esto debe tener un mecanismo técnico real (ej. desacoplar identidad verificada de identificador público hasta que el riesgo se descarte), no solo una promesa de producto.

### 15.3 Cadena de suministro de software

- **SBOM** (inventario de dependencias, `cargo-sbom` o similar) generado en cada build, no solo `cargo audit` puntual.
- **Firma de artefactos de build** (ej. `cosign`/Sigstore) para que un binario desplegado sea verificablemente el que salió del pipeline de CI, no uno alterado en tránsito.
- **Revisión de salud de dependencias criptográficas**: los crates que implementan Ed25519, Merkle trees y hashing deben ser los de mayor adopción y auditoría en el ecosistema Rust (ej. `ed25519-dalek`), evitando implementaciones propias o de mantenimiento único para primitivas criptográficas — "no ruedes tu propia criptografía".

### 15.4 Insider threat y separación de funciones

El event store append-only (§11) protege contra alteración retroactiva, pero no contra un administrador con acceso legítimo que abusa de ese acceso.

- **Doble control (four-eyes) para operaciones administrativas sensibles**: ninguna acción destructiva o de alto impacto (ej. suspender un expediente, modificar permisos de revisor) debe poder ejecutarla una sola persona sin aprobación de una segunda.
- **Auditoría de acceso administrativo** con el mismo rigor que la auditoría de firmas del Areópago — quién vio o tocó qué expediente, con timestamp, igual de trazable e inmutable.
- **Principio de mínimo privilegio** estricto: acceso de producción (DB, logs con IPs, llaves de infraestructura) limitado al mínimo equipo posible, revisado periódicamente.

### 15.5 Pentesting y respuesta a incidentes

- **Pruebas de penetración periódicas** (al menos anuales, y antes de cualquier cambio mayor al esquema de firmas o anclaje), incluyendo pruebas específicas de los escenarios de §15.1 y §15.2 — no solo un escaneo automatizado de la API.
- **Plan de respuesta a incidentes** documentado: quién se activa, cómo se contiene, cómo se comunica a los usuarios afectados, y en qué plazo — coherente con las obligaciones de notificación de la Ley 1581 de 2012 (§10) ante una brecha de datos personales.
- **Bug bounty o canal de divulgación responsable** una vez el proyecto tenga tracción, para que investigadores de seguridad reporten hallazgos antes de que los exploten terceros.

### 15.6 Resumen de prioridad

| Prioridad | Medida | Por qué es urgente |
|---|---|---|
| Crítica | Firmado de revisores nunca en el servidor (15.1) | Es la garantía central del producto; sin esto, todo lo demás es cosmético |
| Crítica | MFA + rotación/revocación de llaves de revisor (15.1) | Compromiso de una sola cuenta no debe poder forjar un acta oficial |
| Alta | Protección de metadatos de aportantes/reporteros (15.2) | Riesgo de daño físico real a personas, no solo reputacional |
| Alta | CDN/anti-DDoS de red (L3/L4) + HumanGuard para anti-bot de aplicación (L7) (15.2) | La plataforma es un blanco predecible en momentos de mayor impacto noticioso; son dos capas independientes, ninguna sustituye a la otra |
| Media | SBOM y firma de artefactos (15.3) | Reduce riesgo de compromiso silencioso de la cadena de build |
| Media | Doble control administrativo (15.4) | Cierra el último vector de alteración que el event sourcing no cubre por sí solo |
| Continua | Pentesting + plan de incidentes (15.5) | No es una medida única sino un proceso que debe mantenerse vivo |

## 16. Cumplimiento legal — especificaciones técnicas

§10 documentó *qué* exige cada norma. Esta sección traduce cada implicación en una especificación concreta de dominio, esquema o endpoint — para que el cumplimiento sea parte del diseño, no una lista de buenas intenciones aparte del código.

### 16.1 Máquina de estados del `Expediente` — verificación obligatoria antes de publicar (10.1, 10.6)

La ausencia de verificación fue exactamente lo que sancionó la SIC (Res. 43121/2024). Se modela como invariante del dominio, no como convención de proceso:

```rust
enum EstadoExpediente {
    Borrador,                 // solo visible para quien lo crea + Areópago
    EnVerificacion,           // evidencia adjunta, pendiente de quórum de firmas
    Sellado { veredicto: Veredicto },   // publicado, inmutable (§11.1)
    Corregido { motivo: String },       // tras retractación (16.3), republicado con misma prominencia
    Anonimizado,               // tras caducidad funcional (16.2) o solicitud de supresión resuelta
}
```

La transición `Borrador → Sellado` **no existe** en el enum — es estructuralmente imposible saltarse `EnVerificacion` (quórum mínimo de firmas del Areópago, invariante ya definida en §2). El compilador de Rust rechaza cualquier código que intente esa transición directa.

### 16.2 Clasificación de campos en el pipeline de ingesta (10.1, 10.4)

Un único mecanismo de clasificación resuelve a la vez la Ley 1581 (dato personal) y la Ley 1712 (transparencia):

```rust
enum Sensibilidad {
    Funcional,              // cargo, entidad, contrato, sanción ejecutoriada — libre (art. 3 Ley 1712)
    Sensible,                // art. 18 Ley 1712 — nunca se persiste en el expediente público
    ReservadoEnCurso,        // art. 19 Ley 1712 — investigación sin medida de aseguramiento en firme
}
```

- Se aplica **en el paso de ingesta**, antes de que cualquier dato entre a `evento_expediente` (§11.1) — nunca como moderación posterior.
- `Sensibilidad::Sensible` bloquea la ingesta del campo por completo (rechazo automático, no requiere revisión humana).
- `Sensibilidad::ReservadoEnCurso` se persiste pero el adaptador HTTP (`adapters-http`) inyecta obligatoriamente un disclaimer de presunción de inocencia en la respuesta — a nivel de serializer, no de frontend, para que ningún cliente pueda omitirlo.
- Cada documento oficial ingerido añade `fuente: String` y `fecha_extraccion: DateTime` al evento `EvidenciaAdjuntada` (§11.1) — cumple la licencia de atribución de datos.gov.co/SIGEP/SECOP citada en 10.4.

### 16.3 Rectificación, retractación y exclusión por cosa juzgada (10.2, 10.3)

Nuevos tipos de evento en `evento_expediente` (mismo event store de §11.1, misma inmutabilidad):

| Evento | Dispara | Efecto |
|---|---|---|
| `RectificacionSolicitada` | El servidor señalado usa el canal formal de réplica | Inicia el reloj del SLA de respuesta; blindaje procesal del art. 42.7 Decreto 2591/1991 |
| `RectificacionResuelta` | Areópago revisa la solicitud | Puede derivar en `ActaRetractada` o cierre sin cambios, con motivo documentado |
| `ActaRetractada` | Corrección aceptada antes de sentencia de primera instancia (art. 225) | Estado pasa a `Corregido`; **regla de API dura**: el endpoint `GET /expedientes/{id}` sirve la corrección con el mismo template/peso visual que el señalamiento original — no hay una variante "resumida" de una corrección |
| `HechoExcluidoPorCosaJuzgada` | Un hecho coincide con una sentencia absolutoria ejecutoriada (art. 224) | El hecho se excluye del expediente salvo que el Areópago documente interés legítimo explícito |

Esto hace que el "acta sellada" sea, por construcción, el repositorio probatorio que 10.3 exige para sostener la *exceptio veritatis* — no un documento aparte.

### 16.4 Capa social: canal de reporte con SLA (10.2 — blindaje jurisprudencial T-241/23)

Nueva entidad en el contexto social (`domain-social`, CRUD, §11.2 — no requiere event sourcing porque no es evidencia del núcleo):

```sql
CREATE TABLE reporte_contenido (
    id              UUID PRIMARY KEY,
    contenido_id    UUID NOT NULL,          -- comentario o reacción reportado
    motivo          TEXT NOT NULL,
    estado          TEXT NOT NULL DEFAULT 'pendiente',  -- pendiente | resuelto | escalado
    reportado_en    TIMESTAMPTZ NOT NULL DEFAULT now(),
    sla_vencimiento TIMESTAMPTZ NOT NULL,    -- reportado_en + política de SLA configurada
    resuelto_en     TIMESTAMPTZ
);
```

Un worker de monitoreo alerta (§12.4) cuando un reporte se acerca a `sla_vencimiento` sin resolver — la existencia de este canal y su diligencia de respuesta es, según T-241/23, lo que exime a la plataforma de responsabilidad por contenido de terceros.

### 16.5 Retiro por orden judicial con doble control (10.2 + 15.4)

`POST /admin/contenido/{id}/retirar` exige **dos aprobaciones distintas** (four-eyes, §15.4) antes de ejecutarse, y genera su propio evento auditable (`ContenidoRetiradoPorOrdenJudicial`, con referencia al expediente judicial) — es el único supuesto donde la plataforma sí responde por inacción (10.2), así que la ejecución debe ser rápida una vez aprobada, pero nunca unilateral.

### 16.6 Queja disciplinaria estructurada (10.4, art. 86 CGD)

`POST /expedientes/{id}/queja-disciplinaria` — genera un export en el formato que exige el art. 86 (hechos, servidor, entidad, soporte documental) a partir de los eventos ya existentes del expediente sellado, sin duplicar datos: es una **proyección de lectura** más (mismo patrón de §11.1), no una funcionalidad nueva de escritura. Separado explícitamente del "veredicto" propio de la plataforma — la queja no implica que Línea Correcta emita un juicio ante la autoridad disciplinaria, solo estructura la evidencia ya verificada.

### 16.7 Reproducibilidad del hash — serialización canónica (10.5)

Para que un perito o un tercero pueda recalcular el hash de un evento y que coincida (requisito de 10.5 para sostener una tacha de falsedad), el `payload` de cada evento se serializa con **JSON Canónico (RFC 8785 JCS)** antes de aplicar SHA-256 — no el JSON "por defecto" de `serde_json`, cuyo orden de claves no está garantizado entre versiones. Este detalle se documenta como parte del contrato del evento, junto al algoritmo de Merkle (orden de hojas, función de combinación), en el mismo repositorio público que el código (coherente con la promesa de apertura del sitio).

### 16.8 Puerto de firma con soporte a certificación acreditada (10.5)

El trait `FirmaService` (§2, §14 ADR-1) se diseña desde el inicio para soportar dos implementaciones intercambiables, sin reescribir casos de uso si se decide contratar una entidad de certificación ONAC/SIC:

```rust
trait FirmaService {
    fn firmar(&self, payload: &[u8], revisor: &Revisor) -> Firma;
    fn verificar(&self, firma: &Firma) -> ResultadoVerificacion;
    fn nivel_legal(&self) -> NivelFirma;   // FirmaElectronicaSimple (art. 7) | FirmaDigitalCertificada (art. 28-29)
}
```

`ResultadoVerificacion` expone `nivel_legal()` hacia la capa de presentación, para que el expediente muestre honestamente qué peso probatorio tiene cada firma — no todas las firmas del Areópago tienen que migrar a certificación a la vez.

### 16.9 Worker de anonimización por caducidad funcional (10.1, Sentencia T-125/25)

Nuevo binario, junto a los workers ya definidos en §12.2/§16 (estructura de carpetas):

- **`lc-worker-anonimizacion`**: corre periódicamente, evalúa criterios objetivos de caducidad funcional (resolución del caso, prescripción disciplinaria, solicitud de supresión aprobada) y emite el evento `Anonimizado` sobre el expediente correspondiente — desindexa el contenido personal identificable de las proyecciones de lectura públicas (§11.1) **sin alterar ni romper** el event store ni el hash ya anclado en blockchain (§8), porque el hash nunca contuvo el dato personal en primer lugar. Es la implementación literal de "restricción de circulación con conservación interna" que T-125/25 exige como alternativa válida a la supresión total.

### 16.10 Checklist operativo (no es código, pero condiciona el lanzamiento)

- Registro de la base de datos en el RNBD dentro de los 2 meses posteriores al lanzamiento (10.1).
- Aviso de privacidad (contenido mínimo del art. 15, Decreto 1377/2013) integrado en el flujo de onboarding, no solo en una página estática.
- Decisión de negocio, con asesoría legal, sobre constituir una red de veedurías formal (Ley 850/2003) y sobre si la plataforma se autodeclara con fines editoriales (excepción art. 2 Ley 1581) — ninguna de las dos es requisito técnico, pero ambas cambian la superficie legal de todo lo anterior.

## 17. Estructura de carpetas propuesta

Refleja los dos bounded contexts de §7 (núcleo de evidencia vs. capa social), los binarios separados de §12.2 (API + workers) y los adaptadores externos de §8/§15.2/§16.

```
lc-api/                        # workspace Cargo
├── crates/
│   ├── domain/                 # núcleo de evidencia: Expediente (máquina de estados §16.1), Acta, Firma, Veredicto (§2, §11.1)
│   ├── domain-social/          # capa social: Comentario, Reacción, ReporteContenido (§7, §16.4) — crate separado, sin depender de domain/
│   ├── application/            # casos de uso + traits del núcleo (puertos, §2), incl. rectificación/retractación (§16.3) y queja disciplinaria (§16.6)
│   ├── application-social/     # casos de uso de comentarios/reacciones/reportes (§7, §16.4)
│   ├── adapters-http/          # controladores Axum, DTOs, OpenAPI — expone ambos contextos
│   ├── adapters-db/            # PostgresExpedienteRepository, event store (§11.1), repos sociales (§11.2)
│   ├── adapters-crypto/        # firma Ed25519 (FirmaService con nivel_legal(), §16.8), verificación, construcción de Merkle canónica (§16.7)
│   ├── adapters-anclaje/       # AnclajeService: OpenTimestamps/Polygon (§8, §2 puertos de salida)
│   ├── adapters-humanguard/    # BotVerificationService — integración con HumanGuard (§15.2)
│   └── shared/                 # tipos de error comunes (DomainError → ApiError, §4), utils
├── bin/
│   ├── lc-api/                 # binario HTTP principal (Axum)
│   ├── lc-worker-outbox/       # publica eventos del núcleo hacia la capa social (§11.2, §13.1)
│   ├── lc-worker-anclaje/      # construye Merkle por lote y ancla en blockchain (§8, §13.1)
│   └── lc-worker-anonimizacion/ # aplica caducidad funcional / supresión aprobada sobre proyecciones públicas (§16.9)
└── Cargo.toml                  # workspace
```

Los prefijos `domain`/`application` sin sufijo pertenecen siempre al núcleo de evidencia (event-sourced, §11.1); el sufijo `-social` marca explícitamente el contexto CRUD de menor criticidad (§11.2) — así el nombre del crate ya comunica a qué garantías de inmutabilidad está sujeto su contenido.
