# PLAN.md — Implementación del Pipeline de Enriquecimiento

## Estado del Proyecto al Iniciar

### Directorios existentes
- `src/` — vacío (.gitkeep)
- `tests/` — vacío (.gitkeep)
- `utils/` — vacío (.gitkeep)
- `logs/` — vacío (.gitkeep)
- `data/raw/` — 8 archivos fuente + modelo de crédito
- `data/processed/` — vacío (.gitkeep)

### Faltante
- `notebooks/` — crear
- `requirements.txt` — crear

---

## Fases de Implementación

### Fase 1 — Scaffolding
- Crear directorio `notebooks/`
- Crear `requirements.txt` con versiones fijas:
  - `pandas`, `openpyxl`, `jupyter`, `pytest`, `python-dotenv`

> `utils/` ya existe pero el SPEC no lo menciona. No se toca salvo indicación explícita.

---

### Fase 2 — Tests primero (TDD)

Todos los tests deben estar escritos y en **rojo** antes de implementar el código.

**`tests/test_normalizacion.py`**
- `test_normalizar_cedula` — todos los formatos (V-, J-, E-, espacios, mayúsculas)
- `test_normalizar_cedula_float` — entrada `12345678.0` → `12345678`

**`tests/test_extraccion_periodo.py`**
- `test_extraer_periodo_oct` — `SEON-Oct 25.xlsx` → `{"meses": [10], "anio": 2025}`
- `test_extraer_periodo_ago_sep` — `SEON-AGO_SEP 25.xltx` → `{"meses": [8, 9], "anio": 2025}`
- `test_extraer_periodo_invalido` — nombre sin patrón reconocible → `ValueError`

**`tests/test_merge.py`**
- `test_merge_caso_feliz` — cédula y período coinciden → merge correcto
- `test_merge_credito_fuera_periodo` — crédito de 2023 → NaN en columnas de proveedor
- `test_merge_cliente_multiples_meses` — cliente con créditos en sep y oct → cada uno matchea solo su mes
- `test_merge_sin_match_cedula` — cédula no existe en proveedor → NaN
- `test_merge_ago_sep_ambos_meses` — créditos de ago y sep matchean con archivo AGO_SEP
- `test_prefijos_columnas` — columnas tienen prefijos `seon_` y `riskseal_` correctos
- `test_colision_columnas` — detecta columnas con el mismo nombre entre proveedores
- `test_columna_cedula_faltante` — archivo sin columna de cédula → `ValueError`
- `test_duplicados_proveedor` — cédula duplicada → mantiene primer registro y loguea
- `test_huella_fuentes` — columna toma valores correctos (SEON, RISK_SEAL, AMBOS, NINGUNO)
- `test_merge_independiente` — SEON y Risk Seal mergeados independientemente contra el modelo base producen el mismo resultado que cualquier orden de aplicación; el resultado de uno no contamina al otro

---

### Fase 3 — Módulos `src/`

Implementar en este orden hasta que los tests pasen.

#### `src/normalizacion.py`
```python
def normalizar_cedula(valor: Any) -> str
```
Aplica las 7 reglas del SPEC en orden:
1. Convertir a string
2. Strip de espacios
3. Convertir a mayúsculas
4. Eliminar guiones
5. Eliminar prefijos V, J, E (con o sin guión)
6. Eliminar decimales si viene como float (`12345678.0` → `12345678`)
7. El resultado debe ser solo dígitos

#### `src/extraccion_periodo.py`
```python
def extraer_periodo_desde_nombre(nombre_archivo: str) -> dict
# Retorna: {"meses": [int, ...], "anio": int}
```
Mapeo de meses en español: `AGO→8, SEP→9, OCT→10, NOV→11, DIC→12`.
Lanza `ValueError` descriptivo si el patrón no es reconocible.

#### `src/merge.py`
Dos funciones:

```python
def leer_y_normalizar_proveedor(
    filepath: str,
    col_cedula: str,   # "id" para SEON, "reference_id" para Risk Seal
    prefijo: str       # "seon_" o "riskseal_"
) -> tuple[pd.DataFrame, dict]
```
- Lee el archivo Excel/XLTX
- Valida existencia de `col_cedula` → `ValueError` si falta
- Normaliza cédulas → `cedula_normalizada`
- Aplica prefijo a todas las columnas excepto `cedula_normalizada`
- Deduplica por `cedula_normalizada` (mantiene primer registro, loguea cantidad)
- Extrae período desde el nombre del archivo
- Retorna `(df_normalizado, periodo_dict)`

```python
def aplicar_merge_proveedor(
    modelo_df: pd.DataFrame,
    lista_proveedor: list[tuple[pd.DataFrame, dict]],
) -> pd.DataFrame
```
- `modelo_df` es **siempre el modelo base original** — nunca el resultado de un merge previo con otro proveedor.
- Para cada `(proveedor_df, periodo)`:
  - Filtra filas del modelo donde `mes_credito` in `periodo["meses"]` y `año_credito == periodo["anio"]`
  - LEFT JOIN sobre `cedula_normalizada`
  - Acumula resultado en el dataset completo
- Retorna el modelo completo con columnas del proveedor (NaN donde no hay match o fuera de período)

> **Patrón de uso obligatorio en el notebook:**
> ```python
> modelo_con_seon      = aplicar_merge_proveedor(modelo_base, lista_seon)
> modelo_con_riskseal  = aplicar_merge_proveedor(modelo_base, lista_riskseal)  # modelo_base, no modelo_con_seon
> ```

---

### Fase 4 — Validación de Tests
```bash
pytest tests/ -v
```
Todos los tests deben estar en **verde** antes de continuar al notebook.

---

### Fase 5 — Notebook `notebooks/pipeline_enrichment.ipynb`

11 celdas según el SPEC. Ninguna define funciones: solo importa desde `src/` y muestra métricas.

| Celda | Descripción |
|---|---|
| 0 | Configuración, imports, paths, logging |
| 1 | Lectura del modelo de crédito + validación de columnas |
| 2 | Normalización de cédulas del modelo |
| 3 | Extracción de período desde `loan_disbursed_date` |
| 4 | Lectura y normalización de archivos SEON |
| 5 | Merge con SEON (lógica temporal) |
| 6 | Lectura y normalización de archivos Risk Seal |
| 7 | Merge con Risk Seal — LEFT JOIN contra `modelo_base` original (no contra resultado de Celda 5) |
| 8 | Consolidación: unir columnas SEON y Risk Seal sobre `modelo_base`; ambos joins independientes; generar `huella_fuentes` |
| 9 | Reporte final de calidad |
| 10 | Escritura de outputs (Excel + CSV) en `data/processed/` |

---

## Pregunta Abierta Pendiente

- [ ] ¿Los archivos `.xltx` se pueden leer directamente con openpyxl o requieren conversión?
  - Opción A: Verificar antes de implementar (inspeccionar los archivos ahora)
  - Opción B: Manejar dentro de la implementación con fallback y `logger.error` si falla

---

## Orden de Ejecución Resumido

```
1. Crear notebooks/ y requirements.txt
2. Escribir tests/ (todos en rojo)
3. src/normalizacion.py → pytest test_normalizacion.py en verde
4. src/extraccion_periodo.py → pytest test_extraccion_periodo.py en verde
5. src/merge.py → pytest test_merge.py en verde
6. pytest tests/ -v (confirmación final: todos en verde)
7. notebooks/pipeline_enrichment.ipynb
```
