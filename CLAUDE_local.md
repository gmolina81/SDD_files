# CLAUDE.md — Estándares Globales de Codificación Python

Este archivo aplica a todos los proyectos. Define los estándares base
que nunca cambian independientemente del tipo de proyecto.

---

## 1. Logging

Todo módulo debe incluir logging desde el inicio. Sin excepciones.

```python
import logging

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s",
    handlers=[
        logging.FileHandler("logs/pipeline.log"),
        logging.StreamHandler()
    ]
)
logger = logging.getLogger(__name__)
```

Reglas:
- `logger.info()` para inicio/fin de cada etapa y conteos de registros.
- `logger.warning()` para anomalías no fatales.
- `logger.error()` con traceback completo para errores fatales.
- Nunca usar `print()` en lugar de logging.

---

## 2. Manejo de Errores

- Siempre usar `try/except` en operaciones de I/O y conexiones externas.
- Nunca capturar `Exception` sin loguearlo con `logger.error(..., exc_info=True)`.
- Validar precondiciones antes de operar y lanzar errores descriptivos.

```python
required_columns = ["col_a", "col_b"]
missing = [col for col in required_columns if col not in df.columns]
if missing:
    raise ValueError(f"Columnas faltantes en {filename}: {missing}")
```

---

## 3. Tests con pytest

- Escribir los tests **antes** de implementar el código (TDD).
- Cada módulo en `src/` tiene su archivo de tests en `tests/` con prefijo `test_`.
- Cubrir caso feliz, casos borde y casos de error esperados.
- Los tests no deben depender de archivos reales ni conexiones externas.
- Ejecutar `pytest tests/` antes de considerar cualquier tarea completa.

---

## 4. Estructura de Funciones

- Type hints obligatorios en parámetros y retorno.
- Docstring obligatorio con descripción, args y returns.
- Máximo 50 líneas por función. Si supera ese límite, dividir.
- No usar variables de una sola letra salvo índices en loops simples.

---

## 5. Artefactos Obligatorios por Proyecto

Antes de escribir cualquier línea de código, el proyecto debe tener:

- **`SPEC.md`** — especificación técnica: fuentes, reglas de negocio, outputs y tests requeridos.
- **`requirements.txt`** — dependencias con versiones fijas.

Estos dos archivos son la fuente de verdad. Si no existen, crearlos antes de continuar.

---

## 6. Flujo de Trabajo — Aplicar en Todo Requerimiento

Seguir este orden sin saltear pasos:

```
1. Leer el SPEC.md completo antes de tocar cualquier archivo.
2. Usar Plan Mode (Shift+Tab) para planificar la implementación.
   - Identificar módulos a crear o modificar.
   - Identificar casos borde no contemplados en el SPEC.
   - Confirmar el plan con el usuario antes de continuar.
3. Escribir los tests primero (TDD).
   - Los tests deben fallar antes de implementar el código.
4. Implementar el código hasta que los tests pasen.
5. Validar con datos reales antes de dar la tarea por terminada.
```

---

## 7. Gestión de Cambios en el Requerimiento

Si el requerimiento cambia durante el desarrollo:

1. Detener la implementación.
2. Actualizar el `SPEC.md` primero.
3. Actualizar los tests para reflejar el nuevo comportamiento.
4. Recién entonces modificar el código.

> El código siempre debe ser consecuencia del SPEC, nunca al revés.
