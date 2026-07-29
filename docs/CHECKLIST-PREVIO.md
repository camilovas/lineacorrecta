# Checklist previo a programar — La Línea Correcta

> Extraído de `ARQUITECTURA.md` original. Las referencias `§N` conservan la numeración original del documento monolítico — ver [INDICE.md](INDICE.md) para ubicar cada sección en su archivo actual.

## 18. Checklist previo a comenzar la programación

Todo lo anterior es diseño. Esta sección es la lista de lo que debe existir **antes de que se escriba la primera línea de código de negocio** — omitir estos pasos no ahorra tiempo, lo traslada (más caro) a después del primer incidente.

### 18.1 Modelado de amenazas formal (no solo este documento)

- Sesión de **threat modeling** estructurada (STRIDE o similar) sobre los flujos críticos ya identificados: firmar un acta, anclar un lote, retirar contenido por orden judicial (§16.5), rectificación (§16.3). Este documento describe las mitigaciones ya decididas; el ejercicio formal existe para encontrar las que *no* se han pensado — debe involucrar a alguien fuera del equipo que escribió la arquitectura.
- Cada hallazgo del threat modeling se convierte en un ítem de backlog **antes** del sprint 1, no en un "ya lo veremos".

### 18.2 Objetivo de verificación de seguridad explícito

- Fijar por escrito el **nivel ASVS objetivo** por componente (§6.4: Nivel 2 general, Nivel 3 en `adapters-crypto/` y gestión de llaves) — sin esto, "seguro" no es medible y cada desarrollador lo interpreta distinto.
- Traducir los requisitos ASVS aplicables en **criterios de aceptación** de las primeras historias técnicas (autenticación, autorización, gestión de sesión) — no un documento aparte que nadie vuelve a abrir.

### 18.3 Higiene de secretos y entornos desde el día uno

- Gestor de secretos (Vault, o al menos secrets nativos del PaaS de §12.2) configurado **antes** del primer commit con credenciales reales — nunca empezar con `.env` en el repo "temporalmente".
- **Separación completa de entornos**: dev/staging/producción con credenciales, llaves de firma y bases de datos distintas desde el inicio — migrar de "todo compartido" a esto después es mucho más costoso y arriesgado que empezar bien.
- `.gitignore` y hooks de pre-commit que bloqueen commitear secretos (`gitleaks` o equivalente) configurados en el commit inicial del repositorio, no añadidos tras un susto.

### 18.4 Pipeline y control de acceso al repositorio

- **Branch protection** en `main` desde el primer commit: sin push directo, revisión obligatoria de otra persona, CI en verde obligatorio (§6.3, §12.3).
- CI configurado para ejecutar `cargo audit`/`cargo deny`, lint de seguridad (`clippy::unwrap_used` denegado, §6.5) y la suite de tests de cumplimiento legal (§9.7) **desde el primer pipeline funcional**, no agregado "cuando haya tiempo" — un proyecto que nace sin estos gates los adopta después con mucha más fricción organizacional.
- Definir quién tiene acceso de despliegue a producción (lista corta, nombrada, revisada) antes de que exista un entorno de producción al que desplegar.

### 18.5 Estándares de código y definición de "hecho"

- Guía de estilo y lint compartida (`rustfmt.toml`, `clippy.toml`) commiteada antes del primer PR de feature — evita debates de estilo retroactivos sobre código ya escrito.
- **Definición de "hecho" (Definition of Done)** que incluya explícitamente: tests unitarios + de integración pasan, cobertura de `domain/` dentro del umbral (§9.5), sin nuevas advertencias de `clippy`, sin secretos en el diff, y — para cualquier cambio a `domain/`, `adapters-crypto/` o proyecciones de lectura — la suite de cumplimiento legal de §9.7 en verde.
- Plantilla de PR que obligue a declarar explícitamente si el cambio toca datos personales, firmas, o el event store — dispara automáticamente una revisión adicional cuando aplique (four-eyes de §15.4).

### 18.6 Roles y puntos de control humano

- Designar (aunque sea a tiempo parcial al inicio) un responsable de seguridad/**security champion** dentro del equipo — alguien con mandato explícito de bloquear un merge por motivos de seguridad, no solo "buena voluntad" difusa del equipo.
- Confirmar el punto de contacto de **asesoría legal externa** (§10, §16.10) antes del lanzamiento, no durante el primer incidente — varias decisiones de arquitectura (nivel ASVS, retención de datos, clasificación de campos) dependen de validación legal que debería idealmente preceder, no seguir, la implementación.
- Acordar el **plan de respuesta a incidentes** (§15.5) como documento vivo desde el inicio — un plan escrito por primera vez durante un incidente real ya llegó tarde.

### 18.7 Orden de implementación recomendado

No todo se construye en paralelo. Orden sugerido que respeta las dependencias reales entre secciones de este documento:

1. **Esqueleto hexagonal** (§2) con crates vacíos pero con los traits de puertos ya definidos — fija los límites de dependencia antes de que haya código que los viole.
2. **Núcleo de evidencia mínimo**: máquina de estados (§16.1), event store (§11.1), firma/verificación Ed25519 (§15.1) con tests de propiedad (§9.1) — sin esto no hay producto, y es donde más caro sale improvisar después.
3. **Pipeline de ingesta con clasificación de campos** (§16.2) — antes de conectar cualquier fuente real de datos (SIGEP/SECOP), para no ingerir nunca un dato sensible aunque sea en un ambiente de prueba.
4. **Anclaje blockchain** (§8) como worker aislado, detrás del trait `AnclajeService` — puede empezar sobre una red de pruebas mientras el resto del sistema madura.
5. **Capa social** (§7, `domain-social`) — deliberadamente después del núcleo, porque depende de él (evento `ActaSellada`) y tiene menor riesgo legal individual.
6. **HumanGuard y capa de borde** (§15.2) — se integra cuando ya existen flujos reales que proteger, no antes.
7. **Suite de cumplimiento legal (§9.7) y checklist operativo (§16.10)** — corren en paralelo a todo lo anterior desde el paso 2, no al final como "certificación" de cierre.


