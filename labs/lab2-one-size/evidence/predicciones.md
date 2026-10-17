# Predicciones del Laboratorio 2

Se escriben antes de correr cada paso.

## Paso 1: carga y espacio

Fabricio Mena · 15/10/2026, 18:34 · perfil completo (10 906 858 filas en Postgres, 1 000 000 en Valkey)

| Métrica | Postgres | Valkey | Gana |
| --- | ---: | ---: | --- |
| Velocidad de carga | ~150 000 filas/s | ~30 000 filas/s | Postgres, ~5× |
| Espacio del mes completo | ~1 850 MiB en disco | ~6 000 MiB en RAM | Postgres, ~3× |
| Bytes por fila | ~180 B | ~550 B | Postgres, ~3× |

- Postgres guarda cada columna con su tipo (un `smallint` ocupa 2 bytes, un `double` 8) y los nombres
  de las columnas una sola vez, en el esquema. Estimo unos 150 B por fila en la tabla y unos 20 B por
  fila en el índice de la llave primaria.
- Valkey guarda todo como texto y repite los 19 nombres de campo en cada viaje (`pickup_longitude`,
  `improvement_surcharge`, etc.). Solo los nombres ya suman unos 280 B por fila.
- `COPY` manda el CSV tal cual y el servidor lo procesa. En Valkey el cliente de Python tiene que armar
  cada `HSET` con los 19 pares campo-valor. El pipeline evita esperar la red en cada fila, pero ese
  trabajo en el cliente sigue ahí.
- Un año son unos 130 M de viajes. En Postgres serían unos 22 GiB de disco, que caben en una laptop. En
  Valkey serían unos 70 GiB de RAM, que no caben. Con el `--maxmemory 4gb` del curso, ni siquiera el mes
  completo cabría en Valkey.

## Paso 2: consulta de almacén

*Pendiente (Persona 2).*

## Paso 3: escrituras y feed

Christopher · 16/10/2026, 19:35 · Intel Core i7-6500U (4 hilos), 11 GiB de RAM, SSD SATA Patriot P210 ·
Ubuntu 24.04 nativo · 2 000 operaciones por llave, 10 000 mensajes, lote de 1 000, umbral 10

### 3a. Operación por llave (mediana)

| Métrica | Postgres | Valkey | Gana |
| --- | ---: | ---: | --- |
| Lectura por llave | ~0,25 ms | ~0,15 ms | Valkey, ~1,5× |
| Update por llave | ~3 ms | ~0,15 ms | Valkey, ~20× |

- La lectura por llave es casi empate: los dos resuelven la llave con una estructura en memoria (el
  B-tree de la PK ya está en el *buffer cache*; el hash de Valkey es O(1)) y lo que domina es el viaje
  de ida y vuelta por la red de Docker y el cliente de Python. Postgres paga un poco más por parsear el
  SQL, planificar y armar las 20 columnas.
- En el update la diferencia se dispara: con autocommit cada `UPDATE` espera el `fsync` del WAL, y este
  SSD SATA sin DRAM tarda milisegundos en cada `fsync`. Valkey no toca el disco.

### 3b. Feed (mensajes por segundo)

| Variante | Predicción |
| --- | ---: |
| `postgres_commit_por_mensaje` | ~400 msg/s |
| `postgres_commit_asincrono` | ~3 000 msg/s |
| `postgres_lote` | ~6 000 msg/s |
| `valkey_por_mensaje` | ~7 000 msg/s |
| `valkey_aof_always` | ~500 msg/s |
| `valkey_pipeline` | ~50 000 msg/s |

- Par con durabilidad (`valkey_aof_always` vs `postgres_commit_por_mensaje`): casi empate, los dos
  quedan limitados por un `fsync` por mensaje. Espero Valkey ~1,3× arriba, porque escribe menos bytes
  en la bitácora (un comando, no la fila del feed más dos filas de contadores en el WAL).
- Par sin durabilidad (`valkey_por_mensaje` vs `postgres_commit_asincrono`): Valkey ~2× arriba. Esa
  diferencia es el modelo: Postgres guarda la fila en `feed_trip`, dispara el trigger en PL/pgSQL y hace
  dos `INSERT … ON CONFLICT` con sus bloqueos y versiones de fila; Valkey corre cuatro `HINCRBY` en RAM.
- `valkey_pipeline` gana por mucho (~100× sobre `postgres_commit_por_mensaje`), porque además quita el
  viaje de red por mensaje. Es lo más cercano a la relación 160 000 vs 900 del paper.
- En esta laptop la brecha total será sobre todo `fsync` (el disco es lento), pero aun sin `fsync`
  Valkey debería quedar arriba: no toda la brecha es durabilidad.
