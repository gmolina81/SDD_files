# CLAUDE.md — Proyecto: Enriquecimiento de Modelo de Crédito

Este archivo extiende el CLAUDE.md global con las convenciones
específicas de este proyecto. No repetir aquí lo que ya está en el global.

---

## Tipo de Proyecto

Exploración y transformación de datos con **Jupyter Notebook**.
La lógica reutilizable vive en `src/`, el notebook solo orquesta e importa.

---

## Estructura de Carpetas

```
proyecto/
├── data/
│   ├── raw/          # Archivos Excel originales — NO modificar
│   └── processed/    # Outputs generados por el pipeline
├── logs/
├── notebooks/
│   └── pipeline_enrichment.ipynb
├── src/
│   ├── normalizacion.py
│   ├── extraccion_periodo.py
│   └── merge.py
├── tests/
├── CLAUDE.md
├── SPEC.md
└── requirements.txt
```

---

## Librerías del Proyecto

- Transformación de datos: `pandas`
- Lectura de Excel: `openpyxl`
- Notebooks: `jupyter`
- Tests: `pytest`
- Variables de entorno: `python-dotenv`

---

## Convenciones Específicas

- El notebook **nunca define funciones**. Solo importa desde `src/` y muestra métricas.
- Cada celda del notebook termina imprimiendo conteos y métricas del paso ejecutado.
- Las columnas de cédula de cada fuente se renombran a `cedula_normalizada` antes de cualquier merge.
- Prefijos obligatorios en columnas de proveedores: `seon_` y `riskseal_`.
- El campo de join temporal es `loan_disbursed_date` del modelo de crédito.

---

## Referencia

Ver `SPEC.md` para fuentes de datos, reglas de negocio, lógica temporal del match y tests requeridos.
