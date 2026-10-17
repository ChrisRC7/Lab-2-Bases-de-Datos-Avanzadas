# Paso 3: escrituras y feed

Laptop Intel Core i7-6500U (2 núcleos, 4 hilos), 11 GiB de RAM, SSD SATA Patriot P210 · Ubuntu 24.04
nativo, kernel 7.0 · Docker 29.8.2 · perfil completo (10 906 858 filas en Postgres, 1 000 000 en
Valkey) · 16/10/2026.

Parámetros por defecto: 2 000 operaciones por llave (semilla 4601), 10 000 mensajes, lote de 1 000,
umbral 10. Salida completa con las líneas `OK:` de las seis variantes en `salida_escrituras.txt`.

*Nota: el paso 1 se midió en la laptop de Fabricio y este paso en la de Christopher. Si el equipo
acuerda una sola máquina para las medidas finales, hay que repetir este paso allí.*

## Código

**TODO 4 — `lua/feed.lua`**

```lua
redis.call('HINCRBY', 'agg:n', pt, 1)
redis.call('HINCRBYFLOAT', 'agg:suma', pt, fare)

if anom == '1' then
    local anomalias = redis.call('HINCRBY', 'alarma:anomalias', vendor, 1)
    if anomalias % umbral == 0 then
        redis.call('HINCRBY', 'alarma:alertas', vendor, 1)
        return 1
    end
end
return 0
```

Es el trigger `feed_trip_after()` línea por línea. El primer `INSERT … ON CONFLICT` sobre `agg_pago` se
convierte en `HINCRBY` + `HINCRBYFLOAT` (en Valkey un campo que no existe vale 0, así que no hace falta
el caso del `INSERT`). El `IF` de la anomalía se convierte en el `if anom == '1'`: el cliente ya calculó la
anomalía y la manda como string. `HINCRBY` devuelve el valor nuevo, que es el `alarma_vendor.anomalias + 1`
del trigger, y con él se decide la alerta.

**TODO 5 — `enviar_en_lotes()`**

```python
for i in range(0, len(msgs), batch):
    with conn.transaction(), conn.cursor() as cur:
        cur.executemany(INSERT, [m[:4] for m in msgs[i:i + batch]])
```

Cada tramo de 1 000 mensajes es una transacción: un solo commit y una sola espera del WAL en vez de
1 000. El trigger se sigue disparando por cada fila.

## 3a. Operación por llave

| Métrica (mediana de 2 000) | Postgres | Valkey | Gana |
| --- | ---: | ---: | --- |
| Lectura por llave | 0,155 ms | 0,227 ms | Postgres, 1,5× |
| Update por llave | 1,751 ms | 0,124 ms | Valkey, 14,1× |

**Lectura.** Aquí la predicción falló: esperábamos que ganara Valkey y ganó Postgres. En el servidor los
dos hacen muy poco (Postgres baja por el B-tree de la llave primaria, que ya está en el *buffer cache*;
Valkey busca la llave en su tabla hash), así que lo que se mide es sobre todo el cliente. `HGETALL`
devuelve 38 strings (19 campos y 19 valores) y redis-py los parsea en Python puro, porque la imagen no
trae `hiredis`. psycopg usa su implementación en C (`psycopg.pq.__impl__ == "binary"`) para leer la fila.

**Update.** Con autocommit cada `UPDATE` de Postgres es una transacción, y el commit no responde hasta que
el WAL llega a disco con `fsync`. El update tarda 1,6 ms más que la lectura, y casi todo eso es esperar
al disco: es el precio de la **D** de ACID. Valkey, con `--appendonly no`, cambia el valor en RAM y
responde; por eso su update (0,124 ms) es incluso más rápido que su lectura, porque devuelve un solo
número en vez de 38 strings.

## 3b. El feed

| Variante | Mensajes/s | ms por mensaje | ¿Espera al disco por mensaje? |
| --- | ---: | ---: | --- |
| `postgres_commit_por_mensaje` | 483 | 2,071 | Sí (WAL) |
| `postgres_commit_asincrono` | 6 074 | 0,165 | No |
| `postgres_lote` | 12 394 | 0,081 | Una vez por lote |
| `valkey_por_mensaje` | 6 904 | 0,145 | No |
| `valkey_aof_always` | 212 | 4,710 | Sí (AOF) |
| `valkey_pipeline` | 33 674 | 0,030 | No |

Las seis variantes dan 7 alertas y los mismos conteos por tipo de pago (`{'1': 6047, '2': 3892, '3': 47,
'4': 14}`), así que todas hacen el mismo trabajo lógico.

