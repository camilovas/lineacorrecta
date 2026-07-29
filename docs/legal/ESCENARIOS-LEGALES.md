# Escenarios legales simulados — La Línea Correcta

> Extraído de `ARQUITECTURA.md` original. Las referencias `§N` conservan la numeración original del documento monolítico — ver [INDICE.md](../INDICE.md) para ubicar cada sección en su archivo actual.

## 19. Simulación de escenarios legales — respuesta y brechas

Investigación producida por cuatro líneas de simulación independientes (32 escenarios en total) más una auditoría cruzada de todo lo ya escrito en §10/§16. El objetivo no es exhaustividad teórica sino verificar que el diseño responde a situaciones concretas que la plataforma probablemente enfrentará, y detectar dónde la arquitectura actual se queda corta. **Estado** indica si el diseño ya vigente (incluidas las correcciones de §10.1/§16.11 de esta misma revisión) cubre el escenario, o si sigue siendo una brecha abierta que requiere decisión adicional.

### 19.1 Habeas data y privacidad

| # | Escenario | Marco legal | Respuesta de la plataforma | Estado |
|---|---|---|---|---|
| 1 | Un servidor público exige conocer TODA la información que la plataforma tiene sobre él, no solo lo publicado | Art. 14 Ley 1581/2012: consulta en 10 días hábiles (+5 prórroga), reclamo en 15 (+8) | Exportación consolidada de todos los bounded contexts (núcleo + social + logs) vinculados a su identidad, dentro del SLA legal | **Cubierto** — `SolicitudHabeasData::Acceso` (§16.11) con SLA propio |
| 2 | Un aportante de evidencia pide eliminar su identidad tras recibir amenazas | Ley 1581 (supresión) vs. interés legítimo de verificar la fuente (art. 224 CP) | Desacoplar identidad verificada de identificador público sin romper el vínculo interno auditable | **Cubierto** — evento `IdentidadAportanteProtegida` (§16.11) |
| 3 | Un menor de edad aparece mencionado en un documento de evidencia (ej. nepotismo) | Art. 7 Ley 1581 — protección reforzada de datos de menores | Redactar/pixelar el dato del menor conservando el hecho relevante (el nombramiento del funcionario) | **Parcialmente cubierto** — requiere que el clasificador de §16.2 distinga explícitamente "menor de edad" de "dato sensible general" (queda como refinamiento pendiente del enum `Sensibilidad`) |
| 4 | Infraestructura cloud fuera de Colombia (AWS/GCP en regiones no colombianas) | Art. 26 Ley 1581 — transferencia internacional requiere garantías (CCT Red Iberoamericana, Circulares 002/003 de 2025 SIC) | Firmar CCT con el proveedor antes de almacenar cualquier dato personal fuera del país | **Cubierto** en checklist operativo (§16.10), no requiere cambio técnico |
| 5 | Brecha de seguridad con exposición de datos de aportantes | Circular Única SIC Título V — notificación a la SIC en 15 días hábiles desde la detección | Runbook de incidentes con ese reloj vinculado operativamente al plan de §15.5 | **Cubierto** en checklist operativo (§16.10) |
| 6 | La SIC abre investigación de oficio pidiendo prueba de verificación previa (precedente Res. 43121/2024) | Mismo régimen que sancionó a la plataforma de referencia | Exhibir el event store completo del expediente (quórum de firmas, `EstadoExpediente::EnVerificacion` cumplido) | **Ya cubierto** — la máquina de estados de §16.1 lo hace estructuralmente imposible de evadir |
| 7 | Servidor público fallece; sus herederos solicitan algo sobre su expediente | Habeas data post-mortem reconocido por la Corte (T-798/07) y la SIC | Flujo separado de verificación de calidad de heredero antes de conceder acceso/rectificación | **Cubierto** — flujo de habeas data post-mortem (§16.11) |
| 8 | Un tercero (testigo, familiar) aparece mencionado incidentalmente en un documento | Minimización de datos — sin base de legitimación propia si no es objeto del señalamiento | Redactar/generalizar salvo que sea imprescindible para el hecho | **Cubierto** — campo `rol_mencion` en el clasificador (§16.11) |

### 19.2 Injuria, calumnia y moderación de contenido

