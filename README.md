# Sistema HE — Teatro Metropolitan / Paseo La Plaza

Sistema de carga, aprobación, consolidación y pago de Horas Extras. Reemplaza las planillas sueltas por WhatsApp/mail. Arquitectura: archivos HTML estáticos en GitHub Pages + Google Apps Script como backend + Google Sheets como base de datos. Sin login — el control de acceso es "quién tiene el link", salvo la escritura de valores (PIN).

Repo: `github.com/jongoransky-hub/he-teatro`
Sheet: "Sistema HE Teatro"

## Filosofía del proyecto — leer antes de tocar código

- **Single-file HTML, sin build step.** Cada pieza es un `.html` autocontenido (CSS + JS embebido). Se edita directo y se sube a GitHub tal cual.
- **No hay autenticación real.** El acceso a cada pieza depende de a quién se le pasa el link. Esto es una decisión consciente, no un descuido — mantiene el sistema simple para un equipo de confianza. Si el equipo crece o el uso se vuelve más sensible, esto merece revisarse.
- **Los archivos "por rol" (jefe, coordinación) son plantillas compartidas.** Los 5 `jefe_*.html` son el mismo código con un bloque `JEFE_CONFIG` distinto arriba. `dashboard_coordinacion_met.html` y `dashboard_coordinacion_plp.html` son el mismo motor que `dashboard_HE_v9.html` con una constante `TEATRO_FIJO` distinta. **Regla:** cuando se corrige un bug de fondo, se corrige en el archivo base (el dashboard general, o `jefe_leo_munoz.html` como referencia) y se regeneran los derivados — nunca se parchea un derivado suelto, o empiezan a divergir sin que nadie lo note.
- **Patrón de bug ya visto tres veces:** guardar en `localStorage` (del dispositivo) algo que debería vivir en el servidor. Pasó con el historial del técnico, con la detección de solapamiento, y con el estado de "mes cerrado". Los tres ya están corregidos (leen del Sheet), pero si aparece un bug raro de "esto no coincide entre dispositivos", empezar a buscar por acá.
- **Nunca confiar en una actualización de UI antes de confirmar el backend.** Varias funciones actualizaban la pantalla como "hecho" antes de que el POST confirmara éxito — si el servidor rechazaba (ej. mes cerrado), la pantalla mentía. Se corrigió en Pieza 3 (valores) y en los paneles de jefe (aprobar/rechazar/editar). Si se agrega una acción nueva que escribe al backend, seguir ese mismo patrón: escribir primero, mutar la UI solo si `result.ok`.

## Mapa de archivos

| Archivo | Para quién | Alcance |
|---|---|---|
| `formulario_HE_v7.html` | Todo el personal | Carga de HE — MET y PLP |
| `dashboard_HE_v9.html` | Jon | Ve y puede cerrar MET y PLP, todo junto o filtrado |
| `dashboard_coordinacion_met.html` | Carla | Solo Teatro Metropolitan — todas las vistas y el cierre de mes |
| `dashboard_coordinacion_plp.html` | Javier | Solo Paseo La Plaza — todas las vistas y el cierre de mes |
| `jefe_abal.html` | Pablo Abal (Sonido) | Aprobar/rechazar su equipo, cargar propio |
| `jefe_gambetta.html` | Juan Gambetta (Maquinaria, MET) | ídem |
| `jefe_la_rosa.html` | Horacio La Rosa (Iluminación) | ídem |
| `jefe_lanza.html` | Damián Lanza (Maquinaria, PLP) | ídem |
| `jefe_leo_munoz.html` | Leo Muñoz (Iluminación, MET) | ídem |
| `pieza3_director_v5.html` | Jon / Dirección | Valores (con PIN), obras, personal |

URLs: `https://jongoransky-hub.github.io/he-teatro/<archivo>`

## La hoja de cálculo — pestañas

