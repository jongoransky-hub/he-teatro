# Sistema HE — Teatro Metropolitan / Paseo La Plaza

Sistema de carga, aprobación, consolidación y pago de Horas Extras. Reemplaza las planillas sueltas por WhatsApp/mail. Arquitectura: archivos HTML estáticos en GitHub Pages + Google Apps Script como backend + Google Sheets como base de datos. Sin login — el control de acceso es "quién tiene el link", salvo la escritura de valores (PIN).

Repo: `github.com/jongoransky-hub/he-teatro`
Sheet: "Sistema HE Teatro"

## Filosofía del proyecto — leer antes de tocar código

- **Single-file HTML, sin build step.** Cada pieza es un `.html` autocontenido (CSS + JS embebido). Se edita directo y se sube a GitHub tal cual.
- **No hay autenticación real.** El acceso a cada pieza depende de a quién se le pasa el link. Esto es una decisión consciente, no un descuido — mantiene el sistema simple para un equipo de confianza. Si el equipo crece o el uso se vuelve más sensible, esto merece revisarse.
- **Los archivos "por rol" (jefe, coordinación) son plantillas compartidas.** Los 5 `jefe_*.html` son el mismo código con un bloque `JEFE_CONFIG` distinto arriba. `dashboard_coordinacion_met.html` y `dashboard_coordinacion_plp.html` son el mismo motor que `dashboard_HE_v9.html` con una constante `TEATRO_FIJO` distinta. **Regla:** cuando se corrige un bug de fondo, se corrige en el archivo base (el dashboard general, o `jefe_leo_munoz.html` como referencia) y se regeneran los derivados — nunca se parchea un derivado suelto, o empiezan a divergir sin que nadie lo note.
- **Patrón de bug ya visto tres veces:** guardar en `localStorage` (del dispositivo) algo que debería vivir en el servidor. Pasó con el historial del técnico, con la detección de solapamiento, y con el estado de "mes cerrado". Los tres ya están corregidos (leen del Sheet), pero si aparece un bug raro de "esto no coincide entre dispositivos", empezar a buscar por acá.
- **El mes no cambia solo el día 1 — cambia cuando coordinación lo cierra.** El período que muestra cada pieza es *el mes más viejo que todavía no se cerró* (por teatro), no el mes calendario. Septiembre se sigue viendo, cargando, aprobando y corrigiendo el 3, el 6 o el 15 de octubre, hasta que Carla/Javier tocan "Cerrar mes"; recién ahí aparece octubre. Regla de código: **ninguna vista debe usar `new Date()` para decidir qué mes mostrar o cerrar** — en los dashboards se usa `PERIODO` (ver `periodoPorDefecto()`), en los paneles de jefe `mesSel`/`mesesVisibles`, en el formulario `mesesAbiertosDe()`. La constante `PRIMER_MES` (`2026-09`) es el piso: nada anterior cuenta como "pendiente de cierre". Si no se puede leer la hoja "Meses", el respaldo es calendario con gracia hasta el día 15.
- **Nunca comparar una celda de mes o de fecha con `===` contra un texto.** Cuando el script guarda `"2026-09"` o `"2026-09-12"`, Sheets lo convierte solo en fecha. Por eso `cerrarMes` guardaba el mes como fecha y ningún chequeo de "mes cerrado" del servidor coincidía nunca. En el script, todo pasa por `mesKeyDe()` / `filasMeses()`; la columna "Mes" de la hoja Meses va con formato de texto. En los HTML, la clave de mes se recorta a 7 caracteres al leerla.
- **El tipo de producción se normaliza siempre a mayúscula.** La hoja Obras guarda `propia` / `cooperativa` / `tercero`; los paneles comparan contra `PROPIA` / `TERCERO`. Sin `normTipoObra()` el consolidado mandaba todo a "a descontar a terceros" y los mails de descuentos salían en $0. Para una obra que ya no figura en Obras, se usa el "Tipo Obra" grabado en el propio registro (`tipoObraDe()`).
- **Una cooperativa es un tercero.** Criterio de Jon (oct 2026): las HE de una producción tipo `cooperativa` se descuentan igual que las de un tercero. En el formulario se ofrecen en su propio grupo y se imputan a `otro_tercero`; en los paneles conservan la etiqueta "Cooperativa" pero suman en "a descontar a terceros", entran en el filtro "Terceros" y en los mails de descuentos (`esTercero()`).
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
| **Valores** | Tarifas **base** (vigencia en "Vigencia Desde"). 37 conceptos, con Teatro (AMBOS/MET/PLP), Solo Jefe, Solo Montaje. Fuente única — el formulario y Pieza 3 leen de acá, no hay nada hardcodeado. |
| **Ajustes** | Incrementos % generales programados por mes (Desde · Porcentaje · Nota). Se acumulan sobre el valor base según el mes de cada HE. Se cargan desde Pieza 3 → Actualizar → A. |
| **Historial** | Log de cambios de valores (quién, cuándo, qué). |
| **Meses** | Cierres de mes. Clave: Mes + Teatro (MET/PLP se cierran independiente; "AMBOS" es el formato viejo, previo a la separación por teatro). |
| **Log** | Auditoría general de acciones (altas, ediciones, errores). |