| # | Escenario | Marco legal | Respuesta de la plataforma | Estado |
|---|---|---|---|---|
| 1 | Funcionario interpone tutela directa sin solicitar rectificación previa; el juez ordena retiro provisional mientras decide | Art. 42.7 Decreto 2591/1991 (procedibilidad); art. 7 Decreto 2591 (medida cautelar) | Alegar falta de subsidiariedad citando el canal de réplica ya ofrecido; cumplir la cautelar ocultando sin destruir el event store | **Cubierto** — estado `OcultoPorOrdenJudicial` (§16.11) |
| 2 | Un acta "Validado" resulta luego errónea por nueva evidencia | Art. 225 CP — retractación exime de responsabilidad si es antes de sentencia de primera instancia | Evento `ActaRetractada` con misma prominencia visual, sin editar el evento original | **Ya cubierto** — §16.3, §9.7 lo prueba automáticamente |
| 3 | Actuación coordinada inauténtica (bots/brigading) acusando algo no verificado en comentarios | T-241/23 exonera si hay canal de reporte y no hay editorialización | Tratar igual que Meta (moderación neutral) + HumanGuard reforzado, dado el mayor escrutinio por el rol editorial del acta | **Brecha abierta** — falta un playbook específico de "actuación coordinada" con umbral distinto al de spam ordinario |
| 4 | Un medio de comunicación reproduce un expediente y es demandado | Cada difusor responde por su propia conducta; La Línea Correcta enfrenta la carga de la exceptio veritatis sobre el acta original | Mantener metadatos de "generación confiable" y URL permanente citable | **Cubierto** — §10.5, §10.4 (atribución) |
| 5-6 | Uso de un expediente en campaña electoral / régimen de Ley de Garantías | Ley 996/2005 y competencia del CNE recaen sobre el candidato/campaña, no sobre la fuente | Política de neutralidad editorial con criterios objetivos de timing + disclaimer de "no es propaganda de La Línea Correcta" | **Cubierto** en checklist operativo (§16.10) |
| 7 | Un revisor es demandado personalmente por su firma en un acta errónea | Sin norma específica — asunto contractual | Cláusula de indemnidad (hold harmless) + seguro de responsabilidad civil profesional | **Cubierto** en checklist operativo (§16.10) — decisión de negocio, no técnica |
| 8 | Querella penal contra la persona jurídica (La Línea Correcta) | Calumnia solo admite personas naturales como sujeto activo; injuria admite personas jurídicas como sujeto pasivo pero no activo | Individualizar autoría por revisor firmante — la querella debe dirigirse a la persona natural, no a la plataforma | **Ya cubierto** — firmas Ed25519 nominales por revisor (§10.5) lo blindan por diseño |

### 19.3 Valor probatorio, criptografía y proceso judicial

| # | Escenario | Marco legal | Respuesta de la plataforma | Estado |
|---|---|---|---|---|
| 1 | La contraparte "tacha de falso" un acta aportada como prueba | Art. 269-270 CGP — cotejo pericial | JSON canónico (§16.7) + algoritmo de Merkle documentado públicamente + perito de convenio | **Cubierto**, falta solo el convenio formal firmado (checklist §16.10) |
| 2 | Llave privada de un revisor robada; se firma un acta fraudulenta antes de la revocación | Sin presunción de firma digital certificada, autoría queda a valoración libre del juez | Lista de revocación pública con timestamp + evento `FirmaImpugnada` sobre las actas de la ventana comprometida | **Cubierto** — nuevo evento (§16.11) |
| 3 | El proveedor de anclaje (ej. Polygon) sufre un fork o discontinúa el servicio | La verificabilidad práctica se rompe aunque el hecho de que el hash existía siga siendo cierto | Anclaje redundante en al menos dos redes independientes desde el lanzamiento | **Cubierto** — corrección de §8 en esta misma revisión |
| 4 | Orden judicial exige el código/algoritmo de hashing para peritaje | El juez puede exigirlo sin resistencia legal posible | Código abierto + versión firmada (Sigstore, §15.3) reduce fricción — ayuda en vez de complicar | **Ya cubierto** por diseño de apertura del código |
| 5 | Autoridad extranjera solicita datos de un funcionario colombiano investigado en el exterior | Solo vía carta rogatoria tramitada por autoridad judicial colombiana + Cancillería | Política publicada de "no se entrega nada sin exhorto tramitado", verificación de autenticidad | **Cubierto** en checklist operativo (§16.10) |
| 6 | Revisor expulsado tras haber firmado 200 actas | La expulsión no invalida retroactivamente actas firmadas correctamente | Publicar "revisor revocado desde fecha X"; actas anteriores mantienen su `nivel_legal()` intacto | **Ya cubierto** — §15.1 + §16.8 ya separan estado del revisor de validez del acta |
| 7 | Brecha temporal entre el evento y su anclaje en blockchain | El anclaje prueba no-alteración posterior, no el momento exacto de ocurrencia | Documentar explícitamente esta distinción en el dictamen pericial estándar; el hash-chain interno + timestamp de sistema prueban el momento, el anclaje corrobora integridad | **Cubierto conceptualmente** (§11.1), falta solo explicitarlo en el dictamen pericial (checklist §16.10) |
| 8 | Migración de firma electrónica simple a firma digital certificada (ONAC) | Las actas antiguas no ganan presunción retroactiva automática | Evento `ActaRecontrafirmada` sobre el hash ya existente, sin reescribir el payload original | **Cubierto** — nuevo evento (§16.11) |