| Pestaña | Para qué |
|---|---|
| **Registros** | Cada HE cargada. Una fila por ticket. |
| **Personal** | Nómina: nombre, área(s), teatro, jefe, activo, validado. |
| **Obras** | Producciones: nombre, teatro, tipo (propia/cooperativa/tercero), vigencia. |
| **Valores** | Tarifas vigentes. 37 conceptos, con Teatro (AMBOS/MET/PLP), Solo Jefe, Solo Montaje. Fuente única — el formulario y Pieza 3 leen de acá, no hay nada hardcodeado. |
| **Historial** | Log de cambios de valores (quién, cuándo, qué). |
| **Meses** | Cierres de mes. Clave: Mes + Teatro (MET/PLP se cierran independiente; "AMBOS" es el formato viejo, previo a la separación por teatro). |
| **Log** | Auditoría general de acciones (altas, ediciones, errores). |

## Backend — Google Apps Script

Un solo script (`Código.gs`) atado a la planilla, publicado como Web App (`doGet`/`doPost`). El `SCRIPT_URL` es el mismo en las 10 piezas HTML — al redesplegar el script no hace falta tocar ningún HTML.

**Lecturas (GET):** `ping`, `getRegistros` (filtra por `mes`, `nombre`, `teatro`), `getPersonal`, `getObras` (filtra por `teatro`), `getValores`, `getHistorial`, `getMeses`, `getResumen`.

**Escrituras (POST):** `addRegistro`, `editRegistro` (bloqueado si el mes+teatro de ese registro ya está cerrado), `addPersona`, `editPersona`, `addObra`, `editObra`, `updateValores` (requiere PIN), `addHistorial`, `cerrarMes` (requiere `teatro`), `aprobarTicket`.

**PIN de valores:** `1342`, constante `PIN_VALORES` en el script. Solo protege la escritura en Pieza 3 — la lectura de valores es abierta, como todo lo demás en este sistema.

## Cómo actualizar el backend

1. Editor de Apps Script → pegar el `.gs` completo (reemplaza todo) → guardar.
2. Si el cambio agrega o modifica una función de migración puntual (ej. `migrarValoresSeptiembre`, `migrarMesesConTeatro`), correrla una sola vez desde el desplegable de funciones → ▶ Ejecutar. **Ejecutar ≠ Implementar** — son botones distintos, son pasos separados.
3. Implementar → Nueva versión, para que la Web App sirva el código nuevo. El `SCRIPT_URL` no cambia con esto.

## Cómo actualizar un archivo HTML

Renombrar el archivo nuevo para que coincida exactamente con el nombre que ya está en GitHub (los links ya repartidos dependen del nombre exacto) → en el repo, "Add file → Upload files" → arrastrar → Commit. GitHub Pages tarda 1-2 minutos en reflejar el cambio; probar en ventana de incógnito para evitar caché del navegador.

## Historial de decisiones grandes

- **Sep 2026** — Migración de valores hardcodeados en el formulario a la hoja "Valores" como fuente única (`migrarValoresSeptiembre`). Separación de tarifas por teatro (Early Show solo PLP, Boletería distinta por teatro).
- **Sep 2026** — Corrección del flag `overrideOverlap`: el formulario lo mandaba mal mapeado (`overrideHorario`) y nunca marcaba un solapamiento confirmado como tal.
- **Sep 2026** — Separación de "Cierre de mes" por teatro (antes era uno solo para todo el sistema). Bloqueo real de edición post-cierre en el backend (antes el cierre era solo un cartel visual).
- **Sep 2026** — Corrección de tres bugs de "estado guardado en el dispositivo en vez del servidor": historial del técnico, detección de solapamiento, estado de mes cerrado.

## Pendiente / no resuelto todavía

- Pieza 3 (valores, obras, personal) sigue compartida entre MET y PLP — no está separada por coordinador.
- Sin autenticación real en ningún archivo — el control de acceso es la distribución de links.
- Apps Script no tiene versionado formal en uso (existe la función "Administrar implementaciones" pero no se está aprovechando para poder volver atrás fácil).
