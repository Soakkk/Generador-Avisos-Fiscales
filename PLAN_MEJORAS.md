# Plan de mejoras — Generador de Avisos Fiscales

Documento de trabajo para esta sesión y las siguientes. Registra qué se ha decidido,
qué está hecho y qué queda pendiente, con el razonamiento detrás de cada punto para
que cualquier sesión futura (o IA) pueda continuar sin perder contexto.

Última actualización: 2026-07-07.

---

## 1. Verificador de datos (doble check de lo que lee Gemini)

**Problema**: Gemini 2.5 Flash a veces lee mal un dígito del IBAN, del importe o del
NIF. Un error en la cuenta de cargo o el importe en un aviso enviado a un cliente es
grave para la asesoría.

**Solución en dos capas** (decidido):

1. **Validación determinista (instantánea, sin coste, en `src/validation.ts`)**:
   - IBAN: checksum oficial mod-97 (detecta cualquier dígito mal leído). Formato ES + longitud 24.
   - NIF/NIE/CIF: algoritmo oficial de la letra de control (detecta OCR erróneo del número o la letra).
   - Importe: número finito, aviso si es 0 o inusualmente alto (>300.000 €).
   - Periodo: debe ser 1T-4T o 01-12. Modelo: contra lista de modelos AEAT conocidos.
   - Nombre: no vacío, no "Cliente Desconocido", sin caracteres extraños.
2. **Segunda lectura con IA** (`POST /api/gemini/verify-tax` en `server.ts`):
   se reenvía la misma captura con un prompt distinto centrado en transcribir
   dígito a dígito los campos críticos (IBAN, importe, NIF). El servidor compara
   ambas lecturas y devuelve las discrepancias campo a campo. Si las dos lecturas
   coinciden Y el checksum pasa, la probabilidad de error es mínima.

**UI**: cada aviso muestra un estado de verificación:
- ✅ «Verificado» — checksums OK y segunda lectura coincidente.
- ⚠️ «Revisar: IBAN/importe/NIF» — algún checksum falla o las dos lecturas difieren;
  el campo concreto queda marcado en el aviso y en el editor.
- El estado se guarda en `TaxNotice.verificacion` (campo nuevo en `types.ts`).

**Estado**: `validation.ts` + endpoint + UI → ver sección «Registro de cambios».

---

## 2. Multi-impuesto (130 + 303) y almacenaje de capturas

**Problemas detectados**:
- Las capturas se guardan como PNG base64 **completo** dentro de `localStorage`
  (`aeat_raw_notices`). localStorage tiene un límite (~5 MB): con 2-4 capturas de
  pantalla completa se supera, `setItem` lanza `QuotaExceededError` (hoy sin
  capturar) y **se dejan de guardar avisos silenciosamente**. Es el mayor riesgo
  real de pérdida de datos de la app.
- La agrupación por cliente usa el NIF tal cual lo lee Gemini; si en la segunda
  captura lee el NIF con un espacio o guion, el mismo cliente aparece dos veces
  (dos avisos separados en vez de uno unificado).

**Solución (decidido)**:
- **Capturas a disco, no a localStorage**: el servidor guarda la imagen original en
  `%USERPROFILE%\.generador-avisos-fiscales\capturas\<id>.png`
  (`POST /api/capturas`, `GET /api/capturas/:id`, `DELETE /api/capturas/:id`).
  En localStorage solo se guarda una **miniatura JPEG comprimida** (~10-30 KB,
  máx. 640 px) para el carrusel de miniaturas + el id del archivo en disco.
- **Limpieza**: al borrar un aviso/cliente o «Limpiar todo», se borran también sus
  archivos de captura. Barrido al arrancar el servidor: se eliminan capturas
  huérfanas de más de 90 días.
- **Agrupación robusta**: normalizar el NIF como clave (quitar espacios/guiones,
  mayúsculas) y, si el NIF falla el checksum pero el nombre normalizado coincide
  con un grupo existente, agrupar por nombre.
- Migración suave: los avisos antiguos con base64 grande siguen funcionando; al
  cargarse se re-comprimen a miniatura para liberar espacio.

---

## 3. Texto colapsado en la ficha exportada (imagen)

**Causa raíz**: `NoticeCard.tsx` usa `whiteSpace: 'nowrap'` + `textOverflow: 'ellipsis'`
en el nombre del impuesto, cliente y varias filas; y el importe grande (26-36 px) no
se adapta a su longitud. Con nombres largos o importes de 6 cifras el texto se corta
con «…» o se solapa, y en la exportación (html-to-image, que renderiza aparte) se
nota más que en pantalla.

**Solución (decidido)**:
- Permitir salto de línea (wrap) en: nombre del impuesto, nombre del cliente y
  desglose de modelos. Mantener `nowrap` solo en importes, fechas e IBAN (longitud
  acotada y no deben partirse).
- Tamaño del importe **auto-ajustado** según su nº de caracteres (p. ej. 36 px hasta
  9 caracteres, bajando escalonado hasta 22 px).
- Quitar `textOverflow: ellipsis` de todos los campos que llevan datos del cliente
  (jamás debe cortarse un dato con «…» en un documento enviado al cliente).
- Probar los 3 formatos (A/B/C) con: nombre de 60+ caracteres, importe de 7 cifras,
  desglose de 4 impuestos, IBAN presente/ausente.

---

## 4. API key: link directo al navegador + comprobación

