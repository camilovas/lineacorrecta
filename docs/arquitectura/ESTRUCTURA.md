# Estructura de carpetas — La Línea Correcta

> Extraído de `ARQUITECTURA.md` original. Las referencias `§N` conservan la numeración original del documento monolítico — ver [INDICE.md](../INDICE.md) para ubicar cada sección en su archivo actual.

## 17. Estructura de carpetas — v0 (módulos) y evolución futura (crates)

Recalibrado tras el panel de revisión (§20): la fragmentación en 9+ crates independientes es prematura para el tamaño de equipo real. **Para v0 (Sprint 1 en adelante), los mismos límites de dominio se mantienen como módulos dentro de un único crate/binario** — el límite de dependencia (regla de §2: adapters → application → domain, nunca al revés) se respeta igual de estrictamente con `pub(crate)`/módulos que con crates separados; lo único que cambia es el costo de recompilación y el ritual de versionado entre crates, que no se justifica todavía.

```
lc-api/                        # un solo crate Cargo para v0
├── src/
│   ├── domain/                 # núcleo de evidencia: Expediente (máquina de estados §16.1), Acta, Firma, Veredicto (§2, §11.1)
│   ├── domain_social/          # capa social: Comentario, Reacción, ReporteContenido (§7, §16.4) — módulo separado, sin depender de domain/
│   ├── application/            # casos de uso + traits del núcleo (puertos, §2), incl. rectificación/retractación (§16.3)
│   ├── application_social/     # casos de uso de comentarios/reacciones/reportes (§7, §16.4)
│   ├── adapters/
│   │   ├── http/                # controladores Axum, DTOs, OpenAPI — expone ambos contextos
│   │   ├── db/                  # PostgresExpedienteRepository, event store (§11.1), repos sociales (§11.2)
│   │   ├── crypto/               # firma Ed25519, verificación, Merkle canónica (§16.7)
│   │   ├── anclaje/              # AnclajeService: blockchain + TSA (§8 recalibrado)
│   │   └── humanguard/           # BotVerificationService — integración con HumanGuard (§15.2, desde Sprint 3)
│   └── shared/                  # tipos de error comunes (DomainError → ApiError, §4), utils
├── bin/
│   ├── lc-api.rs                 # binario HTTP principal (Axum) — Sprint 1
│   ├── lc-worker-outbox.rs       # publica eventos del núcleo hacia la capa social (§11.2, §13.1) — Sprint 3
│   └── lc-worker-anclaje.rs      # construye Merkle por lote y ancla (§8, §13.1) — Sprint 4
└── Cargo.toml
```

**Backlog (ver `BACKLOG.md`)**: split a workspace multi-crate (`domain`, `domain-social`, `application`, `application-social`, `adapters-http`, `adapters-db`, `adapters-crypto`, `adapters-anclaje`, `adapters-humanguard`, `shared` como crates independientes) y `lc-worker-anonimizacion` como binario propio — se activa cuando el equipo crezca o cuando la velocidad de compilación de un solo crate empiece a doler, no antes. El nombrado con sufijo `_social` para el contexto de menor criticidad se conserva igual en la versión de módulos, para que el día de la migración a crates sea un `cargo` refactor mecánico, no un rediseño.


