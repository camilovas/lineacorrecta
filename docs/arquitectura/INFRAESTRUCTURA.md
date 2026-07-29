# Infraestructura y blockchain — La Línea Correcta

> Extraído de `ARQUITECTURA.md` original. Las referencias `§N` conservan la numeración original del documento monolítico — ver [INDICE.md](../INDICE.md) para ubicar cada sección en su archivo actual.

## 8. Cadena de evidencia inmutable (blockchain)

Objetivo: que ningún expediente, documento o acta pueda alterarse o desaparecer sin que quede matemáticamente demostrado — incluida la propia plataforma como potencial atacante ("la prueba de que nada fue alterado queda fuera de nuestro alcance", ya declarado en el sitio).

### Diseño en capas

1. **Almacenamiento de contenido direccionado por hash (CAS)**: cada documento adjunto (PDF, sentencia, contrato SECOP) se guarda con su hash SHA-256 como identificador. Igual contenido → igual hash → detección trivial de duplicados o alteraciones.
2. **Event store append-only**: cada cambio de estado de un expediente (creado, evidencia adjunta, firmado, veredicto emitido) se escribe como un evento inmutable en una tabla/log que **nunca se actualiza ni se borra** (a nivel de base de datos: sin `UPDATE`/`DELETE`, permisos revocados para esas operaciones sobre esa tabla).
3. **Árbol de Merkle por lote**: periódicamente (ej. cada N actas u horas) se construye un árbol de Merkle sobre los hashes de los eventos del lote, produciendo un `merkle_root` único.
4. **Anclaje en cadena pública**: ese `merkle_root` se publica en una blockchain pública (ej. vía OpenTimestamps sobre Bitcoin, o un contrato simple en una red de bajo costo tipo Polygon) — esto es lo único que sale a una blockchain, **no los documentos ni datos personales** (evita el problema irreversible de poner datos personales en un registro que no admite "derecho al olvido").
5. **Verificación pública**: cualquiera puede recalcular el hash de un documento/evento, reconstruir su prueba de Merkle contra el `merkle_root` anclado, y confirmar la integridad sin confiar en La Línea Correcta.

### Por qué NO todo va a blockchain

- Guardar datos personales o el contenido de expedientes on-chain sería **irreversible y en conflicto directo con la Ley 1581 de 2012** (derecho a rectificación/supresión de datos, §10) — de ahí que solo se ancla el *resumen criptográfico* (hash), nunca el dato.
- El estado transaccional operativo (borradores, comentarios, sesiones) sigue en Postgres; blockchain se usa exclusivamente como **notario de integridad de lo ya sellado**, no como base de datos primaria.

### Redundancia de anclaje — recalibrado tras el panel de revisión (§20)

El hallazgo original (§19.3 escenario 3: una sola red pública es punto único de fallo legal) sigue siendo válido, pero el panel de expertos (§20) recalibró la solución: **para v0, la redundancia no es una segunda blockchain — es un timestamp certificado (TSA/RFC 3161)** como segunda capa, mucho más barato operativamente y con reconocimiento legal más directo en Colombia ("fecha cierta") que una segunda red pública. La segunda blockchain completa queda en `BACKLOG.md` (Sprint 5+), no en el diseño base de lanzamiento.

- **Sprint 1-4**: una sola blockchain pública (ej. OpenTimestamps sobre Bitcoin) + un sello de tiempo TSA como respaldo barato.
- **Backlog (Sprint 5+)**: agregar una segunda red pública como capa adicional de redundancia, cuando el volumen y el riesgo real lo justifiquen — el trait ya está diseñado para soportarlo sin romper nada (ver abajo).

### Puertos involucrados (coherente con §2)

- `trait AnclajeService { fn anclar(&self, merkle_root: Hash) -> Result<Vec<AnclaTx>, AnclajeError>; fn verificar(&self, txs: &[AnclaTx]) -> Result<bool, AnclajeError>; }` — devuelve/verifica un `AnclaTx` por cada red configurada (una sola en v0, más si se agregan después); el `Result` explícito refleja que anclar en una red externa puede fallar (timeout, red caída) — coherente con la regla de "manejo de excepciones sin pánico" de §6.5/OWASP A10. El dominio depende de esta interfaz, no de los proveedores concretos, permitiendo agregar redes de anclaje sin tocar casos de uso — la implementación de v0 registra un solo proveedor (blockchain) + un adaptador TSA; agregar una segunda blockchain en Sprint 5+ es solo registrar un segundo adaptador, no un cambio de contrato.


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

- Bus de eventos ligero (Postgres `LISTEN/NOTIFY` + tabla outbox al inicio; migrar a NATS/Redis Streams si el volumen lo exige) para propagar eventos del núcleo (`ActaSellada`, `EvidenciaAdjuntada`) hacia consumidores: worker de anclaje, worker de proyecciones de lectura, worker de notificaciones. **Aclaración de implementación (hallazgo del panel, §20)**: `NOTIFY` es *best-effort* — si no hay listener conectado en el momento exacto, la notificación se pierde silenciosamente. La **tabla outbox es la única fuente de verdad**; `NOTIFY` es solo una optimización de latencia para evitar polling constante. Todo worker debe hacer polling de respaldo de la tabla outbox (ej. cada pocos segundos) independientemente de si recibió el `NOTIFY` — nunca asumir que `NOTIFY` por sí solo garantiza entrega.
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


