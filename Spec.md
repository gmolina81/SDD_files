# SPEC.md — Pipeline de Enriquecimiento de Modelo de Crédito

## Objetivo

Construir un pipeline de enriquecimiento de datos en Python implementado como un
**Jupyter Notebook**, donde cada etapa del ETL es una celda independiente que muestra
sus métricas y conteos antes de continuar. Las funciones reutilizables y los tests
viven en archivos `.py` separados.

El pipeline toma un modelo de crédito en Excel y lo enriquece con datos de huella
digital de dos proveedores (SEON y Risk Seal), usando la cédula/RIF como clave de
join respetando la ventana temporal de cada archivo. El output es un dataset
enriquecido en Excel y CSV.

---

## Archivos de Entrada

### Modelo de Crédito
| Archivo | Descripción |
|---|---|
| `credit_model_30_dataset.xlsx` | Fuente base. Contiene créditos históricos, datos de clientes y transacciones. |

### Proveedor SEON
| Archivo | Período |
|---|---|
| `SEON-AGO_SEP 25.xltx` | Agosto–Septiembre 2025 |
| `SEON-Oct 25.xlsx` | Octubre 2025 |
| `SEON-NOV 25.xltx` | Noviembre 2025 |
| `SEON-DIC 25.xltx` | Diciembre 2025 |

### Proveedor Risk Seal
| Archivo | Período |
|---|---|
| `Risk Seal-AGO_SEP 25.xltx` | Agosto–Septiembre 2025 |
| `Risk Seal-Oct 25.xlsx` | Octubre 2025 |
| `Risk Seal-NOV 25.xltx` | Noviembre 2025 |
| `Risk Seal-DIC 25.xltx` | Diciembre 2025 |

> **Nota:** Los archivos `.xltx` son plantillas Excel. Tratarlos igual que `.xlsx` en la lectura con openpyxl.

---

## Columnas Clave

| Fuente | Columna de Cédula | Columna de Fecha |
|---|---|---|
| Modelo de crédito | `id_number` | `loan_disbursed_date` |
| SEON | `id` | *(inferida desde nombre de archivo)* |
| Risk Seal | `reference_id` | *(inferida desde nombre de archivo)* |

> Antes de cualquier merge, renombrar las columnas de cédula de cada fuente a `cedula_normalizada` para unificar el campo de join.

---

## Reglas de Negocio

### Normalización de Cédula/RIF

Aplicar en orden sobre `id_number`, `id` y `reference_id` antes de cualquier merge:

1. Convertir a string (Excel puede leerlo como número entero o float)
2. Strip de espacios al inicio y fin
3. Convertir a mayúsculas
4. Eliminar guiones
5. Eliminar prefijos: V, J, E, R (con o sin guión)
6. Eliminar decimales si viene como float (ej: `12345678.0` → `12345678`)
7. El resultado debe ser solo dígitos

**Ejemplos:**
| Input | Output normalizado |
|---|---|
| `V-12345678` | `12345678` |
| `V12345678` | `12345678` |
| `12345678` | `12345678` |
| `12345678.0` | `12345678` |
| ` v-12345678 ` | `12345678` |
| `J-123456789` | `123456789` |
| `R147941346` | `147941346` |

### Patrón de Nombre de Archivo y Extracción de Período

El patrón es: `{Proveedor}-{Periodo}.{extensión}`

El período se extrae del segmento entre `-` y `.` del nombre del archivo:

| Segmento extraído | Mes(es) | Año |
|---|---|---|
| `Oct 25` | 10 | 2025 |
| `NOV 25` | 11 | 2025 |
| `DIC 25` | 12 | 2025 |
| `AGO_SEP 25` | 8 y 9 | 2025 |

> Si el segmento no puede mapearse a un período conocido → lanzar `ValueError` descriptivo y detener el pipeline.

### Lógica Temporal del Match

- El campo `loan_disbursed_date` determina en qué ventana cae cada crédito.
- Un registro de proveedor solo puede matchear contra créditos cuya `loan_disbursed_date` caiga dentro del período del archivo.
- Créditos de años o meses fuera del período disponible **no reciben huella**.
- Si un cliente tiene créditos en múltiples meses, cada crédito matchea únicamente con el archivo del período correspondiente.

### Fallback Temporal

Si una cédula de persona natural (V, E, R) no tiene match exacto en el 
período del crédito, aplicar fallback:
- Buscar la cédula en todos los demás archivos del mismo proveedor.
- Si existe en más de un archivo, usar la huella del archivo más reciente.
- Agregar columna `huella_periodo_origen` con el período del archivo 
  que proveyó la huella. Valores posibles:
  - `EXACTO` — match en el período correspondiente al crédito
  - `FALLBACK_{periodo}` — match encontrado en archivo de otro período
  - `SIN_MATCH` — cédula no encontrada en ningún archivo del proveedor

