# Marco legal — La Línea Correcta

> Extraído de `ARQUITECTURA.md` original. Las referencias `§N` conservan la numeración original del documento monolítico — ver [INDICE.md](../INDICE.md) para ubicar cada sección en su archivo actual.

## 10. Marco legal aplicable (Colombia)

Investigación normativa profundizada (cuatro líneas de investigación independientes: habeas data, responsabilidad por contenido de terceros, valor probatorio de blockchain/firma electrónica, y transparencia/veedurías) que condiciona directamente decisiones de arquitectura y producto. No sustituye asesoría legal formal antes de lanzar a producción — varios puntos aquí (p. ej. si constituirse como veeduría formal, o contratar una entidad de certificación) son decisiones de negocio con implicación legal que un abogado debe validar.

### 10.1 Habeas data — dato público del cargo vs. dato reputacional generado por la plataforma

- **Decreto 1377/2013, art. 3**: el hecho de ser servidor público (cargo, entidad, dependencia) **es un dato público** de tratamiento libre. Pero las denuncias, evidencia y calificaciones de conducta que La Línea Correcta produce **no son "datos públicos por naturaleza"** — son datos reputacionales nuevos generados por la plataforma, y requieren su propia base de legitimación (no autorización del denunciado, sino interés legítimo de veeduría + garantías de veracidad).
- **Precedente directo y grave**: la SIC sancionó en 2024 (Resolución 43121) a una plataforma de "listas negras" de contratistas con **multa de $190.547.400 COP y suspensión de 6 meses** por circular datos sin autorización, no garantizar veracidad de lo reportado por usuarios, y no permitir a los afectados conocer/suprimir su información — **la SIC no reconoció excepción de interés público**. Este es el precedente más cercano al modelo de negocio de La Línea Correcta y debe leerse como advertencia directa, no como referencia lejana.
- **Excepción periodística (art. 2 Ley 1581) — DECISIÓN RESUELTA (gate de negocio, cerrado antes de Sprint 1):** la plataforma **no se autodeclara medio editorial** para efectos de esta excepción. Razón: un segundo dictamen legal (panel de revisión, ver §20) encontró que esto no era realmente una decisión libre de la plataforma — un juez califica por **sustancia** (¿hay curaduría editorial real?), no por la etiqueta que la plataforma se ponga a sí misma. Dado que el Areópago verifica y sella veredictos (comportamiento sustantivamente editorial de todas formas), reclamar la excepción del art. 2 no evita la exposición de §10.2, solo le da munición adicional a un juez de tutela para tratar a la plataforma como "medio de comunicación" — sin ganar realmente el beneficio de la excepción, porque los principios de veracidad/acceso/circulación igual aplican aunque se reclame. **Resolución**: La Línea Correcta cumple la Ley 1581 en su régimen general y completo, sin invocar la excepción — y en vez de resistir la calificación de "medio de comunicación" para efectos de tutela (§10.2), **la asume y la convierte en fortaleza**: el flujo de réplica del servidor (§16.3) ya implementa exactamente lo que el art. 42.7 exige (oportunidad de rectificación previa), así que estar sujeta a ese régimen no es una debilidad nueva, es un requisito que el producto ya satisface por diseño desde el Sprint 2. Sujeto a confirmación final de abogado colombiano antes de publicar el primer expediente real (§18.6), pero no bloquea el diseño ni la planeación de sprints.
- **Reconciliación inmutabilidad ↔ supresión (Sentencia T-125/25)**: la Corte distingue **supresión total** de **restricción de circulación con conservación interna** (anonimización/desindexación pública, manteniendo archivo interno). El criterio no es solo el paso del tiempo sino la "**caducidad funcional**" — cuando el dato deja de cumplir su finalidad. Esto valida directamente el diseño ya definido en §8: anclar solo el hash (nunca el dato personal) permite desindexar/anonimizar el contenido visible sin romper la cadena de integridad.
- **Base legal de retención frente a una solicitud de supresión total** (art. 8 Ley 1581): el derecho de supresión cede ante un "deber legal o contractual de conservación" — el valor probatorio del expediente (§10.5, exceptio veritatis del art. 224 CP) **es** esa base legal, y debe invocarse explícitamente como defensa, no darse por sobreentendida. Aun así, debe existir una **válvula de escape excepcional** para el caso límite en que un juez ordene supresión real (no solo anonimización) de un campo específico: el `payload` de ese campo se cifra con una llave dedicada por expediente desde el origen, de forma que "suprimir" en ese caso extremo significa destruir la llave de cifrado (crypto-shredding) — el evento y su hash permanecen intactos en el event store (no se viola el append-only de §8), pero el contenido queda irrecuperable. Es un mecanismo de último recurso, invocable solo bajo orden judicial explícita y con doble aprobación (§15.4), no una alternativa cotidiana a la anonimización.
- **Registro Nacional de Bases de Datos (RNBD)**, Decreto 1074/2015 cap. 26: **obligatorio y bloqueante, con plazo legal perentorio** (no una mejora discrecional) para toda entidad de naturaleza pública y para privadas con activos > 100.000 UVT — La Línea Correcta probablemente califica. Plazo: registrar la base dentro de los 2 meses de creada, y actualizar cada año entre el 2 de enero y el 31 de marzo. Es el mismo régimen bajo el cual se sancionó el precedente citado arriba — no lanzar sin esto resuelto.

