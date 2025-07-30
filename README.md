# Metadata Toolkit (2025‑07‑30)

`metadata` es un paquete Python que ofrece un módulo `core` con la clase
`Metadata` para enriquecer, validar y explotar esquemas YAML a partir de un
`pandas.DataFrame`.

\*️⃣ **Novedad:** `metadata.update()` ahora acepta rutas HTTP/HTTPS del mismo modo
que `pandas.read_csv`, por lo que puedes trabajar con archivos YAML alojados en
GitHub Raw u otros servidores estáticos.

---

## Características clave

| Función                                     | Descripción                                                                                                                                          |
| ------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| `update`                                    | Enriquecer YAML(s) con estadísticas, *sketches* (`tdigest`, `HyperLogLog`) y caché plana (`metadata.df`). Soporta rutas locales, URLs y ZIP locales. |
| `validate`                                  | Comprueba coherencia entre DataFrame ↔ YAML (tipos, rangos, categorías, nulos, unicidad).                                                            |
| `compare`                                   | Detecta *schema drift* entre dos YAML o entre un snapshot y el estado actual.                                                                        |
| `export_schema`                             | Convierte el YAML a artefactos Spark, SQL o lo que indique un LLM.                                                                                   |
| `generate_expectations`                     | Genera suites de *Great Expectations* a partir del YAML.                                                                                             |
| `transform`                                 | Devuelve un DataFrame coercionado‑al‑esquema (tipos, nulos, columnas extra/faltantes).                                                               |
| `quality_report`                            | Puntaje simple de calidad (completitud + drift).                                                                                                     |
| `snapshot / load_snapshot / list_snapshots` | Control de versiones en memoria del esquema enriquecido.                                                                                             |
| `research`                                  | Usa OpenAI para explorar correlaciones, clusters y anomalías sobre una muestra del DF.                                                               |

---

## Instalación rápida

```bash
pip install Metadata
```

Si prefieres trabajar desde una copia local del repositorio:

```bash
pip install -r requirements.txt
```

### Dependencias opcionales

| Paquete      | Habilita…                                              |
| ------------ | ------------------------------------------------------ |
| `openai`     | Funciones de IA (`research`, `export_schema` con LLM). |
| `tdigest`    | Cálculo de cuantiles compactos en `statistics.sketch`. |
| `datasketch` | Estimación HLL de cardinalidad.                        |

```bash
pip install openai tdigest datasketch
```

## Estructura del proyecto

```
metadata/       # paquete principal
├─ __init__.py  # expone la clase Metadata
└─ core.py      # implementación original del módulo
LICENSE
pyproject.toml
requirements.txt
README.md
```

---

## Ejemplo mínimo

```python
import pandas as pd
from metadata import Metadata

df = pd.read_csv(
    "https://raw.githubusercontent.com/datasciencedojo/datasets/master/titanic.csv"
)
yaml_url = (
    "https://raw.githubusercontent.com/jcval94/Metadata_files/"
    "refs/heads/main/titanic.yaml"
)

metadata = Metadata()
metadata.update(df, yaml_url, inplace=False, verbose=True)   # ✔ /tmp/titanic.yaml actualizado

metadata.printSchema()
metadata.info()
metadata.validate(df, detail=1)
```

---

## Ejemplo avanzado de exploración

```python
from metadata import Metadata
import pandas as pd

# ─────── actualización y validación básicas ───────
metadata = Metadata()
metadata.update(df, "/content/titanic.yaml", inplace=True)
metadata.validate(df, message=True, capitalize=True)

metadata.info(), metadata.printSchema()

metadata.describe()

metadata.show('age')

display(metadata.df)

# ─────── filtrado y segmentación de variables ───────
md = metadata.df          # DataFrame de metadatos

# helper opcional: primera columna timestamp disponible
TS_COLS = [
    "actionability.last_change_event",     # la que sí existe
    "provenance.last_change_event",        # por si la añades después
]
ts_col = next((c for c in TS_COLS if c in md.columns), None)

# 1️⃣  continuas (float)
continuous = md.query("`type.logical_type` == 'float'")

# 2️⃣  continuas + discretas
cont_disc = md.query("`type.logical_type` in ['float', 'integer']")

# 3️⃣  variables objetivo
target = md[md['identity.tags'].apply(lambda t: isinstance(t, list) and 'target' in t)]

# 4️⃣  variables predictoras
predictors = md[md['identity.tags'].apply(lambda t: isinstance(t, list) and 'predictor' in t)]

# 5️⃣  tres más recientes (solo si existe ts_col)
if ts_col:
    recent3 = (md.assign(_ts=pd.to_datetime(md[ts_col], errors='coerce'))
                 .sort_values('_ts', ascending=False)
                 .head(3)
                 .drop(columns='_ts'))
else:
    recent3 = pd.DataFrame(columns=md.columns)   # vacío / placeholder

# 6️⃣  mínimo < 0
min_neg = md.query("`statistics.numeric_summary.min` < 0")

# 7️⃣  > 9 categorías
many_cats = md.query("`statistics.n_distinct` > 9")

# 8️⃣  2 con menor desviación estándar
low_std2 = md.nsmallest(2, 'statistics.numeric_summary.std')

# 9️⃣  variables fáciles de cambiar (dificultad ≤ 2)
easy_change = md[(md['actionability.increase_difficulty'] <= 2) &
                 (md['actionability.decrease_difficulty'] <= 2)]

# ─────── 5 filtros extra de ejemplo ───────

# A. más del 10 % de nulos
miss_gt10 = md.query("`statistics.missing_ratio` > 0.10")

# B. rango (max‑min) < 1
low_range = md[(md['type.logical_type'].isin(['float', 'integer'])) &
               ((md['statistics.numeric_summary.max'] -
                 md['statistics.numeric_summary.min']) < 1)]

# C. categóricas de dominio cerrado
closed_cat = md.query("`domain.categorical.closed` == True")

# D. top‑3 con mayor % nulos
top3_missing = md.nlargest(3, 'statistics.missing_ratio')

# E. columnas con patrón regex definido
patterned = md[md['domain.pattern'].notna() & (md['domain.pattern'] != '')]

# —— imprime nombres para verificar ——
for name, subset in {
        "continuous": continuous,
        "cont_disc": cont_disc,
        "target": target,
        "predictors": predictors,
        "recent3": recent3,
        "min_neg": min_neg,
        "many_cats": many_cats,
        "low_std2": low_std2,
        "easy_change": easy_change,
        "miss_gt10": miss_gt10,
        "low_range": low_range,
        "closed_cat": closed_cat,
        "top3_missing": top3_missing,
        "patterned": patterned}.items():
    print(f"{name}: {list(subset.index)}")
```

---

## Roadmap

* ✔️ Soporte YAML remoto (v 2025‑07‑30)
* ⬜ Descarga y procesado de ZIP remotos
* ⬜ Caché local opcional con expiración
* ⬜ CLI (`metadata-cli update titanic.csv titanic.yaml`)

Contribuciones y *pull requests* son bienvenidos. ¡Disfruta!
