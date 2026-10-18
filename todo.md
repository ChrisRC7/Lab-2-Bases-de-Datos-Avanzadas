# TODO — Laboratorio 2: One size fits all?

**Entrega:** domingo 18 de octubre de 2026, 11:00 pm (un PDF por grupo en TEC Digital)
**Carpeta:** `labs/lab2-one-size/`
**Motores:** Postgres 16 + Valkey 8

---

## Tareas comunes (todo el equipo, antes de empezar)

- [ ] Leer el paper de Stonebraker et al. ("One Size Fits All") y el `README.md` del lab.
- [ ] Leer `common.py` (lector del CSV, conexiones, registro de medidas, cronómetro).
- [ ] Escribir la **predicción previa** de cada paso (¿quién gana y por cuánto?) **antes** de correr nada (vale en la rúbrica).
      Pasos 1 y 3 listos en `labs/lab2-one-size/evidence/predicciones.md`. La del paso 2 se escribió
      después de correr; así se declara en el informe.
- [x] Levantar el entorno: `make build` y `make lab2-up`.
- [x] Descargar el dataset: `docker compose run --rm app python3 labs/lab2-one-size/descargar.py`
      (si falla, usar el CLI de Kaggle hacia `labs/lab2-one-size/data/`).
- [x] Acordar en qué máquina se corren las mediciones finales y anotar máquina, SO, RAM y perfil
      (completo / reducido) en `evidence/README.md`.
      Corrida final en la laptop de Fabricio (Ryzen 7 5700U, WSL 2), anotada en
      `labs/lab2-one-size/evidence/README.md`.
- [ ] No cambiar firmas de funciones ni nombres de llaves de Valkey.

---

## Persona 1 (Fabricio) — Paso 1: Carga y espacio (`cargar.py`)

### Código
- [x] **TODO 1** — `medir_espacio_postgres(cur)`: `SELECT` con `pg_relation_size`, `pg_indexes_size`
      y `pg_total_relation_size` para `'trips'`; retornar la fila.
- [x] **TODO 2** — `cargar_valkey(r, limit)`: guardar los viajes con `HSET` en un **pipeline**
      (lotes + `execute()`), llave `trip:<id>`.
- [x] Correr `cargar.py` (o `--perfil reducido` si hay < 8 GB de RAM) y guardar la salida con las líneas `OK:`.

### Análisis para el informe
- [x] Explicar qué tienen en común `COPY` (Postgres) y el pipeline (Valkey) y cómo resuelven el overhead de carga masiva.
- [x] Explicar por qué el pipeline cambia el tiempo (viajes de ida y vuelta por la red).
- [x] Responder qué pasa si el TODO 1 falla después del `COPY`.
- [x] Comparar `bytes_por_fila` de cada motor y `ram_extrapolada_total` vs `disco_total`.
- [x] Responder: ¿cuál motor podría guardar un año entero en una laptop?
- [x] **Ensayo final:** redactar la apreciación **«Ningún motor gana todas las cargas de trabajo»**
      (en qué paso gana cada uno y por cuántas veces, limitantes de Valkey), con números de los 3 pasos.
      En `labs/lab2-one-size/evidence/paso1.md`, con los números de la corrida final.

---

## Persona 2 (Alisson) — Paso 2: Consulta de almacén (`analitica.py`) + evidencia

### Código
- [x] **TODO 3** — `agregar_lote(r, keys, sums, counts)`: leer `payment_type` y `fare_amount` con
      `HMGET` en un pipeline y acumular en `sums` y `counts` (equivalente manual al `HashAggregate`).
- [x] Correr `analitica.py` y guardar la salida con las líneas `OK:`.

### Análisis para el informe
- [x] Leer `evidence/plan_olap_postgres.txt` de abajo hacia arriba:
  - [x] `Parallel Seq Scan on trips`: workers, filas por loop, tiempo.
  - [x] `Buffers: shared read=…` → convertir a MiB (× 8 KiB) y comparar con `disco_tabla` del paso 1.
  - [x] `Partial HashAggregate` y `Finalize GroupAggregate` (agregación en dos fases).
  - [x] Señalar el **nodo que más tiempo consume** y relacionarlo con el row-store (lee las 20 columnas para usar 2).
- [x] Reportar la pregunta no anticipada (propina promedio por hora) y por qué el modelo relacional la responde fácil.
- [x] Generar gráficas y tabla: `graficas.py` → `evidence/graficas.png` y `evidence/tabla.md`.
- [x] **Ensayo final:** redactar la apreciación **«Que Valkey pierda el `GROUP BY` no refuta el argumento
      del data warehouse»** (no se prueba un column-store sino un key-value sin primitiva de agregación),
      usando el plan de Postgres y citando la sección del paper.

---

## Persona 3 (Christopher) — Paso 3: Escrituras y feed (`escrituras.py`, `lua/feed.lua`)

### Código
- [x] **TODO 4** — `lua/feed.lua`: traducir el trigger `feed_trip_after()` de `sql/pg_feed.sql`
      a `HINCRBY` / `HINCRBYFLOAT` sobre `agg:n`, `agg:suma`, `alarma:anomalias`, `alarma:alertas`
      (ojo: `ARGV` son strings → `anom == '1'`).
- [x] **TODO 5** — `enviar_en_lotes(conn, msgs, batch)`: agrupar `batch` mensajes por transacción con
      `conn.transaction()` y `executemany`.
- [x] Correr `escrituras.py` (y `--solo-feed` mientras depura) y guardar la salida con las líneas `OK:`
      de las seis variantes. Salida en `evidence/salida_escrituras.txt`.
- [x] (Opcional) Correr `escrituras.py --umbral 100 --mensajes 50000` (Count100 exacto del paper).
      Salida en `evidence/salida_escrituras_count100.txt`, comentada en `paso3.md`.

### Análisis para el informe
- [x] 3a: comparar la mediana de lectura y update por llave en cada motor; explicar el costo del `fsync` del WAL (la D de ACID).
- [x] 3b: incluir la tabla con los mensajes/segundo de las seis variantes.
- [x] Explicar por qué el lote (TODO 5) y el pipeline de Valkey cambian el tiempo.
- [x] Explicar inbound (Valkey, Lua dentro del servidor) vs outbound (Postgres, guarda y dispara trigger).
- [x] **Ensayo final:** redactar la apreciación **«La brecha no es solo el overhead de `fsync()`»**:
      separar con los dos pares (`valkey_aof_always` vs `postgres_commit_por_mensaje` y
      `valkey_por_mensaje` vs `postgres_commit_asincrono`) qué parte es durabilidad y qué parte es modelo.
      En `labs/lab2-one-size/evidence/paso3.md`, junto con todo el análisis del paso 3.

---

## Cierre (todo el equipo)

- [x] Correr todo en la máquina acordada, desde un `resultados.jsonl` limpio, y regenerar `graficas.py`.
- [x] Unir las tres apreciaciones en la respuesta final (**máx. 2 páginas**), con al menos un número de
      cada paso y las secciones del paper citadas.
- [x] Armar el PDF (`labs/lab2-one-size/REPORTE.tex` y `REPORTE.pdf`):
  1. Código de los TODO 1–5 + salidas `OK:`.
  2. `graficas.png` y `tabla.md` + máquina, SO, RAM y perfil.
  3. Plan de Postgres con el nodo dominante señalado y páginas leídas en MiB.
  4. Respuesta de máx. 2 páginas.
- [ ] Revisar contra la rúbrica y subir a TEC Digital antes del domingo 18/10, 11:00 pm.
- [ ] Limpiar: `make lab2-down` y el `DROP TABLE` del README.
