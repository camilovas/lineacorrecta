# Modelo de datos y eventos — La Línea Correcta

> Extraído de `ARQUITECTURA.md` original. Las referencias `§N` conservan la numeración original del documento monolítico — ver [INDICE.md](../INDICE.md) para ubicar cada sección en su archivo actual.

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


