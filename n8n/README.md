# Restaurar y ejecutar los workflows en n8n

Probado importando ambos workflows en n8n 2.20 (Docker).

## 1. Levantar n8n con Docker (PowerShell)

```powershell
docker run -it --rm --name n8n -p 5678:5678 `
  -e NODES_EXCLUDE='[]' `
  -v n8n_data:/home/node/.n8n `
  -v "<RUTA_LOCAL_DE_EVIDENCIAS>:/home/node/datos" `
  docker.n8n.io/n8nio/n8n
```

- `NODES_EXCLUDE='[]'` habilita el nodo Execute Command del paso 2. Desde n8n 2.0 viene deshabilitado por defecto; actívalo solo en un entorno local de confianza.
- `<RUTA_LOCAL_DE_EVIDENCIAS>` es una carpeta de tu equipo (por ejemplo `C:\demo\evidencias`). El paso 2 escribe en `/home/node/datos`, que queda reflejado ahí.

## 2. Preparar el Google Sheet

Crea un libro con dos hojas e importa los CSV de [`../ejemplos`](../ejemplos) (Archivo → Importar → *Insertar hojas nuevas*, **desmarcando** "Convertir texto en números, fechas y fórmulas" para que la columna `FECHA` quede como texto):

| Hoja | Archivo | Contenido |
| --- | --- | --- |
| `Actividades_2026` | `actividades_ejemplo.csv` | Actividades de prueba, con casos de alerta (sin fecha, sin evidencia, sin extensión, fuera de rango) |
| `REVISION_IA` | `revision_ia_encabezados.csv` | Solo encabezados; el paso 1 agrega las filas |

`FECHA` debe tener el formato `jueves, abril 16, 2026`.

## 3. Importar y configurar

1. Abrir `http://localhost:5678` e importar cada JSON con **Import from file**.
2. Credenciales:
   - **Google Sheets (OAuth2)** en los nodos de Google Sheets.
   - **OpenAI** en `IA_Correctora`: credencial *Header Auth* con nombre `Authorization` y valor `Bearer` + tu clave.
3. Reemplazar `<ID_DE_TU_GOOGLE_SHEET>` en los nodos de Google Sheets.
4. En el prompt de `IA_Correctora`, reemplazar los marcadores `[Descripción oficial de la obligación N, ...]` y `[criterio específico ...]`. Para una demo puedes usar las obligaciones ficticias de [`../ejemplos/obligaciones_ejemplo.md`](../ejemplos/obligaciones_ejemplo.md).
5. El rango del informe está fijo en `Normalizar_Datos` y `Validar_Completitud` (16-04-2026 a 13-05-2026). Si lo cambias, cambia ambos nodos.

## 4. Ejecutar

1. **Paso 1** → *Execute workflow*. Con los datos de ejemplo hace unas 20 llamadas a `gpt-4o-mini`: 6 actividades + 14 días hábiles sin registro que `Validar_Completitud` rellena como "Seguimiento daily" con alertas. Todas quedan en `REVISION_IA` con `ESTADO_REVISION = PENDIENTE`.
2. En `REVISION_IA`, revisar y cambiar a `APROBADO` las filas que correspondan.
3. **Paso 2** → *Execute workflow*. Crea `OBLIGACION_<N>/<fecha>_<título>/LEEME.txt` en la carpeta de evidencias solo para las filas aprobadas.

Ambos workflows arrancan con disparador manual y se exportan inactivos. Cada nodo tiene notas que explican qué hace.