**Implicaciones de arquitectura:**
- Pipeline de ingesta con **clasificación de campos en el origen** (no como moderación reactiva): dato funcional del cargo (libre) vs. dato evaluativo/denuncia (requiere flujo de verificación del Areópago antes de publicarse) vs. dato sensible (art. 18 Ley 1712, bloqueado — ver 10.4).
- Módulo de **verificación de veracidad obligatorio** antes de publicar cualquier señalamiento — no opcional; es precisamente lo que la SIC sancionó por ausencia.
- Mecanismo de acceso/corrección/supresión para el servidor señalado, con SLA de respuesta, y política de "caducidad funcional" que dispare anonimización automática (ej. tras resolución del caso o prescripción disciplinaria) — aunque el hash permanezca anclado.
- Registro en el RNBD desde el lanzamiento, con recordatorio de actualización anual (ene-mar) en el calendario operativo del equipo.
- **Flujo de habeas data separado del flujo de rectificación editorial** (ver §16.3): acceso, actualización, supresión y revocación de autorización (art. 8 Ley 1581) son derechos distintos de la réplica del art. 42.7, con SLA legal propio (10 días hábiles para consulta + 5 de prórroga, 15 días hábiles para reclamo + 8 de prórroga, art. 14 Decreto 1377/2013) — no deben modelarse como el mismo evento.

### 10.2 Responsabilidad por comentarios de terceros (capa social, §7)

- Colombia **no tiene un safe harbor legislado**; el régimen es jurisprudencial (Corte Constitucional, sentencias **T-241/23, T-289/23, T-061/24, T-256/25**). Regla central: la plataforma **no responde** por contenido de terceros salvo que (a) intervenga directamente en su creación/edición, o (b) desatienda una **orden judicial expresa** de retiro. En T-241/23 la Corte exoneró a Meta precisamente porque existían mecanismos de reporte y la plataforma no editorializó el contenido.
- **Tutela contra particulares (Decreto 2591/1991, art. 42.7)**: procede contra medios de comunicación cuando el afectado ya solicitó rectificación sin respuesta en condiciones de equidad. La jurisprudencia (T-546/10, T-004/22, T-593/17) ha extendido este tratamiento a plataformas digitales. **La Línea Correcta, al publicar expedientes de conducta con curaduría editorial (el Areópago verifica y sella), puede ser calificada como "medio de comunicación"** para este efecto. **Decisión resuelta (ver §10.1)**: la plataforma no pelea esta calificación — la asume desde el diseño, porque el costo de intentar evitarla (reclamar la excepción editorial de Ley 1581) es mayor que el de simplemente cumplir de entrada el requisito que esa calificación implica (rectificación previa, ya cubierto por el flujo de réplica de §16.3).

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
- **Código General Disciplinario (Ley 1952/2019), art. 86**: la acción disciplinaria se activa también por **queja de cualquier persona**, sin formalidad especial — un expediente de La Línea Correcta puede convertirse directamente en insumo de queja disciplinaria ante Procuraduría/Personerías.