**Causa raíz**: la app corre dentro de Electron; el enlace `target="_blank"` a
`aistudio.google.com/apikey` abre una ventana Electron en blanco (o nada) porque
`main-electron.cjs` no define `setWindowOpenHandler`. Hay que interceptar los
`window.open` y delegarlos en `shell.openExternal` (navegador del sistema).

**Solución (decidido)**:
- `main-electron.cjs`: `mainWindow.webContents.setWindowOpenHandler(...)` →
  `shell.openExternal(url)` para URLs https externas.
- `ApiKeySettings.tsx`: botón grande y claro «Crear una clave gratis en Google AI
  Studio» (no un link pequeño en el texto), con instrucciones paso a paso.
- Botón **«Probar clave»**: nuevo endpoint `POST /api/config/test` que hace una
  llamada mínima a Gemini y confirma si la clave guardada funciona de verdad
  (hoy solo se comprueba que hay algo guardado, no que sea válida).

---

## 5. Tamaño y distribución de las herramientas en la app

**Observaciones**: casi toda la interfaz usa letra de 10-12 px (pequeña para uso
diario de oficina); la columna izquierda mezcla zona de pegado, datos de la
asesoría y la tarjeta informativa de plazos con el mismo peso visual; la cabecera
acumula botones (Ajustes, Ejemplo, Limpiar) sin jerarquía.

**Criterio a aplicar** (primera pasada; **iterar con feedback del usuario**):
- La acción principal (pegar captura) debe ser lo más grande y visible.
- Subir el tamaño base de textos y botones de acción (de text-xs a text-sm en
  controles principales).
- La tarjeta de «Plazos AEAT» es informativa → puede compactarse o ser plegable.
- Ventana de Electron: tamaño por defecto 1280×860 y mínimo 1000×700.

**⚠️ Este punto es subjetivo**: hacer una primera pasada razonable y pedir al
usuario capturas/opiniones para ajustar en la siguiente sesión.

---

## Registro de cambios de esta sesión (2026-07-07)

Se va rellenando conforme se completa cada punto:

- [x] Auditoría completa del código (App.tsx, server.ts, NoticeCard, NoticeEditor, ApiKeySettings, main-electron.cjs, types.ts).
- [x] 1. Verificador de datos: `src/validation.ts` (14 tests de checksums pasados),
      endpoint `/api/gemini/verify-tax` (segunda lectura + comparación campo a campo),
      badges «Datos verificados / Revisar datos / Sin verificar» por cliente y panel
      con el detalle de cada problema. Al editar a mano se recalcula.
- [x] 2. Capturas a disco (`/api/capturas`, con limpieza al borrar avisos, barrido de
      +90 días y protección de rutas) + miniatura JPEG en localStorage + migración de
      avisos antiguos + agrupación por NIF normalizado (checksums verificados en preview).
      Extra: arreglado un bug por el que quitar un impuesto en el editor no lo
      eliminaba de verdad al guardar, y el guardado en localStorage ahora avisa si
      se queda sin espacio en vez de fallar en silencio.
- [x] 3. Ficha: sin recortes «…», nombre y desglose con salto de línea, importe
      auto-ajustado (verificado sin desbordamientos en los formatos A/B/C con nombre
      de 79 caracteres e importe de 7 cifras).
- [x] 4. API key: `setWindowOpenHandler` + `shell.openExternal` en Electron (los links
      abren el navegador real), botón grande «Crear una clave gratis en Google AI
      Studio», instrucciones paso a paso y botón «Probar clave» contra Gemini
      (`POST /api/config/test`).
- [x] 5. Primera pasada de UI: ventana 1280×860 (mín. 1000×700), botón principal de
      pegado más grande, texto de WhatsApp a 13px, botones de copia más visibles.
      **Pendiente de feedback del usuario para iterar.**
- [x] Extra: el puerto del servidor acepta la variable `PORT` en desarrollo (el 3000
      seguía ocupado por la app instalada mientras se desarrollaba).

## Pendiente para próximas sesiones

- **Iterar el punto 5 (UI)** con capturas reales del usuario: qué herramientas usa
  más, cuáles estorban, tamaños cómodos en su monitor.
- **Actualizar `INSTRUCTIONS_Y_CONTEXTO_IA.md`**: aún dice que se usa
  `gemini-3.5-flash` (el código real usa `gemini-2.5-flash` porque 3.5 se colgaba
  con imágenes) y no menciona el verificador ni el almacenaje de capturas en disco.
- **Historial**: hoy los avisos viven solo como «activos» hasta que se limpian.
  Valorar un historial consultable (como el de AvisosClientes) con búsqueda por cliente.
- **Copias de seguridad**: botón «Exportar/importar avisos» (JSON) por si se cambia de PC.
- **Probar la cuota de Gemini con la doble lectura activada**: cada captura pasa a
  consumir 2 llamadas; si la cuota gratuita se queda corta, hacer la segunda lectura
  opcional (interruptor en Ajustes).

## Cómo publicar una versión nueva (recordatorio)

```bash
# 1. Subir "version" en package.json (p. ej. 1.1.0)
# 2. Compilar y publicar en GitHub Releases:
npm run electron:publish   # usa GH_TOKEN; publica en Soakkk/Generador-Avisos-Fiscales-releases
# 3. Commit + push del código a Soakkk/Generador-Avisos-Fiscales
```
