# Testing — La Línea Correcta

> Extraído de `ARQUITECTURA.md` original. Las referencias `§N` conservan la numeración original del documento monolítico — ver [INDICE.md](../INDICE.md) para ubicar cada sección en su archivo actual.

## 9. Testing

Estrategia de pruebas alineada con las capas hexagonales: cada capa se prueba con la herramienta adecuada a su nivel de acoplamiento. Dada la naturaleza del producto (evidencia legal, criptografía, datos personales sensibles), la pirámide de pruebas estándar no basta — se complementa con mutación, fuzzing, caos y pruebas de cumplimiento como ciudadanos de primera clase, no como extras opcionales.

### 9.1 Pirámide base

| Capa | Tipo de prueba | Herramienta / enfoque | Qué cubre |
|---|---|---|---|
| `domain/` | **Unitarias puras** | `#[test]` estándar de Rust, sin mocks | Reglas de negocio: quórum mínimo de firmas, transiciones válidas de veredicto (§16.1), invariantes de `Acta` |
| `domain/` (crypto) | **Property-based testing** | `proptest` | Firma/verificación Ed25519, construcción de Merkle tree — propiedades como "verificar(firmar(x)) siempre es true" para cualquier input |
| `application/` | **Unitarias con dobles** | Fakes/mocks de los traits (`FakeExpedienteRepository` en memoria) | Casos de uso orquestando el dominio correctamente, sin tocar red ni DB |
| `adapters-db/` | **Integración** | `testcontainers` (Postgres real en Docker) | Queries reales, migraciones, constraints de integridad referencial |
| `adapters-http/` | **Contract testing** | Generación desde OpenAPI (`utoipa` + `schemathesis` o similar) | La API cumple su propio contrato documentado; detecta breaking changes antes de que rompan al cliente móvil/web |
| API completa | **E2E** | Cliente HTTP contra el binario levantado en CI (`reqwest` + entorno docker-compose) | Flujos completos: crear expediente → adjuntar evidencia → firmar → sellar acta → verificar anclaje → generar queja disciplinaria (§16.6) |

### 9.2 Seguridad y autorización (más allá del checklist OWASP de §6)

- **Tests de autorización negativos explícitos** por cada regla de §6 (ej. "usuario sin rol revisor no puede firmar", "usuario A no puede leer datos de propiedad-nivel de usuario B") — API1/API3/API5 del OWASP API Top 10; tan obligatorios como los tests funcionales, no un anexo.
- **Fuzzing** (`cargo-fuzz` / `afl.rs`) sobre cualquier parser de entrada no confiable: payloads JSON de eventos, documentos SECOP/SIGEP ingeridos (§16.2), y muy especialmente la deserialización de firmas y estructuras criptográficas — es la superficie más común de vulnerabilidades de memoria/lógica incluso en Rust (parsers con lógica manual siguen siendo falibles aunque el lenguaje evite corrupción de memoria).
- **Tests de la máquina de estados del `Expediente`** (§16.1): además de los unitarios, un test exhaustivo que enumera todas las transiciones posibles del enum y confirma que las no listadas explícitamente **no compilan** — convierte la garantía legal en una prueba de tipo, no solo de comportamiento en runtime.

### 9.3 Mutation testing — probar que las pruebas realmente prueban algo

- **`cargo-mutants`** sobre `domain/` y `adapters-crypto/`: introduce mutaciones automáticas al código (invertir un `if`, cambiar `<` por `<=`, eliminar una línea) y verifica que al menos un test falle. Una alta cobertura de líneas (§9.5) no garantiza que los tests detecten errores reales — mutation testing sí lo mide directamente.
- Aplicado con prioridad a la lógica de verificación de firmas y construcción del Merkle tree: es exactamente el código donde un test "que pasa pero no prueba nada" tendría el mayor costo posible (una firma inválida aceptada como válida).

### 9.4 Chaos engineering y resiliencia

- **Inyección de fallos controlada** en staging: caída súbita del worker de anclaje (§8, §13.1) a mitad de un lote, pérdida de conexión a Postgres durante una escritura de evento, timeout del proveedor de blockchain — confirma que el patrón outbox/at-least-once (§13.1) realmente recupera sin duplicar ni perder eventos.
- **Simulacros de restauración de backup** (no solo verificar que el backup se generó, sino restaurarlo completo en un entorno aislado y validar integridad) con cadencia trimestral — un backup nunca probado no es un backup, es una suposición.
- **Prueba de "servidor comprometido"**: ejercicio periódico que confirma en un entorno de staging que, aun con acceso total a la base de datos, no es posible forjar una firma válida — validación práctica y recurrente de la garantía de §15.1, no solo diseño en papel.