### 19.4 Transparencia, veedurías y proceso disciplinario

| # | Escenario | Marco legal | Respuesta de la plataforma | Estado |
|---|---|---|---|---|
| 1 | Un funcionario señalado es absuelto en el proceso disciplinario externo | Art. 86 Ley 1952/2019 + retractación en equidad "sin dilación" | Seguimiento periódico del estado del proceso + actualización automática con SLA de 5 días hábiles desde conocimiento fehaciente | **Cubierto** — mecanismo de seguimiento + SLA (§16.11) |
| 2 | Procuraduría solicita el expediente completo, incluida la identidad del aportante | No hay privilegio de protección de fuente oponible a autoridad disciplinaria con investigación ya abierta | Entregar identidad si la orden es motivada y formal, con validación de competencia + doble aprobación | **Cubierto** — protocolo de requerimientos oficiales (§16.11), distinto del canal de orden judicial de retiro |
| 3 | SIGEP/SECOP cambian su licencia o retiran un dataset ya usado | Licencia de atribución — responsabilidad del reutilizador, no de la fuente | Los expedientes ya publicados no caen retroactivamente; se sustentan con `fuente`+`fecha_extraccion` inmutables | **Ya cubierto** — §16.2, §10.4 versionan esto desde el diseño |
| 4 | Alegación de propaganda política encubierta durante campaña | Competencia del CNE; tutela por afectación a derechos políticos | Criterios objetivos de timing + disclaimer de neutralidad | **Cubierto** en checklist operativo (§16.10) |
| 5 | Queja disciplinaria generada por la plataforma con evidencia luego insuficiente/manipulada | Responsabilidad recae en quien aporta evidencia falsa, no en el canal estructurador | Trazabilidad de autoría por pieza de evidencia (`actor_id`) + constancia explícita de que la queja no es un juicio propio de la plataforma | **Ya cubierto** — §16.6 ya separa "queja" de "veredicto propio" |
| 6 | Datos del sector Fuerza Pública (régimen especial) | Art. 19 Ley 1712 — reserva por seguridad/defensa nacional | Subcategoría de clasificación reforzada, distinta del resto del catálogo | **Cubierto** — `Sensibilidad::ReservadoSeguridadNacional` (§16.11) |
| 7 | Desigualdad de completitud de datos entre una alcaldía pequeña y una entidad nacional | Riesgo reputacional/probatorio, no de trato desigual jurídico strictu sensu | Indicador de completitud de fuente visible por entidad, para no confundir vacío de datos con ausencia de irregularidades | **Cubierto** — indicador de completitud (§16.11) |
| 8 | Se constituye formalmente una red de veedurías ciudadanas asociada (Ley 850/2003) | Obligaciones de acta constitutiva, registro, rendición de cuentas — un sujeto colectivo distinto | Decisión de negocio separada, no responsabilidad de la plataforma tecnológica | **No aplica a la arquitectura** — correctamente fuera del alcance técnico (§10.4) |

### 19.5 Lectura transversal de los 32 escenarios

- **24 de 32 escenarios ya quedan cubiertos** por el diseño existente más las correcciones de esta revisión (§8, §10.1, §10.6, §16.11); **7 quedan resueltos como checklist operativo/contractual** (no requieren código, pero sí una decisión o gestión antes del lanzamiento); **1 sigue siendo una brecha genuinamente abierta**: el playbook de actuación coordinada inauténtica (§19.2 escenario 3) no tiene todavía una especificación concreta más allá de "reforzar HumanGuard" — es el punto pendiente de mayor prioridad para la próxima iteración de este documento.
- El patrón que se repite en los escenarios "ya cubiertos": casi todos se resuelven **añadiendo un evento nuevo al event store existente**, nunca modificando o rompiendo la inmutabilidad de lo ya sellado — es la señal de que la decisión de arquitectura de event sourcing (§11, ADR-2 en §14) sigue siendo la correcta incluso bajo presión de casos límite no anticipados originalmente.


