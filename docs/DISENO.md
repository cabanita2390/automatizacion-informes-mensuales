# Diseño: automatización de informes mensuales de ejecución contractual

## Objetivo

Automatizar la elaboración del informe mensual de ejecución que exige un contrato de prestación de servicios con una entidad pública. El proceso manual consiste en redactar cada actividad, asignarla a una obligación del contrato, organizar las evidencias y armar una carpeta con un documento por obligación más un informe consolidado.

El sistema completo debe:

1. Leer las actividades registradas en Google Sheets.
2. Corregir la redacción y clasificar cada actividad en una obligación contractual con IA.
3. Pasar por una revisión humana obligatoria.
4. Organizar las evidencias en una estructura de carpetas por obligación.
5. Generar un documento Word por obligación.
6. Generar el informe mensual y comprimir la carpeta final para entrega.

Hoy están construidos los puntos 1 a 4 (parcialmente el 4). Los puntos 5 y 6 están diseñados en este documento.

---

## Estado de la implementación

El diseño original preveía 4 workflows. Lo construido se reparte así:

| Workflow del diseño | Implementado en | Estado |
| --- | --- | --- |
| 1. Datos (leer, filtrar por periodo, normalizar, separar evidencias) | Paso 1: `Leer_Actividades`, `Normalizar_Datos`, `Validar_Completitud` | Hecho. La hoja `CONFIG` no existe: el periodo está fijo en el código |
| 2. IA (corrección, título, clasificación, consolidación) | Paso 1: `IA_Correctora`, `Consolidar_Final` | Hecho |
| 3. Revisión (escribir `REVISION_IA`, pausa humana, leer aprobados) | Paso 1: `Escribir_REVISION_IA` · Paso 2: `LECTOR_APROBADOS_REVISION_IA` | Hecho. La pausa es manual: se ejecuta el paso 2 después de aprobar |
| 4. Generación (carpetas, evidencias, Word, informe, ZIP) | Paso 2: `GENERAR_RUTAS_CARPETAS`, `CREAR_CARPETA_ACTIVIDAD` | Parcial: crea carpetas por obligación y actividad con un `LEEME.txt` de control. No copia evidencias ni genera documentos |

---

## Stack

**Actual:** n8n en Docker, Google Sheets, OpenAI API (`gpt-4o-mini` por HTTP, salida JSON, temperatura 0), nodos Code en JavaScript, nodo Execute Command y una carpeta local montada en el contenedor para las evidencias.

**Ideas para una versión futura:** PostgreSQL como almacenamiento, OCR de evidencias, generación de Word con plantillas, configuración por hoja `CONFIG` y ejecución programada.

---

## Decisiones de arquitectura

- **Google Sheets como fuente maestra.** Es donde ya se registraban las actividades; evita un Excel local.
- **Evidencias locales, no en Drive.** Por tamaño se quedan en el disco local, en una ruta base por año:
  ```text
  <RUTA_BASE>\ESTADISTICA <AÑO>\
  ```
- **Validación humana obligatoria.** La IA solo propone; nada se genera sin `APROBADO`.
- **La IA no inventa categorías.** Solo clasifica entre las 12 obligaciones del contrato: no crea una 13, no renumera ni reformula.
- **Alertas explícitas en vez de fallos silenciosos.** Cada actividad lleva su lista de alertas hasta la hoja de revisión.
- **Trazabilidad.** Se conservan el texto original y el corregido, y la explicación de la clasificación.
- **Entradas saneadas antes del shell.** Los valores que llegan al nodo Execute Command (título, fecha, obligación) pasan por una lista blanca de caracteres, porque provienen de la hoja y de la IA.

---

## Pipeline

```text
Google Sheets (Actividades_2026)
↓ normalización y validación de completitud
↓ IA: corrección + título + clasificación
REVISION_IA (estado PENDIENTE)
↓ revisión humana
APROBADO
↓ creación de carpetas por obligación y actividad       ← hasta aquí está construido
↓ copia y renombrado de evidencias
↓ generación de OBLIGACION X.docx
↓ generación de INFORME_MENSUAL
↓ ZIP final
```

---

## Hojas de Google Sheets

### `Actividades_2026` (entrada)

| Columna | Ejemplo |
| --- | --- |
| `FECHA` | `jueves, abril 16, 2026` (texto) |
| `Actividad base` | `reunion de seguimiento con el equipo` |
| `EVIDENCIA` | Uno o varios nombres de archivo, uno por línea |
| `MES DEL INFORME` | `MAYO` |