### 9.5 Cobertura, calidad y accesibilidad de clientes

- El dominio (`domain/`) exige **cobertura cercana al 100%** por ser lógica crítica y sin dependencias externas que dificulten testear.
- Los casos de uso (`application/`) se prueban con fakes, no con la base de datos real — mantiene el test suite rápido (segundos, no minutos) y ejecutable en cada commit.
- **Pruebas de accesibilidad (WCAG 2.1 AA)** en web y móvil — `axe-core` integrado en los tests E2E de frontend, y verificación manual con lector de pantalla en los flujos críticos (consultar expediente, aportar evidencia, firmar réplica). Es una plataforma de participación ciudadana; excluir usuarios con discapacidad visual o motora contradice su propósito, no es un "nice to have" de UI.
- **Pruebas de regresión visual** (Playwright/Percy) específicamente sobre la vista pública de un expediente y su corrección/retractación (§16.3) — la regla de "misma prominencia visual" de §10.3 solo se puede garantizar en el tiempo con una prueba automática que falle si alguien reduce accidentalmente el tamaño del bloque de corrección en un refactor de CSS.

### 9.6 Datos de prueba seguros para privacidad (Ley 1581)

- **Nunca** usar datos reales de SIGEP/SECOP ni de usuarios reales en entornos de staging/desarrollo — fixtures sintéticas generadas (nombres, cédulas y cargos ficticios pero con la misma forma/distribución estadística que los datos reales) para que los tests de clasificación de campos (§16.2) sean representativos sin exponer datos personales fuera de producción.
- Entornos de staging con la misma política de acceso restringido que producción para cualquier dato que sí provenga de una fuente real (ej. muestras anonimizadas para depurar un bug reportado) — un incidente de privacidad en "solo staging" sigue siendo un incidente bajo la Ley 1581.

### 9.7 Pruebas de cumplimiento legal como suite propia

Un conjunto de tests explícitamente vinculado a §16, no mezclado con los tests funcionales genéricos, para que una revisión legal pueda auditar "qué prueba cada garantía":

- Un expediente con `Sensibilidad::Sensible` en cualquier campo **nunca** aparece en la proyección de lectura pública (10.1/16.2).
- Una corrección (`ActaRetractada`) se sirve por el mismo endpoint y con el mismo peso de payload/template que el señalamiento original (10.3/16.3).
- Un hecho marcado `HechoExcluidoPorCosaJuzgada` sin interés legítimo documentado no aparece en ninguna proyección (10.3/16.3).
- El hash recalculado de un evento con serialización canónica (§16.7) coincide siempre con el `hash_evento` almacenado — test de reproducibilidad que un perito judicial podría replicar literalmente.
- El worker de anonimización (§16.9) desindexa el contenido personal de las proyecciones públicas sin alterar el `merkle_root` ya anclado — prueba directa de que "restricción de circulación con conservación interna" (T-125/25) funciona como se prometió.

### 9.8 Pruebas continuas en producción

- **Monitoreo sintético**: transacciones canario ejecutadas cada pocos minutos contra producción (crear expediente de prueba en un tenant de test, firmarlo, verificar que se ancla) — detecta degradación real antes de que un usuario la reporte.
- **Despliegues canario/progresivos** (§12.3): un cambio se expone primero a un porcentaje pequeño de tráfico con monitoreo activo de tasa de error antes de completar el rollout — especialmente en el core de firmas/anclaje, donde un bug en producción tiene costo legal, no solo técnico.
- **Feature flags** (ya definidos en §12.3) permiten desactivar una funcionalidad social defectuosa sin un rollback completo del binario.

### 9.9 Reglas de calidad en CI

- CI bloquea merge si: fallan tests, `cargo audit`/`cargo deny` reporta CVE crítico sin mitigar, cobertura de `domain/` cae por debajo del umbral fijado, o el score de mutation testing (§9.3) sobre `adapters-crypto/` baja del umbral acordado.
- Los tests de integración/E2E corren en CI con contenedores efímeros, nunca contra ambientes compartidos.
- La suite de cumplimiento legal (§9.7) corre en cada PR que toque `domain/`, `adapters-crypto/` o las proyecciones de lectura — no solo en un pipeline nocturno, porque una regresión ahí es simultáneamente un bug y una exposición legal.


