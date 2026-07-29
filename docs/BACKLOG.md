# Backlog — La Línea Correcta

Todo lo que el panel de revisión (ver `ARQUITECTURA.md` §20) decidió posponer respecto al diseño original, más los hallazgos de seguridad que sí deben resolverse pronto aunque sean baratos. Cada ítem indica **por qué** se difiere y **cuándo** retomarlo, no solo qué es — para que posponerlo sea una decisión trazable, no un olvido.

Prioridad: 🔴 crítico (resolver antes de tracción real) · 🟡 importante (Sprint 5-8) · 🟢 madurez (cuando el producto ya tenga usuarios y presupuesto)

## 🔴 Crítico — antes de escalar tráfico real

1. **Playbook de actuación coordinada inauténtica (brigading)** — la única de las 32 brechas simuladas que sigue genuinamente abierta (`ARQUITECTURA.md` §19.2 escenario 3, §20.2). Convergencia de seguridad y legal como el vector de ataque más barato y probable. No tiene hoy más especificación que "reforzar HumanGuard". Diseñar: umbral distinto al de spam ordinario, detección de patrón temporal/coordinación, escalamiento a revisión humana antes de cualquier acción automática sobre el contenido.
2. **Verificación cruzada de evidencia contra fuente oficial** (hallazgo de seguridad, §20.1) — un documento SECOP/SIGEP falsificado con apariencia oficial pasa hoy el Areópago por confianza visual. Añadir: comparación de hash/checksum contra el dataset público en el momento de ingesta cuando la fuente lo permita; si no es posible verificar, marcarlo explícitamente como "no verificable contra fuente" en el expediente, no como verificado.
3. **Protocolo de firma con confirmación visible** (hallazgo de seguridad) — MFA de cuenta no defiende contra un dispositivo de revisor comprometido que sustituye el contenido antes de firmar. Evaluar mecanismo tipo hardware wallet (mostrar hash/resumen en pantalla del dispositivo de firma) para revisores, no solo llave nunca-en-servidor.
4. **Política fail-open/fail-closed de HumanGuard** (convergencia seguridad + legal, §20.2) — qué pasa con los flujos protegidos (registro, comentarios, aporte de evidencia) si HumanGuard falla, tiene downtime, o es explotado con un ataque adversarial. Hoy no está definido.
5. **Persona jurídica formal + seguro de responsabilidad** (hallazgo legal) — antes de publicar el primer expediente real, resolver si el fundador responde con patrimonio personal o si existe una entidad constituida con separación patrimonial. No es código, es lo más urgente de todo el backlog en términos de riesgo real.

## 🟡 Importante — Sprint 5 en adelante, cuando haya tracción

6. **Segunda blockchain de anclaje** (recalibrado en §8/§20.1) — la redundancia de v0 es blockchain única + TSA/RFC 3161; una segunda red pública se agrega cuando el volumen/riesgo real lo justifique. El trait `AnclajeService` ya soporta esto sin cambio de contrato.
7. **Split a workspace multi-crate** (§17) — `domain`, `domain-social`, `application`, `application-social`, `adapters-http/db/crypto/anclaje/humanguard`, `shared` como crates independientes. Migración mecánica desde los módulos de v0 (mismos nombres, mismos límites) — se activa cuando el equipo crece o cuando compilar un solo crate empieza a doler.
8. **`lc-worker-anonimizacion` como binario automatizado** (§16.9) — en v0 es un script manual que el equipo ejecuta; se automatiza cuando el volumen de expedientes lo justifique.
9. **CQRS ligero + réplica de lectura** (§12.1, §13.3) — Postgres monolítico sin réplica sirve de sobra para miles de expedientes; agregar cuando el tráfico del feed social lo exija.
10. **Fold completo del event store + proyecciones asíncronas** — v0 usa tabla append-only + estado actual actualizado síncronamente (barato, alto valor legal); el fold puro con reconstrucción de agregado se evalúa si realmente hace falta replay histórico completo.
11. **Motor de queja disciplinaria (art. 86 CGD)** (§16.6) — bajo costo, alto valor, pero no bloquea el lanzamiento del núcleo.
12. **CCT de transferencia internacional de datos, runbook de notificación de brecha (15 días hábiles SIC)** — resolver antes de manejar volumen real de datos personales, en paralelo al crecimiento, no en Sprint 1.
13. **Convenio formal con perito informático** (§16.10) — para tener listo antes del primer litigio real, no antes de tener el primer expediente.

## 🟢 Madurez — cuando el producto ya tenga usuarios reales

14. **ASVS Nivel 3** en `adapters-crypto`/gestión de llaves (v0 arranca en Nivel 1-2 básico).
15. **Entidad de certificación acreditada ONAC** para firmas del Areópago (v0 usa firma electrónica simple, válida per art. 7 Ley 527, sin la presunción legal reforzada del art. 28-29).
16. **Mutation testing, chaos engineering, fuzzing** (§9.3, §9.4, §9.2) — inversión de madurez de ingeniería; llegan cuando el código del núcleo ya esté estable (antes de eso, cambia demasiado para que la inversión rinda).
17. **Crypto-shredding activo** (§10.1) — el esquema de datos debe reservar el campo de llave desde Sprint 1 (ver `ARQUITECTURA.md` §20.4), pero la funcionalidad de destrucción de llave bajo orden judicial se activa solo si/cuando ocurra el escenario.
18. **Indicador de completitud de fuente por entidad territorial** (§19.4 escenario 7).
19. **Constitución formal de red de veedurías (Ley 850/2003)** — decisión de negocio separada, evaluar solo si aporta valor real más allá de lo que la plataforma tecnológica ya hace (§10.4, §19.4 escenario 8).
20. **Bug bounty / canal de divulgación responsable** (§15.5) — cuando el proyecto tenga tracción suficiente para que valga la pena.

## Notas de trazabilidad

- Cada ítem aquí tiene su justificación técnica/legal completa en `ARQUITECTURA.md` — este archivo es el índice ejecutable, no la fuente de verdad del razonamiento.
- Ningún ítem 🔴 requiere arquitectura nueva no contemplada — todos encajan en los puertos/traits ya definidos en `ARQUITECTURA.md` §2, §8, §15.
- Revisar este backlog al cierre de cada sprint: mover ítems entre prioridades si la realidad del producto (usuarios, incidentes, presión legal) cambia el orden.