### Predicción contra lo medido

| Variante | Predicción | Medido |
| --- | ---: | ---: |
| `postgres_commit_por_mensaje` | ~400 | 483 |
| `postgres_commit_asincrono` | ~3 000 | 6 074 |
| `postgres_lote` | ~6 000 | 12 394 |
| `valkey_por_mensaje` | ~7 000 | 6 904 |
| `valkey_aof_always` | ~500 | 212 |
| `valkey_pipeline` | ~50 000 | 33 674 |

Postgres sin `fsync` resultó el doble de rápido de lo que estimamos: el trigger en PL/pgSQL y los dos
*upserts* cuestan menos de lo que pensábamos. La sorpresa es `valkey_aof_always`, que esperábamos un
poco por encima de Postgres y quedó 2,3 veces por debajo (se explica abajo).

### ¿Por qué el lote (TODO 5) y el pipeline cambian el tiempo?

Los dos atacan un costo fijo que se paga por cada envío y no por cada mensaje.

- **El lote de Postgres quita esperas al disco.** `postgres_commit_por_mensaje` hace 10 000 commits y
  cada uno espera un `fsync` del WAL. `postgres_lote` hace 10: el trigger corre igual para cada fila, pero
  el disco se espera una vez cada 1 000 mensajes. Pasa de 483 a 12 394 msg/s, 25,7 veces más. Además,
  `executemany` de psycopg 3 manda las filas en modo *pipeline* de libpq, sin esperar la respuesta de
  cada `INSERT`; por eso el lote supera también a `commit_asincrono` (2,0×), que ya no esperaba al disco
  pero sí hacía un viaje de red por mensaje.
- **El pipeline de Valkey quita viajes de red.** `valkey_por_mensaje` no toca el disco, así que lo único
  que lo frena es esperar la respuesta de cada `EVALSHA` antes de mandar el siguiente: 10 000 viajes de
  ida y vuelta. El pipeline manda 1 000 llamadas juntas y recibe 1 000 respuestas juntas: 10 viajes.
  Pasa de 6 904 a 33 674 msg/s, 4,9 veces más. Valkey hace exactamente el mismo trabajo; se ahorra la
  espera.

Es la misma idea del paso 1 (`COPY` y el pipeline de `HSET`): no pagar el costo fijo por cada fila.

### Inbound (Valkey) vs outbound (Postgres)

**Outbound (Postgres).** El mensaje primero se **guarda**: un `INSERT` en `feed_trip`, que crea una
versión de fila (MVCC), la escribe en el WAL y en el heap. Después el trigger `AFTER INSERT` actualiza
`agg_pago` y, si hay anomalía, `alarma_vendor`, con dos `INSERT … ON CONFLICT` que toman bloqueos de fila
y generan más versiones y más WAL. El resultado se calcula *después* de almacenar el dato.

**Inbound (Valkey).** El mensaje **no se guarda**: pasa por `feed.lua` dentro del servidor y solo quedan
los cuatro hashes de contadores. El script corre de forma atómica, en RAM, sin parsear SQL ni planificar,
sin MVCC y sin volver al cliente entre un `HINCRBY` y el siguiente. Es lo que el paper llama procesar el
mensaje «mientras pasa» en vez de «después de guardarlo» (sección 4.1), con la lógica de la aplicación
corriendo dentro del motor (sección 4.3).

El costo de lo inbound es que, al final, Postgres tiene los 10 000 mensajes en `feed_trip` y puede
responder cualquier pregunta nueva sobre ellos; Valkey solo tiene los contadores que se previeron.

### La brecha: durabilidad vs modelo

Si se resta el tiempo por mensaje de la variante sin disco al de la variante con disco, queda lo que
cuesta el `fsync` por mensaje en cada motor:

| | Con `fsync` por mensaje | Sin `fsync` | Costo del `fsync` |
| --- | ---: | ---: | ---: |
| Postgres | 2,071 ms (483 msg/s) | 0,165 ms (6 074 msg/s) | 1,906 ms (92 % del tiempo) |
| Valkey | 4,710 ms (212 msg/s) | 0,145 ms (6 904 msg/s) | 4,565 ms (97 % del tiempo) |

