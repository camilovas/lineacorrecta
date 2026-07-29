# Sprint 1 — La Línea Correcta

Primer sprint accionable. Objetivo: un incremento de software funcional real (principio Scrum), levantando la fundación sobre la que todo lo demás se construye sin rearquitecturar (ver `ARQUITECTURA.md` §20.4). Referencias entre paréntesis apuntan a la sección de `ARQUITECTURA.md` que sustenta cada decisión.

## Gate de negocio — se cierra antes de escribir código de dominio

- [ ] **Confirmar la decisión legal "medio de comunicación", no "medio editorial"** (§10.1, §10.2, §20.1) — ya resuelta y documentada en `ARQUITECTURA.md`; este ítem es solo la validación final con un abogado colombiano real antes de que el diseño de moderación/rectificación se dé por definitivo. Condiciona el campo `fuente_tipo` del serializer (§10.4) y el tono de moderación de comentarios desde el día 1.

## Objetivo funcional del sprint

Como ciudadano/Areópago, puedo: crear un expediente, adjuntar evidencia clasificada, y consultarlo por API — con el hash de integridad ya calculado (aunque el anclaje externo llegue en Sprint 4).

## Backlog del sprint

### Núcleo de dominio (`domain`, módulo — no crate, ver §17)

- [ ] `enum EstadoExpediente { Borrador, EnVerificacion, Sellado { veredicto } }` (§16.1) — las variantes `Corregido`, `OcultoPorOrdenJudicial`, `Anonimizado` se agregan cuando el caso de uso correspondiente se implemente (Sprint 4+), no antes; el enum crece con el producto, no se anticipa vacío.
- [ ] `enum Veredicto { Validado, NoConcluyente, Desvirtuado }`
- [ ] `enum Sensibilidad { Funcional, Sensible, ReservadoEnCurso }` (§16.2) — variante `ReservadoSeguridadNacional` se agrega cuando se conecte una fuente del sector Fuerza Pública (backlog), no en Sprint 1.
- [ ] Invariante: quórum mínimo de firmas para transición a `Sellado` (§2, §3-L).
- [ ] Tests unitarios puros del dominio (§9.1) — sin mocks, sin DB. Cobertura cercana al 100% desde el primer commit, no como meta futura.

### Esquema de datos (`adapters/db`, Postgres)

- [ ] Tabla `evento_expediente` **append-only con hash-chain** (§11.1, §20.4) — sin fold completo ni proyecciones asíncronas todavía; el estado actual se actualiza síncronamente en la misma transacción.
- [ ] **Campo reservado para llave de cifrado por-expediente** en el `payload` desde este sprint, aunque el crypto-shredding no se active hasta que ocurra el escenario real (§10.1, advertencia de §20.4 — evita romper hashes ya anclados más adelante).
- [ ] Revocar privilegios `UPDATE`/`DELETE` sobre `evento_expediente` a nivel de rol de DB desde el primer despliegue, no como hardening posterior.
- [ ] Migraciones versionadas (`sqlx migrate`) desde el commit inicial.

### Verificación de veracidad (no negociable, §10.1, §16.1)

- [ ] Caso de uso `CrearExpediente` → estado `Borrador`.
- [ ] Caso de uso `AdjuntarEvidencia` con clasificación de `Sensibilidad` en el momento de ingesta (no como moderación posterior) — campo `Sensibilidad::Sensible` bloquea la ingesta de ese campo automáticamente.
- [ ] Caso de uso `IniciarVerificacion` → `EnVerificacion` (requiere evidencia adjunta).
- [ ] Endpoint/checklist manual para que el Areópago confirme quórum — sin ML ni automatización de verificación todavía; es un checklist humano vía API.

### API (`adapters/http`)

- [ ] `POST /expedientes`, `GET /expedientes/{id}`, `POST /expedientes/{id}/evidencia` (§5).
- [ ] Versionado `/api/v1/...` desde el día uno (§5).
- [ ] OpenAPI generado desde código (`utoipa`) desde el primer endpoint, no agregado después (§5).
- [ ] Errores en formato RFC 9457 Problem Details desde el primer handler (§5).

### CI/CD y calidad (§6.3, §18.4)

- [ ] Branch protection en `main` desde el primer commit — sin push directo, revisión obligatoria.
- [ ] `cargo audit`/`cargo deny` + `clippy::unwrap_used` denegado en CI desde el primer pipeline, no agregado después.
- [ ] `rustfmt.toml`/`clippy.toml` commiteados antes del primer PR de feature.

### Explícitamente FUERA de Sprint 1 (ver `BACKLOG.md`)

- Firma Ed25519 real y sellado → **Sprint 2**.
- Canal de reporte/rectificación con SLA, trámite RNBD → **Sprint 2**.
- Capa social, HumanGuard, anclaje real en blockchain → **Sprint 3-4**.
- Cualquier cosa marcada 🟡 o 🟢 en `BACKLOG.md`.

## Definición de "hecho" del sprint (§18.5)

- Tests unitarios + integración pasan; cobertura de `domain` dentro del umbral.
- Sin advertencias de `clippy`; sin secretos en el diff.
- El expediente creado en `Borrador` no puede alcanzar `Sellado` sin pasar por `EnVerificacion` con quórum — verificado con un test explícito que confirma que la transición directa no compila/no es representable (§9.2).
- OpenAPI actualizado y coincide con los endpoints reales (contract test básico).
