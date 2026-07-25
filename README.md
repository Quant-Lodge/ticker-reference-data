# ticker-reference-data

Reference data de tickers US, regenerada automáticamente cada día.

Dos datasets derivados de la data de referencia de [Polygon.io](https://polygon.io)
más correcciones propias (splits que el vendor perdió, resolución de renames por CIK):

| Archivo | Qué es | Tamaño |
|---|---|---|
| [`data/fresh_universe.csv`](data/fresh_universe.csv) | tickers "frescos": IPO o reverse split en los últimos 365 días | ~95 KB |
| [`data/ticker_changes.json`](data/ticker_changes.json) | renames de ticker resueltos por CIK | ~1.1 MB |

## Consumo

Cada día se publica un **release tageado con el `as_of`** del snapshot. Los assets
de un tag son inmutables, así que la URL identifica el contenido sin ambigüedad:

```bash
# El snapshot de una fecha concreta. 200 = es ese día, garantizado. 404 = todavía
# no se publicó (y tu proceso sabe que tiene que seguir con su copia anterior).
curl -sSfL https://github.com/Quant-Lodge/ticker-reference-data/releases/download/2026-07-24/fresh_universe.csv -o fresh_universe.csv

# El último publicado, sea cual sea.
curl -sSfL https://github.com/Quant-Lodge/ticker-reference-data/releases/latest/download/fresh_universe.csv -o fresh_universe.csv
```

**Preferí los releases para consumo automatizado.** También están los archivos en
`data/`, pero se sirven por `raw.githubusercontent.com`, que apunta a una rama
mutable y por eso cachea (`max-age=300`): justo después de un push podés recibir
la versión anterior. Medido sirviendo contenido de 218 s atrás. Para leerlos a
ojo desde el navegador da igual; para un proceso que necesita el dato del día, no.

```
https://raw.githubusercontent.com/Quant-Lodge/ticker-reference-data/main/data/fresh_universe.csv
https://raw.githubusercontent.com/Quant-Lodge/ticker-reference-data/main/data/ticker_changes.json
```

## `fresh_universe.csv`

Un ticker califica como fresco si `rs_days ≤ 365` **o** `ipo_days_eff ≤ 365`.

```csv
ticker,rs_days,ipo_days_eff,fresh_via,as_of_date
AAAC,,225,ipo,2026-07-24
AAHYX,228,,rs,2026-07-24
ABTC,18,324,both,2026-07-24
```

| Columna | Tipo | Descripción |
|---|---|---|
| `ticker` | string | símbolo |
| `rs_days` | int \| vacío | días desde el reverse split más reciente; vacío si no tuvo |
| `ipo_days_eff` | int \| vacío | días desde el IPO efectivo; vacío si no se pudo determinar |
| `fresh_via` | `ipo` \| `rs` \| `both` | qué condición lo califica |
| `as_of_date` | `YYYY-MM-DD` | fecha del snapshot, idéntica en todas las filas |

Una columna puede exceder los 365 días si la **otra** es la que califica: `ABTC`
entra por su reverse split de hace 18 días aunque su IPO sea de hace 324.

Corte típico: ~3,800 tickers (~2,700 por IPO, ~1,000 por reverse split, ~110 por ambos).

**El universo no está filtrado por precio, volumen ni tradeabilidad** — incluye
OTC, fondos y units. Filtrá según tu caso de uso.

`ipo_days_eff` es *efectivo*, no la fecha de listado literal: incorpora el inicio
del bloque de trading continuo, de modo que un ticker reciclado tras un delisting
no aparece con la antigüedad del emisor anterior.

## `ticker_changes.json`

Mapa `ticker` → resolución del rename, con el CIK de SEC como identidad estable.

```json
{
  "FTFT": { "cik": "0001066923", "old_ticker": "SPU" },
  "MBOT": { "cik": "0000883975", "old_ticker": "STEMD" }
}
```

| Campo | Descripción |
|---|---|
| `cik` | CIK de SEC (string con ceros a la izquierda); `null` si no se resolvió |
| `old_ticker` | símbolo anterior; `null` o ausente si no hubo rename |
| `flat_file_only` | `true` = el símbolo aparece en los flat files pero no se resolvió su identidad |

Es un **cache de lookups**: registra también los resultados negativos para no
repetir la consulta. De ~15,300 entradas, ~1,150 tienen un rename utilizable; el
resto son consultas sin resultado. Una entrada puede ser `null` (se consultó, sin
respuesta). Al consumirlo, quedate con las que tienen `old_ticker` no vacío.

Caso de uso típico: el día 1 tras un rename, el cierre previo vive bajo el símbolo
viejo.

## Actualización

Regenerado todos los días alrededor de las 02:15 ET por un workflow automatizado.
`fresh_universe.csv` se reescribe entero; `ticker_changes.json` crece de forma
incremental. Los commits solo ocurren cuando algo cambió — la fecha del último
commit es la del último cambio real, no la de la última corrida.

`as_of_date` es la fuente de verdad de la frescura del snapshot. Si tu proceso
depende de tener el dato del día, validá esa columna en vez de confiar en el
mtime del archivo.

## Fuente

Derivado de la data de referencia de Polygon.io. Se publica como conveniencia
para consumo propio y sin garantías de exactitud, completitud ni disponibilidad.
Verificá contra la fuente antes de usarlo para cualquier decisión.