- **Par con durabilidad** (`valkey_aof_always` vs `postgres_commit_por_mensaje`): **gana Postgres, 2,3×**.
  Cuando los dos tienen que esperar al disco en cada mensaje, el modelo de Valkey no le sirve: el
  `fsync` se come el 97 % de su tiempo. El suyo además es más caro (4,6 ms contra 1,9 ms). Una razón
  medida: al activar `appendonly yes`, Valkey arranca una reescritura del AOF en segundo plano que vuelca
  el millón de viajes a disco mientras corre el feed, y compite por el mismo SSD. Con una prueba aparte
  (mismo script, mismos 10 000 mensajes) dio 184 msg/s durante la reescritura y 254 msg/s después, todavía
  debajo de Postgres. Otra causa probable, que no medimos: el WAL de Postgres usa segmentos de 16 MiB ya
  reservados, y el AOF es un archivo que crece, así que cada `fsync` también tiene que confirmar el tamaño
  nuevo del archivo. Postgres lleva décadas optimizando justo ese camino; Valkey no está hecho para eso.
- **Par sin durabilidad** (`valkey_por_mensaje` vs `postgres_commit_asincrono`): **gana Valkey, pero
  solo 1,14×** (6 904 contra 6 074). Sin disco, los dos quedan limitados por el viaje de red por mensaje
  y por el cliente de Python, que es igual para ambos. La diferencia de modelo (guardar la fila más el
  trigger contra cuatro `HINCRBY` en RAM) es de solo 0,02 ms por mensaje.
- **Sin red ni disco por mensaje** (`valkey_pipeline` vs `postgres_lote`): **Valkey, 2,7×** (33 674
  contra 12 394). Aquí se ve mejor el modelo: con los costos fijos repartidos entre 1 000 mensajes,
  Postgres gasta 0,081 ms por mensaje en guardarlo y disparar el trigger, y Valkey 0,030 ms en correr el
  script.

En esta laptop la mayor parte de la brecha entre la variante más lenta de Postgres y la más rápida de
Valkey (483 contra 33 674 msg/s, 70×) es durabilidad: solo quitar el `fsync` multiplica Postgres por
12,6. Pero no es toda: con el disco y la red repartidos, el modelo inbound sigue siendo 2,7 veces más
rápido. Y cuando a Valkey se le pide la misma durabilidad, pierde.

## Count100 exacto del paper (opcional)

*Pendiente: se completa con `salida_escrituras_count100.txt`.*

## Borrador del ensayo: «La brecha no es solo el overhead de `fsync()`»

En el paper, StreamBase procesó 160 000 mensajes por segundo y el RDBMS, 900, en la misma máquina
(sección 3.2). La explicación fácil es que el RDBMS escribe a disco y el motor de streams no. Nuestras
mediciones del paso 3b permiten separar esa explicación en dos partes, porque comparan los dos motores
en pares que esperan al disco igual.

La durabilidad pesa mucho. En Postgres, el `fsync` del WAL cuesta 1,9 ms por mensaje, el 92 % del
tiempo de `postgres_commit_por_mensaje` (483 msg/s). Solo con `synchronous_commit = off` sube a
6 074 msg/s, 12,6 veces más, sin tocar el modelo. Y la durabilidad no es una ventaja de Valkey: con
`appendfsync always`, Valkey baja a 212 msg/s y pierde contra Postgres por 2,3 veces. Cuando los dos
garantizan la **D** en cada mensaje, el que gana es el que tiene la bitácora mejor optimizada, no el
que procesa en memoria.

Pero el `fsync` no explica todo. Con los dos sin esperar al disco, Valkey sigue arriba: 1,14× por
mensaje, cuando la red y el cliente de Python son el cuello de botella, y 2,7× cuando esos costos se
reparten en lotes de 1 000 (`valkey_pipeline` 33 674 contra `postgres_lote` 12 394). Esa diferencia
es el modelo. Postgres procesa outbound: primero guarda el mensaje en `feed_trip` (versión de fila,
heap, WAL) y después un trigger hace dos `INSERT … ON CONFLICT`. Valkey procesa inbound: el mensaje
pasa por un script Lua dentro del servidor y solo quedan cuatro contadores. Es la diferencia que el
paper atribuye al procesamiento inbound (sección 4.1) y a correr la lógica de la aplicación dentro del
motor (sección 4.3).

El paper también dice cómo resuelve StreamBase la durabilidad sin `fsync`: con alta disponibilidad por
réplicas en vez de una bitácora en disco (sección 4.4). Eso es justo lo que nuestro Valkey no tiene; con
AOF, lo único que le queda es esperar al disco, y ahí pierde.

En resumen: en nuestra laptop la mayor parte de la brecha de 70× entre `postgres_commit_por_mensaje`
y `valkey_pipeline` viene de la durabilidad, pero una parte de 2,7× es del modelo y no desaparece al
quitar el `fsync`. Ninguna de las dos partes por sí sola explica el resultado del paper.

*Revisar los números de sección contra el PDF del paper antes de entregar.*
