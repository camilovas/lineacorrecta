# Seguridad — La Línea Correcta

> Extraído de `ARQUITECTURA.md` original. Las referencias `§N` conservan la numeración original del documento monolítico — ver [INDICE.md](../INDICE.md) para ubicar cada sección en su archivo actual.

## 6. Seguridad — OWASP

La Línea Correcta usa **cuatro** referencias OWASP distintas, cada una cubriendo una superficie distinta del sistema — tratarlas como una sola lista sería quedarse corto. Todas vigentes a 2026: API Security Top 10 (2023), Top 10 web general (2025, actualización mayor recién publicada), Top 10 CI/CD Security Risks, y ASVS 5.0.0 como estándar de verificación.

### 6.1 OWASP API Security Top 10 (2023) — prioritario, el sistema es API-first

| Riesgo | Mitigación en La Línea Correcta |
|---|---|
| API1 Broken Object Level Authorization | Todo acceso a `/expedientes/{id}` valida que el solicitante tenga permiso sobre ese recurso específico, no solo autenticación general |
| API2 Broken Authentication | Verificación de identidad real para aportantes/revisores; tokens de vida corta + refresh; nunca credenciales en query string |
| API3 Broken Object Property Level Authorization | Campos sensibles (ej. datos del aludido antes de publicación) filtrados según rol en el serializer, no en el cliente |
| API4 Unrestricted Resource Consumption | Rate limiting por IP/usuario, límites de tamaño de payload y de adjuntos, timeouts en operaciones de anclaje |
| API5 Broken Function Level Authorization | Endpoints de Areópago (firmar acta) exigen rol `revisor_acreditado` verificado en middleware, no solo en frontend |
| API6 Unrestricted Access to Sensitive Business Flows | Throttling + verificación anti-bot con **HumanGuard** (§15.2) en flujos críticos (registro, aporte de evidencia, creación masiva de expedientes) para prevenir abuso automatizado |
| API7 Server Side Request Forgery | Cualquier URL externa (enlaces de evidencia) se valida contra allowlist de dominios antes de fetch |
| API8 Security Misconfiguration | Headers de seguridad (`Strict-Transport-Security`, `X-Content-Type-Options`), CORS restrictivo, sin `debug mode` en prod |
| API9 Improper Inventory Management | Un solo esquema OpenAPI versionado; endpoints deprecados se retiran, no quedan huérfanos sin documentar |
| API10 Unsafe Consumption of APIs | Datos de SECOP/SIGEP/terceros se validan y sanitizan antes de persistir, tratados como input no confiable |

### 6.2 OWASP Top 10:2025 (web general) — cubre lo que la lista API-first no cubre

Actualización mayor publicada en enero de 2026 (la primera desde 2021, sobre 175.000 CVE y 589 CWE analizados). Relevante porque los clientes web/móvil de La Línea Correcta tienen superficie propia (sesiones de navegador, renderizado de contenido generado por usuarios en la capa social, dependencias de frontend) que la lista de API no cubre directamente. Dos categorías son **nuevas** y aplican directo a decisiones ya tomadas en este documento:

| Riesgo 2025 | Relevancia para La Línea Correcta |
|---|---|
| A01 Broken Access Control (#1 desde hace 4 ciclos; ahora absorbe SSRF, antes categoría propia) | Mismo control que API1/API7 de §6.1 — el punto SSRF de la validación de URLs externas de evidencia (API7) sigue vigente, ahora bajo este paraguas más amplio |
| A02 Cryptographic Failures | Directamente relevante a §15.1/§16.7/§16.8: uso correcto de Ed25519, serialización canónica antes de hashear, nunca criptografía casera |
| **A03 Software Supply Chain Failures (nueva)** | Formaliza lo que §15.3 ya proponía (SBOM, firma de artefactos) como categoría de primer nivel, no una nota aparte — refuerza que no es opcional |
| A04 Injection | SQL (mitigado por `sqlx` parametrizado, §6.1 prácticas transversales), pero también inyección en el pipeline de ingesta de documentos SECOP/SIGEP (§16.2) — el parser de esos documentos es superficie de ataque, no solo la API HTTP |
| A05 Insecure Design | Justifica retroactivamente el enfoque de este documento: threat modeling y máquina de estados (§16.1) *antes* de escribir código de negocio, no una auditoría posterior |
| A06 Security Misconfiguration | Igual que API8 |
| A07 Vulnerable and Outdated Components | Igual que `cargo audit`/`cargo deny` de §6.1, extendido a dependencias de frontend (`npm audit` si hay clientes JS) |
| A08 Identification and Authentication Failures | MFA de revisores (§15.1), gestión de sesión |
| A09 Security Logging and Monitoring Failures | Cubierto por observabilidad de §12.4, pero con foco específico: un fallo de anclaje o un intento de autorización repetido debe generar alerta, no solo log |
| **A10 Mishandling of Exceptional Conditions (nueva)** | Directamente relevante a Rust: prohibir `.unwrap()`/`.expect()` en rutas de producción del núcleo de evidencia — un panic no controlado en el proceso que firma/ancla actas es un incidente de disponibilidad *y* potencialmente de integridad si interrumpe una escritura a medias. Se traduce en regla de linting obligatoria (`clippy::unwrap_used` denegado en CI para `domain/`, `application/`, `adapters-crypto/`) |

### 6.3 OWASP Top 10 CI/CD Security Risks — protege el pipeline de §12.3, no solo el producto

Un atacante que compromete el pipeline no necesita encontrar una vulnerabilidad en el código: inyecta la suya directamente en el build que se despliega. Los dos vectores más explotados en la práctica:

- **Poisoned Pipeline Execution (CICD-SEC-4)** — el más común en incidentes reales: un PR de un colaborador externo o un hook mal restringido logra ejecutar código arbitrario dentro del pipeline de CI. Mitigación: PRs de fuentes no confiables corren en un entorno sin acceso a secrets; cualquier paso del pipeline que sí tenga acceso a credenciales de despliegue requiere aprobación explícita, nunca se dispara automáticamente sobre código no revisado.
- **Insufficient Flow Control (CICD-SEC-1)** — ya parcialmente cubierto por "ningún deploy a producción sin aprobación humana" (§12.3, ADR-7 en §14), pero se extiende a: ningún cambio a `main` sin revisión de otra persona (branch protection), y las credenciales de despliegue nunca se materializan en logs ni en variables expuestas a pasos no confiables del pipeline.
- **Dependency Chain Abuse (CICD-SEC-3)** — dependency confusion/typosquatting sobre crates de Rust o paquetes npm del frontend; mitigado por `Cargo.lock`/`package-lock.json` commiteados y verificados en CI (fallo si el lockfile no coincide con lo declarado), y por el SBOM de §15.3/§6.2 (A03).
- **Inadequate Identity and Access Management (CICD-SEC-2)** — las credenciales que el pipeline usa para desplegar (llaves de infraestructura, tokens de Docker Hub/registry) siguen el mismo principio de mínimo privilegio de §15.4, con rotación periódica, nunca compartidas entre staging y producción.

### 6.4 ASVS 5.0.0 — el estándar de verificación, no solo un checklist de riesgos

El Top 10 (§6.1/§6.2) dice *qué puede salir mal*; el **ASVS 5.0.0** (publicado mayo 2025, ~350 requisitos verificables en 17 categorías) dice *qué implementar y cómo verificarlo* — es la referencia que se usa para escribir criterios de aceptación de seguridad, no solo para post-mortems.

- **Nivel objetivo recomendado: ASVS Nivel 2** (aplicaciones que manejan datos sensibles/transacciones significativas) como mínimo para todo el sistema, y **Nivel 3** específicamente para los módulos de firma/verificación (`adapters-crypto/`) y gestión de llaves (§15.1) — el nivel más alto, reservado normalmente para aplicaciones de máximo riesgo, es proporcional aquí porque un fallo criptográfico tiene consecuencia legal directa (§10.5), no solo técnica.
- Los requisitos ASVS relevantes para el núcleo de evidencia se referencian por ID (`v5.0.0-X.Y.Z`) en los tickets/ADRs correspondientes desde el diseño, no se auditan retroactivamente — es la diferencia entre "seguro por diseño" y "seguro por auditoría".

### 6.5 Prácticas transversales

- **Autenticación/autorización**: JWT firmado o sesiones opacas + verificación de rol en cada caso de uso (defensa en profundidad, no solo en el gateway).
- **Validación de entrada estricta** en el borde (adaptador HTTP), tipado fuerte de Rust como primera línea de defensa.
- **Prepared statements / ORM parametrizado** (`sqlx`) — cero SQL concatenado.
- **Secrets** fuera del repo (vault/env), rotación periódica de llaves de firma.
- **Logging sin datos sensibles**, con trazabilidad de quién accedió a qué expediente (auditoría, coherente con el propio principio de la plataforma).
- **Dependencias auditadas**: `cargo audit` / `cargo deny` en CI contra CVEs conocidos.
- **HTTPS obligatorio**, HSTS, TLS 1.2+.
- **Manejo de excepciones sin pánico** (A10 §6.2): `Result<T, E>` explícito en todo el núcleo, `.unwrap()` prohibido por lint en CI fuera de tests.


## 15. Modelo de amenazas y seguridad avanzada

El §6 (OWASP API Top 10) cubre la superficie estándar de una API. Esta sección cubre lo que es **específico del modelo de amenaza de La Línea Correcta**: una plataforma que publica señalamientos verificados sobre personas con poder real —incluida fuerza pública y altos cargos— y que por diseño no puede permitirse que un solo servidor comprometido falsifique un veredicto.

### 15.1 Gestión de llaves de firma (Ed25519) — el activo más crítico del sistema

La llave privada de un revisor del Areópago es, en la práctica, más sensible que cualquier dato en la base de datos: quien la controle puede forjar un acta oficial.

- **La plataforma nunca custodia llaves privadas de revisores.** El firmado ocurre en el dispositivo del revisor (hardware wallet, HSM personal, o al menos un enclave seguro del dispositivo) — el backend solo recibe y verifica la firma, nunca la genera ni la almacena. Si un servidor de La Línea Correcta se compromete, un atacante no debe poder firmar nada.
- **Ceremonia de generación de llaves** documentada y reproducible (idealmente presencial o con testigos para revisores fundadores), con registro público de la llave pública asociada a cada identidad de revisor desde el día uno.
- **Rotación y revocación**: proceso definido para invalidar la llave de un revisor comprometido, coaccionado o que deja el Areópago — incluye lista de revocación pública, consistente con el registro abierto de revisores ya descrito en el sitio.
- **MFA obligatorio** en la cuenta de plataforma de cada revisor, independiente y más estricto que el login de un ciudadano consultante — comprometer esa cuenta no debe bastar para firmar (la firma criptográfica en el dispositivo del revisor es la segunda barrera real).

### 15.2 Adversario con recursos, no solo atacante oportunista

El modelo de amenaza no es "hacker aleatorio buscando una vulnerabilidad genérica" — incluye actores con incentivo directo para silenciar o manipular la plataforma: el propio funcionario señalado, redes organizadas, o en el peor caso un actor con recursos estatales.

- **Protección de borde en dos capas independientes, no una sola herramienta:**
  - **Capa de red (L3/L4) — DDoS volumétrico y CDN**: sigue siendo necesario un proveedor de borde tipo Cloudflare (o equivalente) delante de toda la infraestructura, para absorber inundaciones de tráfico y servir contenido estático/cacheado cerca del usuario. HumanGuard **no cubre esta capa** — es un servicio de aplicación (L7), no un proveedor de red perimetral.
  - **Capa de aplicación (L7) — detección de bots y abuso automatizado**: aquí se usa **HumanGuard** (producto propio, `api.humanguard.app`) en lugar de un CAPTCHA tradicional, integrado en los puntos de fricción críticos: registro de cuenta, aporte de evidencia, comentarios y "me gusta" (mitiga directamente el riesgo de brigading/manipulación de percepción de §7 y el abuso de flujos de negocio de OWASP API6 en §6). Funciona con challenges gamificados de 3-5s + análisis de comportamiento por IA (ensemble XGBoost+LSTM), sin fricción tipo CAPTCHA clásico, y ya está en producción y validado (Accuracy 99.06%, Recall 100%, FPR 2.4% sobre datos reales).
  - El flujo de verificación: SDK web/Android de HumanGuard emite un `result_token` firmado tras el challenge → el backend de La Línea Correcta lo valida vía `POST /v1/verify-token` contra la API de HumanGuard antes de aceptar la acción (registro, comentario, like, aporte de evidencia) — se integra como un adaptador más en `adapters-http/` (§2), detrás de un trait `BotVerificationService`, para poder sustituirlo si hiciera falta sin tocar los casos de uso.
  - Las dos capas son complementarias, no sustitutas: **la capa de red protege la disponibilidad de la infraestructura; HumanGuard protege la integridad del comportamiento humano de la capa social.** Eliminar Cloudflare (o equivalente) sin reemplazo dejaría la plataforma expuesta a DDoS volumétrico, que ningún detector de bots a nivel de aplicación puede mitigar por sí solo.
- **Protección de metadatos de quien aporta evidencia o comenta**: IPs y datos de sesión no deben ser accesibles a todo el equipo interno ni quedar en logs de aplicación — solo a un proceso de seguridad estrictamente necesario, con acceso auditado (principio de mínimo privilegio aplicado también hacia adentro del propio equipo).
- **Política de respuesta a solicitudes legales/gubernamentales de identificación de usuarios**, definida y publicada *antes* de que ocurra una solicitud real — qué se entrega, bajo qué orden judicial, y qué se notifica al usuario afectado si la ley lo permite.
- **Protección reforzada para reporteros/aportantes en riesgo documentado**, coherente con lo ya declarado en el sitio ("la atribución individual puede divulgarse en diferido") — esto debe tener un mecanismo técnico real (ej. desacoplar identidad verificada de identificador público hasta que el riesgo se descarte), no solo una promesa de producto.

### 15.3 Cadena de suministro de software

- **SBOM** (inventario de dependencias, `cargo-sbom` o similar) generado en cada build, no solo `cargo audit` puntual.
- **Firma de artefactos de build** (ej. `cosign`/Sigstore) para que un binario desplegado sea verificablemente el que salió del pipeline de CI, no uno alterado en tránsito.
- **Revisión de salud de dependencias criptográficas**: los crates que implementan Ed25519, Merkle trees y hashing deben ser los de mayor adopción y auditoría en el ecosistema Rust (ej. `ed25519-dalek`), evitando implementaciones propias o de mantenimiento único para primitivas criptográficas — "no ruedes tu propia criptografía".

### 15.4 Insider threat y separación de funciones

El event store append-only (§11) protege contra alteración retroactiva, pero no contra un administrador con acceso legítimo que abusa de ese acceso.

- **Doble control (four-eyes) para operaciones administrativas sensibles**: ninguna acción destructiva o de alto impacto (ej. suspender un expediente, modificar permisos de revisor) debe poder ejecutarla una sola persona sin aprobación de una segunda.
- **Auditoría de acceso administrativo** con el mismo rigor que la auditoría de firmas del Areópago — quién vio o tocó qué expediente, con timestamp, igual de trazable e inmutable.
- **Principio de mínimo privilegio** estricto: acceso de producción (DB, logs con IPs, llaves de infraestructura) limitado al mínimo equipo posible, revisado periódicamente.

### 15.5 Pentesting y respuesta a incidentes

- **Pruebas de penetración periódicas** (al menos anuales, y antes de cualquier cambio mayor al esquema de firmas o anclaje), incluyendo pruebas específicas de los escenarios de §15.1 y §15.2 — no solo un escaneo automatizado de la API.
- **Plan de respuesta a incidentes** documentado: quién se activa, cómo se contiene, cómo se comunica a los usuarios afectados, y en qué plazo — coherente con las obligaciones de notificación de la Ley 1581 de 2012 (§10) ante una brecha de datos personales.
- **Bug bounty o canal de divulgación responsable** una vez el proyecto tenga tracción, para que investigadores de seguridad reporten hallazgos antes de que los exploten terceros.

### 15.6 Resumen de prioridad

| Prioridad | Medida | Por qué es urgente |
|---|---|---|
| Crítica | Firmado de revisores nunca en el servidor (15.1) | Es la garantía central del producto; sin esto, todo lo demás es cosmético |
| Crítica | MFA + rotación/revocación de llaves de revisor (15.1) | Compromiso de una sola cuenta no debe poder forjar un acta oficial |
| Alta | Protección de metadatos de aportantes/reporteros (15.2) | Riesgo de daño físico real a personas, no solo reputacional |
| Alta | CDN/anti-DDoS de red (L3/L4) + HumanGuard para anti-bot de aplicación (L7) (15.2) | La plataforma es un blanco predecible en momentos de mayor impacto noticioso; son dos capas independientes, ninguna sustituye a la otra |
| Media | SBOM y firma de artefactos (15.3) | Reduce riesgo de compromiso silencioso de la cadena de build |
| Media | Doble control administrativo (15.4) | Cierra el último vector de alteración que el event sourcing no cubre por sí solo |
| Continua | Pentesting + plan de incidentes (15.5) | No es una medida única sino un proceso que debe mantenerse vivo |