Las cédulas tipo J (jurídicas) no aplican fallback. Se clasifican 
directamente como `huella_fuentes = NO_APLICA`.

**Ejemplo:**
- `SEON-Oct 25.xlsx` → solo matchea con créditos donde `loan_disbursed_date` esté en octubre 2025.
- `SEON-AGO_SEP 25.xlsx` → matchea con créditos de agosto 2025 o septiembre 2025.
- Crédito otorgado en marzo 2023 → no recibe huella.

### Tipo de Join

- **Modelo ← SEON:** LEFT JOIN por `cedula_normalizada` + período de `loan_disbursed_date`
- **Modelo ← Risk Seal:** LEFT JOIN por `cedula_normalizada` + período de `loan_disbursed_date`
- Créditos sin huella quedan con `NaN` en columnas del proveedor (no se descartan).

### Columnas del Output

- Todas las columnas del modelo de crédito.
- Columnas de SEON con prefijo `seon_` (para evitar colisiones).
- Columnas de Risk Seal con prefijo `riskseal_` (para evitar colisiones).
- Excluir columnas completamente vacías o ilegibles.
- Agregar columna huella_fuentes: valores SEON, RISK_SEAL, AMBOS, NINGUNO, NO_APLICA.

### Manejo de Estructura Variable

- Si la columna de cédula requerida no existe → `ValueError` con nombre de archivo y columna faltante.
- Si columnas opcionales no existen → `logger.warning()` y continuar.

### Manejo de Duplicados

- Cédula duplicada dentro del mismo archivo de proveedor → mantener primer registro y loguear cantidad.

---

## Estructura del Proyecto

```
proyecto/
├── data/
│   ├── raw/                  # Archivos Excel originales — NO modificar
│   └── processed/            # Outputs generados por el pipeline
├── logs/                     # Logs generados automáticamente
├── notebooks/
│   └── pipeline_enrichment.ipynb   # Notebook principal del ETL
├── src/
│   ├── normalizacion.py      # Función normalizar_cedula()
│   ├── extraccion_periodo.py # Función extraer_periodo_desde_nombre()
│   └── merge.py              # Funciones de merge con lógica temporal
├── tests/
│   ├── test_normalizacion.py
│   ├── test_extraccion_periodo.py
│   └── test_merge.py
├── CLAUDE.md
└── SPEC.md
```

---

## Estructura del Notebook

Cada celda tiene un título Markdown, el código de la etapa, y al final imprime
sus métricas. El notebook **no define funciones**: las importa desde `src/`.

### Celda 0 — Configuración e Imports
```
- Imports de librerías y módulos de src/
- Definición de paths a archivos de entrada
- Configuración de logging
```
**Output:** confirmación de paths encontrados y versiones de librerías.

### Celda 1 — Lectura del Modelo de Crédito
```
- Leer credit_model_30_dataset.xlsx
- Validar columnas requeridas: id_number, loan_disbursed_date
- Loguear columnas encontradas
```
**Output:**
- Total de registros leídos
- Rango de fechas de loan_disbursed_date (min, max)
- Cantidad de valores nulos en id_number

### Celda 2 — Normalización de Cédulas del Modelo
```
- Aplicar normalizar_cedula() sobre id_number → cedula_normalizada
- Detectar y loguear cédulas que no pudieron normalizarse
```
**Output:**
- Registros normalizados exitosamente
- Registros con cédula inválida o vacía (y ejemplos)
- Cédulas duplicadas dentro del modelo

### Celda 3 — Extracción de Período desde loan_disbursed_date
```
- Parsear loan_disbursed_date → columnas mes_credito y año_credito
- Clasificar registros: dentro o fuera del período disponible (ago–dic 2025)
```
**Output:**
- Distribución de créditos por mes/año
- Registros dentro del período disponible
- Registros fuera del período (no recibirán huella)

### Celda 4 — Lectura y Normalización de Archivos SEON
```
- Para cada archivo SEON:
  - Inferir período desde nombre de archivo
  - Leer y detectar columnas
  - Normalizar cédulas (id → cedula_normalizada)
  - Aplicar prefijo seon_
```
**Output por archivo:**
- Nombre del archivo y período inferido
- Total de registros leídos
- Registros con cédula inválida
- Cédulas duplicadas

### Celda 5 — Merge con SEON
```
- Para cada archivo SEON:
  - Filtrar créditos del modelo que caen en ese período
  - LEFT JOIN por cedula_normalizada
  - Acumular resultado
```
**Output por archivo:**
- Créditos elegibles para ese período
- Matches encontrados (y %)
- Créditos sin match en SEON

### Celda 5a — Control de No Matches SEON
- Analizar registros sin match exacto en SEON
- Clasificar por tipo de cédula (J, V, E, R)
- Identificar cédulas V/E/R candidatas a fallback