### `REVISION_IA` (salida del paso 1, entrada del paso 2)

| Columna | Origen |
| --- | --- |
| `FECHA_ISO` | Fecha normalizada `AAAA-MM-DD` |
| `FECHA_ORIGINAL` | Texto original de la fecha |
| `MES_DEL_INFORME` | Hoja de entrada |
| `ACTIVIDAD_BASE` | Texto original |
| `ACTIVIDAD_CORREGIDA` | IA |
| `TITULO_ACTIVIDAD` | IA |
| `OBLIGACION_PROPUESTA` | IA (1 a 12) |
| `DESCRIPCION_OBLIGACION` | IA (copia de la descripción oficial) |
| `EVIDENCIAS_ORIGINALES` | Hoja de entrada |
| `ALERTAS` | Validación previa, separadas por coma |
| `ESTADO_REVISION` | `PENDIENTE` al escribir; la persona lo cambia a `APROBADO` o `RECHAZADO`. `ERROR` si la respuesta de la IA no se pudo leer |
| `EXPLICACION_CLASIFICACION` | IA |
| `OBSERVACIONES` | IA + alertas no mencionadas por la IA |
| `FUENTE` | Nombre de la hoja de origen |

Solo las filas con `APROBADO` continúan al paso 2.

---

## Alertas

| Alerta | Cuándo |
| --- | --- |
| `SIN_FECHA` | La fecha está vacía o no se puede interpretar. La actividad no se descarta |
| `SIN_EVIDENCIA` | La celda de evidencia está vacía |
| `EVIDENCIA_SIN_EXTENSION` | Un nombre de evidencia no tiene extensión |
| `DIA_RELLENADO_AUTOMATICAMENTE` + `REVISAR_ACTIVIDAD_MANUAL` | Día hábil del periodo sin actividades: se agrega "Seguimiento daily" para que la persona lo confirme o lo rechace |
| `ARCHIVO_NO_ENCONTRADO` | Diseñada para la copia de evidencias (no construida) |

No existe una alerta por evidencia repetida: una misma evidencia puede soportar varias actividades.

`Validar_Completitud` salta fines de semana y la lista de festivos configurada en el nodo.

---

## Diseño de lo pendiente

### Estructura final de entrega

```text
5. CARPETA MAYO
├── EVIDENCIAS
└── 5. MAYO
    └── GC_<ID>_<N>_<MES>_<AÑO>
        ├── INFORME_MENSUAL_DE_EJECUCION.docx
        ├── Obligacion 1
        │   ├── OBLIGACION 1.docx
        │   └── evidencias
        └── Obligacion 7
```

ZIP final: `GC_<ID>_<N>_<MES>_<AÑO>.zip`. La carpeta GC no se duplica, y el informe mensual se genera al final, cuando ya existen todas las carpetas, documentos y evidencias de cada obligación.

### Documento `OBLIGACION X.docx`

1. **Encabezado:** campos fijos (programa, NIT, responsable, lugar, elaboró) y el rango de fechas variable.
2. **Actividades**, repetidas por cada actividad aprobada:
   ```text
   Obligación X – Descripción de la actividad realizada

   Actividad:
   [Título corto]

   [Descripción corregida]

   REGISTRO FOTOGRÁFICO
   [Imágenes]
   ```
3. **Enlaces:** manual por ahora.

### Evidencias

- Imágenes (`.jpg`, `.jpeg`, `.png`): se copian, se renombran y se insertan en el Word.
- Otros (`.pdf`, `.xlsx`, `.docx`, `.zip`): se copian y renombran como soporte, sin insertarse.
- Una evidencia reutilizada se copia una sola vez por obligación; la actividad sí se repite en el documento.

Nomenclatura (el contador se reinicia por obligación y sigue el orden del documento):

```text
OBL07_ACT01
OBL07_ACT01_EVD01.jpg
OBL07_ACT01_EVD02.pdf
```

### Modelo de datos

```json
{
  "mes": "MAY",
  "anio": 2026,
  "gc_nombre": "GC_<ID>_<N>_<MES>_<AÑO>",
  "obligaciones": [
    {
      "obligacion": 8,
      "nombre_carpeta": "Obligacion 8",
      "documento": "OBLIGACION 8.docx",
      "actividades": [
        {
          "fecha": "2026-04-20",
          "actividad_base": "Seguimiento daily",
          "actividad_corregida": "",
          "titulo_actividad": "",
          "evidencias_originales": [],
          "evidencias_finales": [],
          "alertas": []
        }
      ]
    }
  ]
}
```
