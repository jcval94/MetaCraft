# Metadata Toolkit

`metadata` es un paquete de Python para enriquecer y validar esquemas YAML a partir de un `pandas.DataFrame`. Ahora `metadata.update()` permite leer YAML directamente desde URLs, igual que `pandas.read_csv`.

## Características

- **update**: enriquece YAML con estadísticas, *sketches* (`tdigest`, `HyperLogLog`) y deja los resultados en `metadata.df`.
- **validate**: comprueba la coherencia entre un DataFrame y el YAML (tipos, rangos, nulos...).
- **compare**: detecta *schema drift* entre dos esquemas.
- **export_schema**: convierte el YAML a otros formatos (Spark, SQL, etc.).
- **generate_expectations**: crea suites de *Great Expectations*.
- **transform**: devuelve un DataFrame ajustado al esquema.
- **quality_report**: puntaje sencillo de calidad (completitud + *drift*).
- **research**: usa OpenAI para explorar relaciones y anomalías.

## Instalación

```bash
pip install Metadata
```

O bien desde el repositorio:

```bash
pip install -r requirements.txt
```

Dependencias opcionales: `openai`, `tdigest`, `datasketch`.

## Ejemplo rápido

```python
import pandas as pd
from metadata import Metadata

# DataFrame de ejemplo
 df = pd.DataFrame({
     'survived': [0, 1, 1, 0],
     'age': [22, 38, 26, 35],
 })

# Esquema mínimo
yaml_schema = {
    'schema': [
        {'identity': {'name': 'survived'}},
        {'identity': {'name': 'age'}},
    ]
}

# Guardamos el YAML en disco
import yaml
with open('schema.yaml', 'w') as f:
    yaml.safe_dump(yaml_schema, f, sort_keys=False, allow_unicode=True)

m = Metadata()
m.update(df, 'schema.yaml', inplace=True)
```

### Resultados

```text
✔ schema.yaml actualizado
root
 |-- survived: integer (nullable = false)
 |-- age: integer (nullable = false)
<class 'metadata.dataset'>
Columns: 2 entries
 #   Column            Non-Null Count   Dtype
---  ------            --------------   -----
 0   survived                        4   integer
 1   age                             4   integer
dtypes: integer(2)
Validation passed: True
```

## Roadmap

- ✔️ Soporte de YAML remoto (v 2025‑07‑30)
- ⬜ Descarga de ZIP remotos
- ⬜ Caché local opcional
- ⬜ CLI (`metadata-cli update titanic.csv titanic.yaml`)

¡Contribuciones bienvenidas!