**Implicaciones de arquitectura:**
- Motor de clasificación de campos por tipo (funcional / sensible art. 18 / en investigación no ejecutoriada art. 19, con disclaimer de presunción de inocencia) — mismo pipeline de ingesta de 10.1, un solo sistema de clasificación cubre habeas data y transparencia a la vez.
- **Atribución obligatoria y versionado con fecha de extracción** en cada registro republicado (cumple la licencia abierta y sustenta la defensa de "dato tomado de fuente oficial en su momento").
- Botón/flujo de **"generar queja disciplinaria"** que estructura la evidencia en formato compatible con el art. 86 CGD para envío directo a Procuraduría/Personería — funcionalidad separada del "veredicto" propio de la plataforma, y de bajo costo relativo de implementar con alto valor legal/de producto.
- Separación visual estricta entre **dato oficial verificado** y **comentario/opinión de usuario** — mezclar ambos vicia la presunción de veracidad de la fuente pública y aumenta el riesgo del punto 10.2. Especificación técnica concreta (antes huérfana, ahora cerrada): la proyección de lectura pública (`expediente_actual`, §11.1) incluye un campo `fuente_tipo: FuncionalOficial | EvidenciaVerificada | ComentarioUsuario` en cada bloque de contenido, y `adapters-http` rechaza en tiempo de serialización cualquier respuesta que combine bloques de distinto `fuente_tipo` en una misma estructura sin el campo discriminador explícito — la regla vive en el contrato de API, no solo en el diseño visual del frontend.

### 10.5 Valor probatorio de firmas Ed25519 y anclaje blockchain

- **Ley 527/1999, art. 7 vs. art. 28-29**: una firma Ed25519 sin entidad certificadora acreditada **sí es válida** como "firma electrónica" (art. 7, no requiere certificadora), pero no alcanza el estatus de "firma digital" reforzada (art. 28-29) que otorga **presunción legal de autoría e integridad**. Sin esa presunción, la fuerza probatoria queda a valoración libre del juez.
- **CGP arts. 244-247, 269-270**: los mensajes de datos se presumen auténticos, pero deben aportarse **en su formato nativo de generación** (no solo el PDF renderizado) para valoración plena (art. 247); la contraparte puede "tachar de falso" el acta, y sin certificación reforzada la carga de sustentar técnicamente la integridad (vía peritaje) recae en La Línea Correcta.
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

Recalibrado tras auditoría: las obligaciones legales con plazo perentorio no comparten nivel con las mejoras discrecionales, aunque ambas fueran "Alta" en una versión anterior de esta tabla.

| Prioridad | Acción | Sustento |
|---|---|---|
| **Crítica — bloqueante** | Módulo de verificación de veracidad antes de publicar (ya cubierto en diseño, pero debe ser innegociable) | Precedente SIC Res. 43121/2024 (sanción directa a plataforma similar) |
| **Crítica — bloqueante** | Canal de reporte interno + réplica como rectificación previa formal | T-241/23, art. 42.7 Decreto 2591/1991 — reduce exposición a tutela |
| **Crítica — bloqueante, plazo legal duro** | Registrar la base de datos en el RNBD **dentro de los 2 meses posteriores al lanzamiento** | Decreto 1074/2015 — mismo régimen bajo el cual se sancionó el precedente de Res. 43121/2024; no es una mejora, es una obligación con fecha límite |
| **Crítica — bloqueante** | Válvula de escape de supresión total (crypto-shredding) para orden judicial excepcional (§10.1) | Art. 8 Ley 1581 — sin esto, una orden judicial de supresión real no tiene cómo cumplirse sin violar el append-only del event store |
| Alta — no bloqueante | Evaluar contratar entidad de certificación acreditada para firmas del Areópago | Máximo apalancamiento probatorio disponible (art. 28-29 Ley 527), pero es una mejora incremental, no un requisito para operar desde el día uno |
| Media | Botón de "generar queja disciplinaria" (art. 86 CGD) | Bajo costo de implementación, alto valor legal y de producto |
| Media | Motor único de clasificación de campos (funcional/sensible/reservado) | Reconcilia Ley 1581 (10.1) y Ley 1712 (10.4) con un solo pipeline |
| Continua | Mantener asesoría legal externa activa, no solo este documento | Varios puntos (veeduría formal, calificación como medio editorial) son decisiones de negocio con efecto jurídico |

