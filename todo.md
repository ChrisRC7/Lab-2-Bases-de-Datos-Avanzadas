# TODO — Laboratorio 2: One size fits all?

**Entrega:** domingo 18 de octubre de 2026, 11:00 pm (un PDF por grupo en TEC Digital)
**Carpeta:** `labs/lab2-one-size/`
**Motores:** Postgres 16 + Valkey 8

---

## Tareas comunes (todo el equipo, antes de empezar)

- [ ] Leer el paper de Stonebraker et al. ("One Size Fits All") y el `README.md` del lab.
- [ ] Leer `common.py` (lector del CSV, conexiones, registro de medidas, cronómetro).
- [ ] Escribir la **predicción previa** de cada paso (¿quién gana y por cuánto?) **antes** de correr nada (vale en la rúbrica).
- [ ] Levantar el entorno: `make build` y `make lab2-up`.
- [ ] Descargar el dataset: `docker compose run --rm app python3 labs/lab2-one-size/descargar.py`
      (si falla, usar el CLI de Kaggle hacia `labs/lab2-one-size/data/`).
- [ ] Acordar en qué máquina se corren las mediciones finales y anotar máquina, SO, RAM y perfil
      (completo / reducido) en `evidence/README.md`.
- [ ] No cambiar firmas de funciones ni nombres de llaves de Valkey.

---

## Persona 1 — Paso 1: Carga y espacio (`cargar.py`)

### Código
- [ ] **TODO 1** — `medir_espacio_postgres(cur)`: `SELECT` con `pg_relation_size`, `pg_indexes_size`
      y `pg_total_relation_size` para `'trips'`; retornar la fila.
- [ ] **TODO 2** — `cargar_valkey(r, limit)`: guardar los viajes con `HSET` en un **pipeline**
      (lotes + `execute()`), llave `trip:<id>`.
- [ ] Correr `cargar.py` (o `--perfil reducido` si hay < 8 GB de RAM) y guardar la salida con las líneas `OK:`.

### Análisis para el informe
- [ ] Explicar qué tienen en común `COPY` (Postgres) y el pipeline (Valkey) y cómo resuelven el overhead de carga masiva.
- [ ] Explicar por qué el pipeline cambia el tiempo (viajes de ida y vuelta por la red).
- [ ] Responder qué pasa si el TODO 1 falla después del `COPY`.
- [ ] Comparar `bytes_por_fila` de cada motor y `ram_extrapolada_total` vs `disco_total`.
- [ ] Responder: ¿cuál motor podría guardar un año entero en una laptop?
- [ ] **Ensayo final:** redactar la apreciación **«Ningún motor gana todas las cargas de trabajo»**
      (en qué paso gana cada uno y por cuántas veces, limitantes de Valkey), con números de los 3 pasos.

---

## Persona 2 — Paso 2: Consulta de almacén (`analitica.py`) + evidencia

### Código
- [ ] **TODO 3** — `agregar_lote(r, keys, sums, counts)`: leer `payment_type` y `fare_amount` con
      `HMGET` en un pipeline y acumular en `sums` y `counts` (equivalente manual al `HashAggregate`).
- [ ] Correr `analitica.py` y guardar la salida con las líneas `OK:`.

### Análisis para el informe
- [ ] Leer `evidence/plan_olap_postgres.txt` de abajo hacia arriba:
  - [ ] `Parallel Seq Scan on trips`: workers, filas por loop, tiempo.
  - [ ] `Buffers: shared read=…` → convertir a MiB (× 8 KiB) y comparar con `disco_tabla` del paso 1.
  - [ ] `Partial HashAggregate` y `Finalize GroupAggregate` (agregación en dos fases).
  - [ ] Señalar el **nodo que más tiempo consume** y relacionarlo con el row-store (lee las 20 columnas para usar 2).
- [ ] Reportar la pregunta no anticipada (propina promedio por hora) y por qué el modelo relacional la responde fácil.
- [ ] Generar gráficas y tabla: `graficas.py` → `evidence/graficas.png` y `evidence/tabla.md`.
- [ ] **Ensayo final:** redactar la apreciación **«Que Valkey pierda el `GROUP BY` no refuta el argumento
      del data warehouse»** (no se prueba un column-store sino un key-value sin primitiva de agregación),
      usando el plan de Postgres y citando la sección del paper.

---

## Persona 3 — Paso 3: Escrituras y feed (`escrituras.py`, `lua/feed.lua`)

### Código
- [ ] **TODO 4** — `lua/feed.lua`: traducir el trigger `feed_trip_after()` de `sql/pg_feed.sql`
      a `HINCRBY` / `HINCRBYFLOAT` sobre `agg:n`, `agg:suma`, `alarma:anomalias`, `alarma:alertas`
      (ojo: `ARGV` son strings → `anom == '1'`).
- [ ] **TODO 5** — `enviar_en_lotes(conn, msgs, batch)`: agrupar `batch` mensajes por transacción con
      `conn.transaction()` y `executemany`.
- [ ] Correr `escrituras.py` (y `--solo-feed` mientras depura) y guardar la salida con las líneas `OK:`
      de las seis variantes.
- [ ] (Opcional) Correr `escrituras.py --umbral 100 --mensajes 50000` (Count100 exacto del paper).

### Análisis para el informe
- [ ] 3a: comparar la mediana de lectura y update por llave en cada motor; explicar el costo del `fsync` del WAL (la D de ACID).
- [ ] 3b: incluir la tabla con los mensajes/segundo de las seis variantes.
- [ ] Explicar por qué el lote (TODO 5) y el pipeline de Valkey cambian el tiempo.
- [ ] Explicar inbound (Valkey, Lua dentro del servidor) vs outbound (Postgres, guarda y dispara trigger).
- [ ] **Ensayo final:** redactar la apreciación **«La brecha no es solo el overhead de `fsync()`»**:
      separar con los dos pares (`valkey_aof_always` vs `postgres_commit_por_mensaje` y
      `valkey_por_mensaje` vs `postgres_commit_asincrono`) qué parte es durabilidad y qué parte es modelo.

---

## Cierre (todo el equipo)

- [ ] Correr todo en la máquina acordada, desde un `resultados.jsonl` limpio, y regenerar `graficas.py`.
- [ ] Unir las tres apreciaciones en la respuesta final (**máx. 2 páginas**), con al menos un número de
      cada paso y las secciones del paper citadas.
- [ ] Armar el PDF:
  1. Código de los TODO 1–5 + salidas `OK:`.
  2. `graficas.png` y `tabla.md` + máquina, SO, RAM y perfil.
  3. Plan de Postgres con el nodo dominante señalado y páginas leídas en MiB.
  4. Respuesta de máx. 2 páginas.
- [ ] Revisar contra la rúbrica y subir a TEC Digital antes del domingo 18/10, 11:00 pm.
- [ ] Limpiar: `make lab2-down` y el `DROP TABLE` del README.