## Backend — Google Apps Script

Un solo script (`Código.gs`) atado a la planilla, publicado como Web App (`doGet`/`doPost`). El `SCRIPT_URL` es el mismo en las 10 piezas HTML — al redesplegar el script no hace falta tocar ningún HTML.

**Lecturas (GET):** `ping`, `getRegistros` (filtra por `mes`, `nombre`, `teatro`), `getPersonal`, `getObras` (filtra por `teatro`), `getValores`, `getHistorial`, `getMeses`, `getResumen`.

**Escrituras (POST):** `addRegistro` (rechaza con `codigo:"MES_CERRADO"` si el mes+teatro ya está cerrado, deja el ticket en el Log y avisa por mail a coordinación), `editRegistro` (bloqueado si el mes+teatro de ese registro ya está cerrado, o si se lo quiere mover a un mes cerrado), `addPersona`, `editPersona`, `addObra`, `editObra`, `updateValores` (requiere PIN), `addHistorial`, `cerrarMes` (requiere `teatro`; no admite doble cierre), `aprobarTicket` (bloqueado en mes cerrado), `avisarPagoListo`, `actualizarCC` (CC y "Sala asume"), `imputarObra` (mueve un registro a otra producción; bloqueado en mes cerrado), `marcarCasoEspecial`.

`getObras` devuelve las producciones vigentes desde el mes más viejo que siga abierto en algún teatro (`mesMasViejoAbierto()`), no desde el mes calendario; con `todas=1` (lo usa Pieza 3) devuelve todas, sin filtrar. `ping` informa la versión del script (`v3.3` desde noviembre 2026) — sirve para confirmar que la nueva implementación quedó publicada.

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

- **Oct 2026** — Corrección del cambio de mes. Todas las piezas tomaban el mes de `new Date()`: el 1° de octubre septiembre desaparecía de los dashboards, de los paneles de jefe y de "Mi historial" con las cargas todavía sin cerrar, y el botón "Cerrar mes" pasaba a apuntar a octubre. Ahora el período es el mes más viejo sin cerrar; los dashboards tienen un selector de mes y un cartel de estado arriba de todo, los jefes ven una pastilla por cada mes abierto y un desplegable "Meses cerrados" para consultar (solo lectura) lo ya cerrado, y el formulario no deja cargar en un mes ya cerrado para ese teatro. De paso: "Avisar que ya se puede cobrar" y "Ver este mes" quedaron también en el historial de cierres, y las producciones del formulario se filtran por la fecha del ticket (no por la fecha de hoy).

- **Oct 2026** — Apps Script v3 + segunda pasada de HTML. Servidor: el cierre de mes pasa a ser un bloqueo real (antes el mes se guardaba como fecha y no se reconocía como cerrado: se podía seguir editando, cerrar dos veces, y "Avisar que ya se puede cobrar" no encontraba el mes); `addRegistro` y `aprobarTicket` respetan el cierre; `getObras` no esconde producciones de un mes abierto; un `%` en cualquier texto ya no rompe la carga. Formulario: "¡Registrado!" solo si el servidor confirma (antes se mostraba aunque el servidor rechazara o no hubiera conexión). Paneles: propias/terceros bien clasificados.

- **Oct 2026** — Cooperativas habilitadas (antes el formulario no las ofrecía: 11 producciones de la planilla no se podían elegir). Pieza 3 pide `getObras?todas=1` y ve también las producciones terminadas o dadas de baja, para poder reactivarlas (antes desaparecían del panel al mes siguiente). Script v3.1: mail de coordinación MET cargado.

