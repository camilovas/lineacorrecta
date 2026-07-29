# Arquitectura núcleo — La Línea Correcta

> Extraído de `ARQUITECTURA.md` original. Las referencias `§N` conservan la numeración original del documento monolítico — ver [INDICE.md](../INDICE.md) para ubicar cada sección en su archivo actual.

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
- **`GET /expedientes/{id}/prueba-merkle`** — endpoint público, sin autenticación, que expone el hash del expediente, su posición en el árbol de Merkle del lote y el/los `AnclaTx` correspondientes (§8), para que cualquier tercero (periodista, perito, ciudadano) pueda verificar la integridad sin confiar en la plataforma ni requerir soporte del equipo — cierra la brecha entre la promesa de "verificación pública" de §8.5 y un contrato de API concreto (§16.11).


