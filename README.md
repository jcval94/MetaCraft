# Metadata Toolkit

`metadata` es un paquete de Python para enriquecer y validar esquemas YAML a partir de un `pandas.DataFrame`. Ahora `metadata.update()` permite leer YAML directamente desde URLs, e incluso descargar ZIP remotos con varios esquemas, igual que `pandas.read_csv`.

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
m.quality_report(df)
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
Quality score: 100.0 (A)
```

### Ejemplo con ZIP remoto

`metadata.update()` también puede procesar archivos ZIP alojados en la web. Basta con pasar la URL que termine en `.zip`:

```python
m.update(df, 'https://ejemplo.com/esquemas.zip', verbose=True)
```
Esto descargará el ZIP a un directorio temporal, aplicará las actualizaciones y dejará el archivo resultante en la misma carpeta (o en la ruta indicada con `output`).

### Edición de metadatos vía `metadata.df`

Tras `m.update()` el esquema queda en `m.df`, un DataFrame editable. Los cambios
pueden propagarse al YAML con `m.df.upgrade()`:

```python
# 1) Si todas las columnas son enteros
m.df['type.logical_type'] = 'integer'

# 2) Cambiar la descripción de `age`
m.df.loc['age', 'identity.description_i18n.es'] = 'Edad del pasajero'

# 3) Ajustar el rango permitido de `age`
m.df.loc['age', ['domain.numeric.min', 'domain.numeric.max']] = [0, 120]

m.df.upgrade('schema.yaml')  # guarda el YAML actualizado
```

## Roadmap

- ✔️ Soporte de YAML remoto (v 2025‑07‑30)
- ✔️ Descarga de ZIP remotos (v 2025‑07‑30)
- ⬜ Caché local opcional
- ⬜ CLI (`metadata-cli update titanic.csv titanic.yaml`)

¡Contribuciones bienvenidas!
