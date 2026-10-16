# Paso 1: carga y espacio

Laptop AMD Ryzen 7 5700U, 16 GB de RAM (9,7 GiB y 4 CPU para WSL 2) · Windows 11 Home con Ubuntu
24.04 en WSL 2 · Docker Desktop 29.5.2 · perfil completo · 15/10/2026.

Salida completa con las líneas `OK:` en `salida_cargar.txt`.

## Resultados

| Métrica | Postgres | Valkey |
| --- | ---: | ---: |
| Filas cargadas | 10 906 858 | 1 000 000 |
| Tiempo de carga | 71,9 s (`COPY`) + 13,3 s (índice PK) | 54,0 s |
| Filas por segundo | 151 596 | 18 528 |
| Espacio medido | 1 653,6 MiB tabla + 233,7 MiB índice = 1 887,7 MiB en disco | 523,9 MiB en RAM |
| Bytes por fila | 181,5 | 549,3 |
| Mes completo | 1 887,7 MiB en disco | 5 714,1 MiB en RAM (extrapolado) |

## Predicción contra lo medido

| Métrica | Predicción | Medido |
| --- | ---: | ---: |
| Postgres, filas/s | ~150 000 | 151 596 |
| Valkey, filas/s | ~30 000 | 18 528 |
| Postgres, bytes por fila | ~180 | 181,5 |
| Valkey, bytes por fila | ~550 | 549,3 |
| Valkey, RAM del mes | ~6 000 MiB | 5 714 MiB |

El espacio salió como se esperaba. La carga en Valkey fue más lenta de lo previsto, así que Postgres
ganó por 8,2× y no por 5×: armar cada `HSET` en Python cuesta más de lo que se estimó.

## Preguntas

**¿Qué tienen en común los dos métodos de carga? ¿Cómo se resuelve el overhead de cargar muchos datos?**

Los dos evitan mandar un comando por fila y esperar la respuesta de cada uno. `COPY` manda el CSV como
un solo flujo de bytes, en una sola transacción, y el servidor lo parsea; no hay un `INSERT` ni un
commit por fila. El pipeline junta 10 000 `HSET` en el cliente y los manda juntos con `execute()`. En
los dos casos el costo fijo de cada envío se paga una vez por bloque y no una vez por fila.

**¿Por qué el pipeline cambia el tiempo?**

Sin pipeline, cada `r.hset()` manda el comando y espera la respuesta antes de mandar el siguiente: serían
1 000 000 de viajes de ida y vuelta por la red. Con lotes de 10 000 son 100 viajes. Valkey hace el mismo
trabajo; lo que se ahorra es el tiempo esperando la red.

Aun con pipeline, Valkey cargó a 18 528 filas/s, 8,2 veces menos que `COPY` (151 596 filas/s). Con
`COPY`, Python solo pasa líneas de texto y el servidor hace el resto. Con Valkey, Python tiene que armar
cada `HSET` con 19 pares campo-valor.

**¿Qué pasa si el TODO 1 falla después del `COPY`?**

La conexión usa autocommit, así que cuando se llama al TODO 1 el `COPY`, la llave primaria y el `VACUUM`
ya están confirmados. La tabla `trips` queda cargada, pero el script se detiene: no se registran las
medidas de disco y la carga de Valkey no se ejecuta. Al corregir y volver a correr, `pg_trips.sql` hace
`DROP TABLE` y la carga se repite completa, unos 85 s en esta máquina.

**Compare `bytes_por_fila` y `ram_extrapolada_total` contra `disco_total`.**

Valkey usa 549,3 B por viaje y Postgres 181,5 B (159 de la tabla y 22 del índice): 3,0 veces menos.
Para el mes completo serían 5 714 MiB de RAM en Valkey contra 1 888 MiB de disco en Postgres.

Postgres guarda cada valor con su tipo (un `smallint` ocupa 2 bytes) y los nombres de las columnas una
sola vez, en el esquema. Valkey guarda todo como texto y repite los 19 nombres de campo en cada viaje.
Además, 5 714 MiB no caben en el Valkey del curso (`--maxmemory 4gb`), así que el mes completo no se
podría cargar.

**¿Cuál motor podría guardar un año entero en su laptop?**

Postgres. Un año son unos 131 M de viajes (12 meses como este). En Postgres serían unos 22 GiB de disco;
en Valkey, unos 67 GiB de RAM. Esta laptop tiene 16 GB de RAM y casi 950 GB de disco libre.

## Borrador del ensayo: «Ningún motor gana todas las cargas de trabajo»

*Los valores entre corchetes se completan cuando estén los resultados de los pasos 2 y 3.*

En el Paso 1 gana Postgres. Cargó el mes completo a 151 596 filas/s, 8,2 veces más rápido que Valkey
(18 528 filas/s), y cada viaje ocupa 181 B en disco contra 549 B en RAM, 3,0 veces menos. En el Paso 2
también gana Postgres: el `GROUP BY` sobre el mismo millón de filas tarda [P2] ms contra [P2] ms en
Valkey, [P2]× menos, porque Valkey no tiene esa primitiva y el cliente tiene que traer cada fila para
agruparla (sección 4.2 del paper).

En el Paso 3 el resultado se invierte. Valkey gana la lectura y el update por llave ([P3a] ms contra
[P3a] ms) y procesa el feed a [P3b] mensajes/s contra [P3b] de Postgres, [P3b]× más. Es el mismo patrón
que muestra el paper con StreamBase y el RDBMS, 160 000 contra 900 mensajes por segundo (sección 4): el
motor que procesa el mensaje sin guardarlo primero gana ese tipo de carga (sección 4.1).

Los números también muestran los límites de Valkey. Todo tiene que caber en RAM, y el mes completo
(5 714 MiB) ya no cabe en los 4 GB que tiene asignados. No tiene tipos ni esquema, por eso ocupa 3 veces
más. No tiene `GROUP BY`. Y sin AOF no escribe nada a disco: con `appendfsync always` baja a [P3b]
mensajes/s.

Cada motor gana donde su diseño calza con la carga, y ninguno gana todo. Es lo que el paper plantea en
las secciones 1 y 7: un solo motor no puede ser el mejor para cargas tan distintas.