Simulación exhaustiva de escenarios que sustentan y amplían esta tabla: ver §19. Especificaciones técnicas de las brechas cerradas: ver §16.11.


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
    OcultoPorOrdenJudicial { referencia_proceso: String },  // medida cautelar provisional (§16.11) — desindexado, NO implica que el contenido era incorrecto
    Anonimizado,               // tras caducidad funcional (16.2) o solicitud de supresión resuelta
}
```

`OcultoPorOrdenJudicial` es distinto de `Corregido`: una corrección implica que el señalamiento era incorrecto (art. 225 CP); un ocultamiento judicial provisional es una medida cautelar que puede revertirse si el juez la levanta — de ahí que sea un estado propio y no una variante de `Corregido`. En ambos casos el evento subyacente permanece en el event store (§8); lo único que cambia es la proyección de lectura pública.

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

`POST /expedientes/{id}/queja-disciplinaria` — genera un export en el formato que exige el art. 86 (hechos, servidor, entidad, soporte documental) a partir de los eventos ya existentes del expediente sellado, sin duplicar datos: es una **proyección de lectura** más (mismo patrón de §11.1), no una funcionalidad nueva de escritura. Separado explícitamente del "veredicto" propio de la plataforma — la queja no implica que La Línea Correcta emita un juicio ante la autoridad disciplinaria, solo estructura la evidencia ya verificada.

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
- **Cláusulas Contractuales Tipo (CCT) de la Red Iberoamericana** con cualquier proveedor cloud fuera de Colombia (Circular Externa 003/2025 y 002/2025 SIC) antes de almacenar cualquier dato personal fuera del territorio — obligatorio si se usa AWS/GCP/Azure en regiones no colombianas, documentado en el aviso de privacidad.
- **Runbook de notificación de brecha con reloj de 15 días hábiles** (Circular Única SIC, Título V) desde la detección del incidente — vinculado operativamente al plan de respuesta a incidentes de §15.5, no solo mencionado ahí en abstracto.
- **Convenio permanente con perito informático** para atender tachas de falsedad (§10.5) y protocolo escrito de quién en el equipo lo activa y en qué plazo.
- **Anexo de procedimiento para cartas rogatorias / asistencia judicial internacional**: ninguna entrega de datos a autoridad extranjera sin trámite formal vía Cancillería (canal `judicial@cancilleria.gov.co`) — extensión del protocolo de solicitudes legales ya exigido en §15.2, específica para el canal internacional.
- **Cláusula de indemnidad (hold harmless) + seguro de responsabilidad civil profesional** para los revisores del Areópago que actúan dentro del protocolo — decisión contractual, no técnica, pero condiciona si la plataforma puede reclutar revisores dispuestos a firmar con su identidad real.
- **Política de neutralidad editorial en período electoral**: criterios objetivos y documentados de timing de publicación (aplicados igual antes/durante/después de campaña), más un disclaimer explícito de que un expediente no constituye propaganda de La Línea Correcta cuando es citado por terceros en campaña — mitiga el riesgo de que el CNE o un tutelante aleguen interferencia electoral (§19.4, escenario 4).

### 16.11 Brechas de diseño cerradas tras la simulación de escenarios (§19)

Cada uno de estos elementos nace de un hallazgo concreto de la simulación — no son mejoras genéricas, cierran un escenario específico documentado en §19.

- **`SolicitudHabeasData { tipo: Acceso | Actualizacion | Supresion | Revocacion }`** — nuevo tipo de evento en el núcleo (§11.1), separado de `RectificacionSolicitada`/`RectificacionResuelta` (§16.3), con su propio reloj de SLA (10 días hábiles + 5 de prórroga para consulta; 15 + 8 para reclamo, art. 14 Decreto 1377/2013) y su propia alerta operativa, con el mismo rigor que ya tenía `reporte_contenido` (§16.4) — cierra el hallazgo de auditoría #3 y #7, y el escenario 1 de §19.1.
- **`IdentidadAportanteProtegida`** — evento que anonimiza la proyección pública de la identidad de un aportante sin romper el vínculo interno auditable que el Areópago necesita para sostener veracidad (art. 224 CP) — cierra §19.1 escenario 2.
- **Campo `rol_mencion: Aludido | Aportante | TerceroIncidental`** en el clasificador de §16.2 — el enum `Sensibilidad` hoy solo distingue tipo de dato, no el rol de quien aparece mencionado; un testigo o familiar incidental (§19.1 escenario 8) requiere minimización aunque el dato en sí no sea técnicamente "sensible".
- **Subcategoría `Sensibilidad::ReservadoSeguridadNacional`**, distinta de `ReservadoEnCurso`, para datos del sector Fuerza Pública (art. 19 Ley 1712 en su vertiente de defensa/seguridad) — con revisión legal previa obligatoria antes de habilitar esa fuente de ingesta (§19.4 escenario 6).
- **Flujo de habeas data post-mortem**: caso de uso separado para causahabientes/herederos que acreditan interés legítimo sobre el expediente de un servidor fallecido (§19.1 escenario 7) — distinto del flujo del propio servidor vivo.
- **Estado `EstadoExpediente::OcultoPorOrdenJudicial`** añadido a la máquina de estados de §16.1 — permite cumplir una medida cautelar de retiro provisional (art. 7 Decreto 2591/1991) desindexando de la proyección pública sin romper el event store ni equivaler a una `ActaRetractada` (que implica que el contenido era incorrecto; el ocultamiento judicial provisional no lo implica) — cierra §19.2 escenario 1.
- **Evento `FirmaImpugnada`** sobre actas firmadas en la ventana entre el robo de una llave de revisor y su revocación efectiva (§15.1) — marca el acta como "bajo revisión" sin alterar el hash original ni el event store, y dispara notificación a los afectados — cierra §19.3 escenario 2.
- **Evento `ActaRecontrafirmada`** — cuando un revisor migra de firma electrónica simple a firma digital certificada (§16.8), las actas anteriores se **contra-firman** (nuevo evento sobre el hash ya existente) en vez de reescribirse, ganando presunción legal reforzada desde la fecha de la contra-firma sin romper la cadena — cierra §19.3 escenario 8.
- **Actualización *ex officio* con SLA de 5 días hábiles**: cuando la plataforma tiene conocimiento fehaciente de una absolución o archivo de un proceso disciplinario externo, dispara automáticamente el equivalente de `ActaRetractada` sin esperar solicitud del servidor — requiere un mecanismo de seguimiento/consulta periódica al estado del proceso en Procuraduría/Personería (§19.4 escenario 1).
- **Protocolo de atención a requerimientos oficiales** (Procuraduría/Fiscalía con investigación propia ya abierta): validación de competencia del requirente + doble aprobación (§15.4) + registro auditable de la entrega — distinto del canal de orden judicial de retiro de §16.5, porque aquí la plataforma sí debe entregar identidad del aportante si la autoridad lo ordena motivadamente (§19.4 escenario 2).
- **Indicador de completitud de fuente por entidad territorial** en la capa de presentación — evita que un vacío de datos de una alcaldía pequeña (menor completitud de SIGEP/SECOP que una entidad nacional) se lea erróneamente como ausencia de irregularidades (§19.4 escenario 7).
- **Endpoint `GET /expedientes/{id}/prueba-merkle`** (§5) que expone la prueba de Merkle de un expediente para verificación pública independiente — el mecanismo ya estaba descrito conceptualmente en §8.5 pero no tenía contrato de API concreto (hallazgo de auditoría #5).


