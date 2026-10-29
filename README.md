# Automatización de informes mensuales con n8n e IA

Automatiza la elaboración del informe mensual de ejecución de un contrato de prestación de servicios: toma las actividades registradas en Google Sheets, corrige su redacción y las clasifica con IA, pasa por una revisión humana obligatoria y prepara la estructura de carpetas por obligación.

## Estado
Prototipo funcional parcial, usado de forma manual durante un periodo contractual. Está construida la mitad inicial del flujo: 2 workflows que cubren la lectura, la IA, la revisión humana y la creación de carpetas. La generación de documentos está diseñada pero no construida.

| Etapa | Estado |
| --- | --- |
| Lectura de actividades desde Google Sheets | Hecho |
| Normalización, alertas y relleno de días hábiles sin registro (JavaScript) | Hecho |
| Corrección de redacción y clasificación con OpenAI (`gpt-4o-mini`) | Hecho |
| Hoja de revisión humana (`REVISION_IA`, solo avanza lo aprobado) | Hecho |
| Creación de carpetas por obligación y actividad (con un archivo de control) | Hecho |
| Copia y renombrado de evidencias | Diseñado, sin construir |
| Generación de documentos Word por obligación | Diseñado, sin construir |
| Generación del informe mensual y ZIP final | Diseñado, sin construir |

El periodo del informe (16-04-2026 a 13-05-2026) está fijo en el código de dos nodos.

## Flujo
```
Google Sheets → normalización y alertas → IA (corrección + título + clasificación)
→ REVISION_IA → aprobación humana → lectura de aprobados → carpetas por obligación
```

## Stack
n8n (Docker) · Google Sheets · OpenAI API (`gpt-4o-mini`, salida JSON, temperatura 0) · nodos Code en JavaScript · nodo Execute Command

## Cómo probarlo
[`n8n/README.md`](n8n/README.md) explica cómo levantar n8n, importar los workflows y cargar los datos de [`ejemplos/`](ejemplos), que incluyen obligaciones ficticias para la demo.

## Estructura
```
├── n8n/
│   ├── paso-1-correccion-y-clasificacion-ia.json
│   ├── paso-2-estructura-de-carpetas.json
│   └── README.md                     cómo restaurar y ejecutar los workflows
├── ejemplos/
│   ├── actividades_ejemplo.csv       hoja de entrada con casos de alerta
│   ├── revision_ia_encabezados.csv   encabezados de la hoja de revisión
│   └── obligaciones_ejemplo.md       obligaciones ficticias para el prompt
└── docs/DISENO.md                    arquitectura, hojas, alertas y diseño de lo pendiente
```

## Decisiones de diseño
- Validación humana obligatoria antes de generar cualquier documento.
- La IA no inventa categorías: clasifica solo entre las obligaciones definidas en el contrato y explica su clasificación.
- Alertas explícitas por actividad (sin fecha, sin evidencia, evidencia sin extensión, día rellenado) en lugar de fallos silenciosos. Si la respuesta de la IA no se puede leer, la fila queda en estado `ERROR`.
- Trazabilidad: cada actividad conserva su texto original y el corregido.
- Los valores que llegan al comando de shell pasan por una lista blanca de caracteres, porque vienen de la hoja y de la IA.

## Nota sobre los datos
Los datos del contrato (descripciones de obligaciones, identificadores, rutas, credenciales e IDs de hojas) fueron reemplazados por marcadores como `<ID_DE_TU_GOOGLE_SHEET>`. Para usarlo hay que completar esos valores y crear las credenciales en n8n.