- **Oct 2026** — La hoja "Obras" pasa a ser la única fuente de producciones en los paneles. Antes se mezclaba con una lista vieja embebida en el HTML, y aparecían en los desplegables producciones que no existen en la planilla (Votemos, Momi, El Cuarto de Verónica, En Otras Palabras, K Rental…) con un tipo que no se podía corregir desde Pieza 3. La lista embebida queda solo como respaldo si la planilla no responde. **Para agregar una producción o cambiarle el tipo: Pieza 3 → Obras (o directo en la hoja); ningún HTML se toca.**

- **Oct 2026** — Imputación. En "Imputar", elegir una producción ahora **mueve el registro a esa producción** (acción `imputarObra` del script, v3.2): cambia Obra, Tipo Obra y CC, y guarda en la columna "Obra Original" con qué producción se había cargado. Antes ese desplegable solo grababa un código de CC —el mismo para todas las propias de un teatro—, así que el costo seguía sumando donde estaba. No manda mail a la persona ni figura como corrección. De paso: "Sala asume" se guarda (vivía solo en pantalla y se perdía en cada refresco), los totales del consolidado se calculan por registro (un solo registro con "Sala asume" arrastraba toda la producción a "asumido por sala") y lo que asume la sala no entra en los mails de descuento a terceros.

- **Oct 2026** — Imputar de a varios y buscadores. En "Imputar": buscador (persona, producción, descripción; sin distinguir acentos), orden o agrupado por persona / producción / fecha, y selección múltiple — se tildan registros sueltos, un grupo entero ("tildar estos N") o "los que se ven" tras una búsqueda, y se imputan todos a una producción desde la barra flotante. Usa la misma acción `imputarObra`, de a un registro, así que no requiere cambios en el script. Paneles de jefe: buscador y orden (pendientes primero / por persona con subtotales / por fecha); con el buscador activo, el resumen de arriba muestra los números de lo que se está viendo.

- **Oct 2026** — Detalle que justifica cada descuento. Todo lo que se comparte afuera muestra, por cada HE, día de la citación (con día de la semana), horario, horas, lugar (teatro · sala), tipo y la tarea realizada, agrupado por persona con subtotal: consolidado por obra (debajo de cada persona), mail de descuento a cada producción, mail a Gustavo (cada producción con su detalle), mail interno de archivo (detalle completo, con la producción de cada HE); el mail a RRHH y el de pago del día 15 quedan solo con montos —a RRHH y Tesorería les importan los montos— y remiten al PDF para el detalle y el PDF ("Detalle de cada HE", ordenado por producción y persona). Los registros anulados ($0) no se listan en los mails. El Excel de cierre que manda el script ya incluía horario y descripción: sin cambios en el script.

- **Oct 2026** — Mails cortos + ficha adjunta. El cuerpo de cada mail queda compacto: por persona, horas y monto, y debajo cada HE con fecha, horario, horas y monto (RRHH y pago del día 15, solo montos). El "por qué" va en la **ficha para adjuntar** que arma cada mail (botones "PDF para adjuntar" / "Excel para adjuntar" en Generar mails): la de esa producción (con espacio de conformidad para que la producción firme), la de todos los terceros para Gustavo, o la del mes completo para RRHH, pagos e interno. La ficha agrupa por producción y por persona, y por cada HE trae día, horario, horas, lugar, tipo, tarea realizada y monto; los anulados ($0) no se listan. El PDF del consolidado usa la misma ficha con el recorte de filtros activo.

- **Oct 2026** — Detalle de los mails alineado. Cada HE va en una sola línea con columnas fijas (día, horario, horas, monto). "Copiar texto" copia el mail con formato: al pegarlo en Gmail/Outlook el detalle llega como tabla, alineado de punta a punta; para programas que no aceptan formato, viaja también la versión texto.

- **Oct 2026** — Generar mails: los mails a cada producción externa (tercero o cooperativa) van en su propio renglón, "A cada producción externa", uno por cada producción con HE en el mes. Si en el mes no hay ninguna, el panel lo dice y remite a Imputar / CC (una HE cargada como propia que en realidad es de una producción externa se reimputa ahí, y su mail aparece).

- **Oct 2026** — Ficha PDF: cada persona es un encabezado bien marcado (nombre y total más grandes, barra a la izquierda, separación entre personas) y sus HE van debajo en letra más chica y montos en gris, para que se lea a simple vista cuántas personas hay y cómo se compone cada total.