**Output**
- Total sin match por período
- Distribución por tipo de cédula
- Cédulas V/E/R sin match (candidatas a fallback)

### Celda 6 — Lectura y Normalización de Archivos Risk Seal
```
- Mismo proceso que Celda 4 pero para Risk Seal (reference_id → cedula_normalizada)
```
**Output:** mismas métricas que Celda 4.

### Celda 7 — Merge con Risk Seal
```
- Para cada archivo Risk Seal:
  - Filtrar créditos del modelo BASE ORIGINAL que caen en ese período
  - LEFT JOIN por cedula_normalizada contra el modelo base original (no contra el resultado de Celda 5)
  - Acumular resultado
```
**Output:** mismas métricas que Celda 5.

### Celda 7a — Control de No Matches Risk Seal
- Mismo análisis que Celda 5b pero para Risk Seal

**Output:** mismas métricas que Celda 5a.

### Celda 8 — Consolidación y columna huella_fuentes
```
- Partir del modelo base original
- Unir columnas SEON (resultado de Celda 5) sobre el modelo base con LEFT JOIN por cedula_normalizada
- Unir columnas Risk Seal (resultado de Celda 7) sobre el modelo base con LEFT JOIN por cedula_normalizada
- Ambos joins son independientes: ninguno usa como tabla izquierda el resultado del otro
- Generar columna huella_fuentes
- Excluir columnas vacías o ilegibles
- Verificar colisión de columnas entre proveedores
```
**Output:**
- Total de registros en el dataset final
- Distribución de huella_fuentes (SEON / RISK_SEAL / AMBOS / NINGUNO con %)
- Columnas excluidas por estar vacías

### Celda 9 — Reporte Final de Calidad
```
- Resumen ejecutivo del pipeline completo
```
**Output:**
- Total registros modelo de crédito
- Registros en período disponible vs. fuera del período
- Cobertura de huella por proveedor
- Archivos procesados exitosamente vs. con errores

### Celda 10 — Escritura de Outputs
```
- Guardar modelo_enriquecido_YYYYMMDD.xlsx en data/processed/
- Guardar modelo_enriquecido_YYYYMMDD.csv (UTF-8 BOM) en data/processed/
```
**Output:**
- Paths absolutos de los archivos generados
- Tamaño en filas y columnas de cada archivo

---

## Tests Requeridos (pytest — archivo separado)

| Archivo | Test | Descripción |
|---|---|---|
| `test_normalizacion.py` | `test_normalizar_cedula` | Todos los formatos posibles |
| | `test_normalizar_cedula_float` | Excel convirtió la cédula a float |
| `test_extraccion_periodo.py` | `test_extraer_periodo_oct` | `SEON-Oct 25.xlsx` → mes 10, año 2025 |
| | `test_extraer_periodo_ago_sep` | `SEON-AGO_SEP 25.xlsx` → meses [8, 9], año 2025 |
| | `test_extraer_periodo_invalido` | Nombre sin período reconocible → ValueError |
| `test_merge.py` | `test_merge_caso_feliz` | Cédula y período coinciden → merge correcto |
| | `test_merge_credito_fuera_periodo` | Crédito de 2023 → no recibe huella |
| | `test_merge_cliente_multiples_meses` | Cliente con créditos en sep y oct → cada uno matchea solo su mes |
| | `test_merge_sin_match_cedula` | Cédula no existe en proveedor → NaN |
| | `test_merge_ago_sep_ambos_meses` | Créditos de ago y sep matchean con archivo AGO_SEP |
| | `test_prefijos_columnas` | Columnas tienen prefijos seon_ y riskseal_ correctos |
| | `test_colision_columnas` | Detecta columnas con el mismo nombre entre proveedores |
| | `test_columna_cedula_faltante` | Archivo sin columna de cédula → ValueError |
| | `test_duplicados_proveedor` | Cédula duplicada → mantiene primer registro y loguea |
| | `test_huella_fuentes` | Columna toma valores correctos (SEON, RISK_SEAL, AMBOS, NINGUNO) |
| | `test_fallback_cedula_periodo_distinto` | Cédula sin match exacto pero existe en otro archivo → usa huella más reciente |
| | `test_fallback_multiples_archivos` | Cédula en dos archivos → usa el más reciente |
| | `test_juridica_no_aplica` | Cédula J → huella_fuentes = NO_APLICA |

---

## Preguntas Abiertas

- [X] ¿Los archivos `.xltx` se pueden leer directamente con openpyxl o requieren conversión previa? *(verificar en el entorno antes de implementar Celda 4 y 6)*
- [X] Confirmar que SEON y Risk Seal no comparten nombres de columna *(el pipeline lo verifica automáticamente en Celda 8)*