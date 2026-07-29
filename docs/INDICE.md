# Índice de arquitectura — La Línea Correcta

El documento de arquitectura se dividió en archivos por dominio para que se pueda leer, mantener y delegar a revisión (humana o de agentes) por partes, sin cargar las ~20 secciones completas cada vez. Las referencias `§N` dentro de cada archivo conservan la numeración original del documento monolítico — este índice te dice en qué archivo vive cada sección.

| Archivo | Carpeta | Contiene (§ original) | Tema |
|---|---|---|---|
| [ARQUITECTURA-CORE.md](arquitectura/ARQUITECTURA-CORE.md) | `arquitectura/` | §1-5 | Visión general, arquitectura hexagonal, SOLID, DRY, buenas prácticas de API REST |
| [MODELO-DATOS-EVENTOS.md](arquitectura/MODELO-DATOS-EVENTOS.md) | `arquitectura/` | §7, §11 | Bounded contexts (núcleo vs. capa social), event sourcing, esquema de eventos |
| [INFRAESTRUCTURA.md](arquitectura/INFRAESTRUCTURA.md) | `arquitectura/` | §8, §12, §13 | Cadena de evidencia/blockchain, despliegue, escalabilidad y rendimiento |
| [ESTRUCTURA.md](arquitectura/ESTRUCTURA.md) | `arquitectura/` | §17 | Estructura de carpetas — v0 (módulos) y evolución futura (crates) |
| [SEGURIDAD.md](seguridad/SEGURIDAD.md) | `seguridad/` | §6, §15 | OWASP (API Top 10, Top 10:2025, CI/CD, ASVS 5.0), modelo de amenazas avanzado, HumanGuard |
| [TESTING.md](testing/TESTING.md) | `testing/` | §9 | Estrategia de pruebas (unitarias, mutation testing, chaos engineering, cumplimiento legal) |
| [LEGAL.md](legal/LEGAL.md) | `legal/` | §10, §16 | Marco legal aplicable (Colombia) y sus especificaciones técnicas de cumplimiento |
| [ESCENARIOS-LEGALES.md](legal/ESCENARIOS-LEGALES.md) | `legal/` | §19 | 32 escenarios legales simulados, con estado cubierto/brecha |
| [DECISIONES.md](decisiones/DECISIONES.md) | `decisiones/` | §14, §20 | ADRs resumidos + panel de revisión de expertos (discusión y conclusiones) |
| [SPRINT-1.md](sprints/SPRINT-1.md) | `sprints/` | — | Plan accionable del primer sprint (`SPRINT-2.md`, etc. se agregan aquí a medida que avanza el proyecto) |
| [CHECKLIST-PREVIO.md](CHECKLIST-PREVIO.md) | *(raíz de `docs/`)* | §18 | Checklist previo a comenzar la programación |
| [BACKLOG.md](BACKLOG.md) | *(raíz de `docs/`)* | — | Todo lo diferido tras el panel de revisión, priorizado y con su porqué |

## Cómo navegar esto

- **¿Vas a tocar código de dominio/eventos?** Empieza por `arquitectura/ARQUITECTURA-CORE.md` + `arquitectura/MODELO-DATOS-EVENTOS.md`.
- **¿Vas a revisar seguridad o hacer threat modeling?** `seguridad/SEGURIDAD.md`.
- **¿Necesitas justificar una decisión ante un abogado o ante ti mismo antes de publicar algo?** `legal/LEGAL.md` + `legal/ESCENARIOS-LEGALES.md`.
- **¿Vas a planear el próximo sprint?** `sprints/SPRINT-1.md` para el patrón ya usado, `BACKLOG.md` para qué sigue — el siguiente sprint se agrega como `sprints/SPRINT-2.md`, y así sucesivamente.
- **¿Quieres el resumen de por qué se decidió lo que se decidió?** `decisiones/DECISIONES.md` — incluye la discusión completa del panel de 6 expertos y las tensiones que resolvieron.

## Convención de carpetas

Cada dominio que puede crecer con el tiempo tiene su propia carpeta (`sprints/`, `legal/`, `arquitectura/`, `seguridad/`, `testing/`, `decisiones/`) — nuevos archivos del mismo tipo se agregan ahí (ej. `sprints/SPRINT-2.md`, `legal/CONCEPTO-ABOGADO-2026-08.md`, `decisiones/ADR-009-*.md` si en algún momento se prefiere un ADR por archivo). `BACKLOG.md`, `CHECKLIST-PREVIO.md` e `INDICE.md` se quedan en la raíz de `docs/` porque son documentos únicos, no series.

## Visión general (resumen de §1)

Backend en **Rust** (Axum), expuesto como **API REST**, con app móvil y web como clientes. El sistema gestiona expedientes ciudadanos de conducta de servidores públicos, actas de veredicto firmadas (Ed25519) y su anclaje criptográfico en una cadena pública. Arquitectura hexagonal, SOLID/DRY, OWASP, event sourcing en el núcleo de evidencia, capa social separada, marco legal colombiano integrado al diseño técnico — no como documento aparte.

Para el detalle completo de cualquier tema, entra al archivo correspondiente en la tabla de arriba.