- **Oct 2026** — Jefa de sala MET (Luciana Calarota): además de Infantiles, Evento corto y Evento largo, ve **"Función Adulto Matinee"** (paquete por función). Se agrega en el formulario a la lista de IDs técnicos que se le suman a la jefa de sala (`getValores`, ID `matinee`); el monto sale de la misma fila de la hoja "Valores" que usan Sonido/Iluminación/Maquinaria, así que si se actualiza esa fila se actualiza para todos. No se toca la hoja ni el script. Asistentes de sala no la ven (no es jefe); técnicos sin cambios.

- **Nov 2026** — Incrementos programados (Apps Script v3.3). Nueva pestaña **"Ajustes"** (Desde · Porcentaje · Nota): cada fila es un incremento % general que rige para las HE con fecha desde ese mes, y se acumulan. La hoja "Valores" queda como **valor base** (vigencia sep 2026) y ya no se pisa. **El monto lo decide el mes de la HE (fecha de la citación), no el día de carga**: una HE de octubre cargada el 2/11 va con valor de octubre. Programado: oct 2026 = base; +5% desde nov 2026 (dic y ene igual); +5% desde feb 2027 sobre el valor de enero (≈ +10,25% sobre octubre). **Valores en pesos enteros, sin centavos**: cada incremento redondea al peso más cercano y, ante un ,50 justo, hacia arriba (a favor del personal) — p. ej. 84.610 × 1,05 = 88.840,50 → 88.841. Pieza 3: la tabla muestra una columna por tramo (hoy y cada incremento futuro); "A — Incremento % general" ahora se **programa desde un mes** (con lista para quitar), en vez de pisar los valores en el momento; "B" corrige el valor base. No se puede programar ni quitar un incremento de un mes ya cerrado. Si el formulario no recibe ajustes (script viejo) usa el valor base, como antes.

- **Nov 2026** — Pieza 3 → Exportar y compartir → **"Planilla de valores del mes"**: la planilla que se manda al equipo cada mes, armada sola desde los valores base + los incrementos programados. Se elige el mes y para quién (Técnicos, Sala, Boletería, Mant. y Limpieza, Todos); muestra "VALORES HE PLP + MET", el mes (nov-26), "(5% incremento sobre OCT)" o "(sin incremento…)", el mes anterior en gris (opcional) y el valor del mes. Sale en PDF (A4), Excel o texto para pegar en WhatsApp/mail. Los conceptos de un solo teatro se marcan "(solo La Plaza / Metropolitan)". Acceso directo desde la Tabla de valores ("Planilla del mes →").

- **Nov 2026** — Pieza 3 → Exportar y compartir → **"Valores para producciones externas"**: el resumen de costos que se entrega a las producciones externas (formato del que armaba Mano). Título "VALORES HE <MES> <AÑO>", columna "VALORES (Del 01/mm/aa al dd/mm/aa)", sin porcentajes ni comparación con meses anteriores, y las tres notas en rojo (maquinista de guardia, montaje de maquinaria, horario de ingreso 18:00 / mínimos de 3 y 6 horas). PDF, Excel ("Valores HE MET - Nov 26") o texto. Los renglones y de qué fila de Valores sale cada uno están en `EXTERNA_ITEMS` dentro de pieza3 (Hora Extra Técnicos = he3, Madrugada = hem3, Jefe de área = he3j, Programación = prog3, Microfonista y Eléctrico de piso = evco, Guardia proyección = sproy, Armados = armado, Luces M/L = lm/ll); las notas en `EXTERNA_NOTAS`. Las notas se pueden retocar en la vista previa para una exportación puntual (no se guarda).

## Pendiente / no resuelto todavía

- Pieza 3 (valores, obras, personal) sigue compartida entre MET y PLP — no está separada por coordinador.
- Sin autenticación real en ningún archivo — el control de acceso es la distribución de links.
- `EMAIL_COORDINADOR.PLP` sigue vacío en el script (falta el mail de Javier): los mails "al coordinador" de Paseo La Plaza (resumen de cierre con Excel, recordatorios del 28 y del 5, aviso de carga rechazada) le llegan a Jon. Los del MET ya van a Carla, con Jon en copia.
- Apps Script no tiene versionado formal en uso (existe la función "Administrar implementaciones" pero no se está aprovechando para poder volver atrás fácil).
