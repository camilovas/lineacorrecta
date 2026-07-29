# Decisiones arquitectónicas y panel de revisión — La Línea Correcta

> Extraído de `ARQUITECTURA.md` original. Las referencias `§N` conservan la numeración original del documento monolítico — ver [INDICE.md](../INDICE.md) para ubicar cada sección en su archivo actual.

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


## 20. Panel de revisión de expertos — discusión, tensiones y conclusión

Seis revisiones independientes (arquitecto Rust/event sourcing, seguridad ofensiva red-team, segunda opinión legal colombiana, sistemas distribuidos/datos, pragmatismo de producto para equipos pequeños, y metodología Scrum aplicada) sobre el documento completo — no para validarlo, sino para encontrarle fallas reales. Cada uno trabajó de forma independiente, sin ver las conclusiones de los demás.

### 20.1 Tensiones encontradas entre expertos (la discusión)

| Tensión | Posiciones en conflicto | Resolución aplicada |
|---|---|---|
| **Anclaje redundante: ¿sobre-ingeniería o necesario?** | Arquitecto Rust y pragmatismo de producto: una sola blockchain basta para v0. Sistemas distribuidos: el riesgo de fork/discontinuación es real, no lo recortaría | **Ninguno de los dos tenía razón completa**: se ancla en una sola blockchain + un timestamp certificado TSA/RFC 3161 como redundancia barata (§8) — resuelve el riesgo real sin el costo de una segunda blockchain completa. La segunda blockchain queda en `BACKLOG.md` |
| **Cuánto recortar para el MVP** | Pragmatismo de producto: recortar casi todo el andamiaje (event sourcing, ASVS 3, ONAC, mutation/chaos/fuzzing, 9+ crates). Seguridad: encontró huecos reales (ingeniería social contra el Areópago, evidencia falsificada con apariencia oficial) que el recorte no toca | Se distingue entre recortes de **infraestructura cara** (aceptados: menos crates, un solo anclaje, sin ASVS 3 aún) y **controles de proceso casi gratis** (no se recortan: verificación cruzada de evidencia contra fuente oficial, protocolo de firma con confirmación visible) — ver `BACKLOG.md` para el detalle de qué se prioriza |
| **Fragmentación en crates** | Arquitecto Rust y pragmatismo de producto: 9+ crates es topología de equipo grande, ralentiza a un equipo pequeño. Documento original: crates separados desde el diseño | Módulos dentro de un solo crate para v0 (§17), con los mismos nombres/límites que tendrían los crates futuros — migración mecánica cuando el equipo o el tráfico lo justifiquen, no un rediseño |
| **Event sourcing: ¿completo desde el día 1?** | Pragmatismo y arquitecto Rust: demasiado para v0. Sistemas distribuidos: el hash-chain + tabla append-only es barato y de alto valor legal, no es lo mismo que el "event sourcing completo" (fold + proyecciones + CQRS) que sí es prematuro | Se separan dos cosas que el documento original mezclaba: **la tabla append-only con hash-chain SÍ es el diseño de v0** (no es un recorte, es correcto desde el inicio); **el fold completo con proyecciones asíncronas y CQRS ligero sí se difiere** a cuando el tráfico lo justifique (§13.3, backlog) |
| **"Medio editorial" vs. "medio de comunicación"** | Legal (segunda opinión): son mutuamente excluyentes, no una decisión pendiente — el documento original las trataba como si se pudiera "elegir bien" con más análisis | **Resuelto como gate de negocio, cerrado antes de Sprint 1** (§10.1, §10.2): no se reclama la excepción editorial; se asume la calificación de "medio de comunicación" y se convierte en fortaleza, porque el flujo de réplica (§16.3) ya satisface lo que esa calificación exige |

### 20.2 Convergencias inesperadas (dos expertos, sin coordinarse, llegaron a lo mismo)

- **Seguridad y Legal** señalaron independientemente el mismo punto ciego: la **actuación coordinada inauténtica** (brigading) contra un acta es el vector de ataque más barato y probable, y hoy no tiene ni diseño técnico ni respuesta legal (§19.2 escenario 3). Es la única brecha de los 32 escenarios simulados que sigue genuinamente abierta — máxima prioridad en `BACKLOG.md`.
- **Seguridad y Legal** también coincidieron en que depender de HumanGuard (producto propio y joven, sin historial de ataques adversariales documentado) como capa crítica de defensa es un riesgo sistémico sin plan de contingencia — falta definir política de fail-open/fail-closed si HumanGuard falla o es degradado (backlog).

### 20.3 Hallazgos técnicos puntuales aplicados directamente

- `trait AnclajeService` no manejaba fallos explícitamente (violaba la propia regla de A10/§6.5 de no usar pánico sin control) — corregido a `Result` en §8.
- `Postgres LISTEN/NOTIFY` es *best-effort*; el documento no aclaraba que la tabla outbox debe ser la fuente de verdad — aclarado en §13.1.

### 20.4 Roadmap de sprints — de menor a mayor, sin rediseño futuro

Conclusión del experto en Scrum: la arquitectura hexagonal ya elegida **sí soporta bien** "simple ahora, sofisticado después sin rearquitecturar", siempre que los puertos/traits queden definidos completos desde el Sprint 1 aunque su implementación sea mínima. Plan completo, priorizado y accionable: ver `SPRINT-1.md` (fundación) y `BACKLOG.md` (todo lo demás, con el sprint estimado en que se retomaría cada ítem).

Dos advertencias del experto que si no se atienden en Sprint 1 sí forzarían un rediseño más adelante:
1. El esquema de `payload JSONB` de los eventos debe reservar desde Sprint 1 el campo de llave de cifrado por expediente (para el crypto-shredding de §10.1), aunque esa funcionalidad no se active hasta más adelante — agregarlo después invalidaría los hashes ya anclados.
2. Los módulos de dominio (no crates, ver §17) deben mantener desde Sprint 1 los mismos límites y nombres que tendrían como crates futuros — fusionarlos "para ir rápido" ahora y separarlos después sí sería el rediseño que este roadmap busca evitar.

