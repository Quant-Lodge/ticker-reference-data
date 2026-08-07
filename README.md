# ticker-reference-data

Reference data de tickers US, regenerada automáticamente cada día.

Datasets construidos a partir de avisos públicos de los exchanges y de registros
regulatorios, más correcciones propias (splits que las fuentes perdieron, resolución
de renames por CIK, reconciliación entre fuentes de halts):

| Archivo | Qué es | Tamaño |
|---|---|---|
| [`data/fresh_universe.csv`](data/fresh_universe.csv) | tickers "frescos": IPO o reverse split en los últimos 365 días | ~95 KB |
| [`data/ticker_changes.json`](data/ticker_changes.json) | renames de ticker resueltos por CIK | ~1.1 MB |
| [`history/halts/`](history/halts) | trading halts, un CSV por año, 2019→hoy | ~9.8 MB |

Los de `data/` son un **snapshot** que se reescribe cada día; los de `history/` son una
**serie acumulativa** que sólo crece.

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

Los halts tienen su propio espacio de tags, `halts-YYYY-MM-DD`, porque los dos
pipelines corren a horas distintas y cada dataset deriva su tag de su propio dato:

```bash
curl -sSfL https://github.com/Quant-Lodge/ticker-reference-data/releases/download/halts-2026-08-07/halts_2026.csv -o halts_2026.csv
```

Ese release **no** se marca como `latest`: ese puntero es de `fresh_universe.csv`, que
es lo que consumen los bots. Para halts usá el tag con fecha.

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

## `history/halts/`

Trading halts de US equities, un CSV por año. **69,256 eventos**, 2019→hoy.

Un halt no lo publica el exchange: se disemina por el SIP como mensaje administrativo,
y cada venue recibe los de todas las tapes. Por eso NYSE, Nasdaq y Cboe traen las tres
el universo completo, y acá van **reconciliadas**: un evento es una fila, y la columna
`fuentes` dice cuáles lo vieron.

```csv
symbol,symbol_raw,halt_date,halt_time,halt_ms,reason_code,reason_raw,name,listing_market,resume_date,resume_quote_time,resume_trade_time,duracion_seg,pause_threshold_px,px_antes,px_resume,gap_pct,fuentes,ingested_at
BKSY.WS,BKSY WS,2026-08-07,16:05:26.515,515,H11,Regulatory Concern,BlackSky…,NYSE,,,,,,,,,cboe|nasdaq_rss|nyse,2026-08-07T22:29:11
```

| Columna | Descripción |
|---|---|
| `symbol` | símbolo canónico (clase separada con punto: `BKSY.WS`) |
| `symbol_raw` | como lo mandó la fuente que ganó, para auditar |
| `halt_date` / `halt_time` | inicio del halt, hora de **Nueva York**; los ms sólo si Nasdaq lo vio |
| `reason_code` | canónico: `LUDP` (LULD), `T1`/`T2` (noticias), `T12`, `H10` (suspensión SEC), `UNKNOWN` |
| `resume_quote_time` | inicio del período quote-only — **es donde se forma el precio de reapertura** |
| `resume_trade_time` | reanudación efectiva; **vacío = nunca reanudó**, no se imputa el cierre |
| `px_antes` / `px_resume` / `gap_pct` | precio antes y después, y cuánto se movió mientras no se podía operar |
| `fuentes` | qué feeds vieron el evento, separados por `|` |

| Archivo | Eventos |
|---|---|
| `2019.csv` | 1,212 |
| `2020.csv` | 14,895 |
| `2021.csv` | 5,595 |
| `2022.csv` | 6,834 |
| `2023.csv` | 9,175 |
| `2024.csv` | 10,529 |
| `2025.csv` | 12,370 |
| `2026.csv` | 8,646 |

**Un papel puede tener muchos halts el mismo día** — el máximo medido en la serie es 60.
La identidad de un evento es `(symbol, halt_date, halt_time)` al segundo; deduplicar por
`(symbol, date)` borraría 59 de esos 60, justo los días parabólicos.

**Un año viejo puede cambiar cualquier día.** Los feeds devuelven los halts que siguen
ABIERTOS, y algunos llevan años (SVA lleva halteada desde 2023-06-13): cada fila se rutea
al archivo de *su* año, no al de la fecha de corrida.

Piso duro **2019-02-22**: es hasta donde llega el histórico de NYSE, la única fuente con
backfill por rango. Y ojo con cruzar mayo de 2012 en cualquier estadística: antes no
existía el LULD, así que serían regímenes distintos.

`name` es el nombre **actual**, no el de la fecha del halt. Para papeles renombrados el
símbolo tampoco es necesariamente el que se operaba ese día.

## Actualización

`data/` se regenera todos los días alrededor de las 02:15 ET.
`fresh_universe.csv` se reescribe entero; `ticker_changes.json` crece de forma
incremental.

`history/halts/` se actualiza de lunes a viernes a las **22:00 ET**, cuando el día ya
cerró y las tres fuentes convergieron (los halts por noticias se publican hasta entrada
la tarde y muchos reanudan al día siguiente).

Los commits solo ocurren cuando algo cambió — la fecha del último commit es la del
último cambio real, no la de la última corrida.

`as_of_date` es la fuente de verdad de la frescura del snapshot. Si tu proceso
depende de tener el dato del día, validá esa columna en vez de confiar en el
mtime del archivo.

## Fuente

Construido a partir de avisos públicos de los exchanges (NYSE, Nasdaq, Cboe) y de
registros regulatorios, reconciliados entre sí. Se publica como conveniencia para
consumo propio y sin garantías de exactitud, completitud ni disponibilidad.
Verificá contra la fuente primaria antes de usarlo para cualquier decisión.
