# Torneo de Komikan Concurrente — Tarea 2 Sistemas Operativos

**Integrantes:** Nombre Apellido 1, Nombre Apellido 2
**Entrega:** 30/10/2026

Simulación de un torneo de Komikan en tiempo real. Los jugadores se
emparejan (Puma vs Perros), se les asigna una mesa y las 13 fichas de cada
mesa (1 Puma + 12 Perros) se mueven al mismo tiempo, sin turnos, usando hilos
POSIX y sincronización con mutex, variables de condición y semáforos.

---

## 1. Archivos

| Archivo | Descripción |
|---|---|
| `torneo.c` | Código fuente completo |
| `Makefile` | Compilación |
| `config.txt` | Archivo de configuración de ejemplo |
| `config_rapido.txt` | Configuración para partidas rápidas (`min_ms=1`, `max_ms=5`) |
| `informe.pdf` | Informe de diseño (máx. 2 páginas) |
| `README.md` | Este archivo |

## 2. Compilación

```
make
```

Esto ejecuta:

```
gcc -Wall -Wextra -std=c17 -pthread torneo.c -o torneo
```

Para limpiar: `make clean`.

Requisitos: sistema Unix/Linux, `gcc` y la librería pthread.

## 3. Uso

```
./torneo N M [config.txt]
```

| Parámetro | Significado | Rango |
|---|---|---|
| `N` | Número total de jugadores (clientes) | 1 a 2000 |
| `M` | Máximo de mesas jugando a la vez | 1 a 2000 |
| `config.txt` | Archivo de configuración (opcional) | — |

Ejemplos:

```
./torneo 8 3              # 8 jugadores, 3 mesas, config por defecto
./torneo 8 3 config.txt   # lo mismo, usando el archivo
./torneo 7 2 config.txt   # N impar: un jugador queda sin rival
```

Si `N` es impar, un jugador queda sin pareja (cuál depende del orden en que
los hilos llegan al emparejamiento) y el programa lo avisa
(`Jugador X sin rival`), pero el resto del torneo sigue normal.

## 4. Archivo de configuración

Formato `clave=valor`, una por línea. Las líneas cuyo primer carácter es `#`
se ignoran. Ejemplo (`config.txt`):

```
min_ms=50
max_ms=300
seed=1234
puma=12
perros=0,1,2,3,4,5,6,7,8,9,10,11
```

| Clave | Significado |
|---|---|
| `min_ms` | Retardo mínimo entre movimientos de una ficha (ms), debe ser ≥ 1 |
| `max_ms` | Retardo máximo (ms), debe ser ≥ `min_ms` |
| `seed` | Semilla para las decisiones aleatorias |
| `puma` | Casilla inicial del Puma (0 a 27) |
| `perros` | Casillas iniciales de los 12 perros, separadas por coma |

Reglas de validación: las 13 casillas deben estar entre 0 y 27 y no pueden
repetirse. Si el archivo es inválido, el programa muestra
`Configuracion incorrecta` y termina. Si no se entrega archivo, se usan los
valores del ejemplo de arriba. Las claves desconocidas generan un aviso.

Ejemplo de `config_rapido.txt`:

```
min_ms=1
max_ms=5
seed=1234
puma=12
perros=0,1,2,3,4,5,6,7,8,9,10,11
```

## 5. Tablero

Las casillas se numeran así:

- **0 a 24:** cuadrícula 5x5 (fila = n/5, columna = n%5).
- **25, 26, 27:** el trono (27 es la punta).

```
        27
     25    26
 0  1  2  3  4
 5  6  7  8  9
10 11 12 13 14
15 16 17 18 19
20 21 22 23 24
```

Conexiones del trono: 25 con 0, 1, 26 y 27; 26 con 3, 4, 25 y 27; 27 con 2.
En la grilla, los vecinos son solo los de arriba, abajo, izquierda y derecha.

En pantalla se dibuja con: `P` = Puma, `D` = perro, `.` = casilla vacía.

## 6. Reglas implementadas

- **Puma:** se mueve a casillas adyacentes libres o captura saltando en línea
  recta sobre un perro hacia una casilla libre (el perro saltado debe ser
  adyacente al Puma y la casilla de llegada adyacente al perro). El salto
  `7 → 2 → 27` (y su inverso) se trata como caso especial para atravesar la
  unión entre la grilla y el trono. Gana con **6 o más capturas**.
- **Perros:** se mueven a casillas adyacentes libres y no capturan. Ganan si el
  Puma queda **sin movimientos** (ni pasos ni capturas posibles).
- No hay turnos: cada ficha duerme un tiempo aleatorio entre `min_ms` y
  `max_ms` y luego intenta mover.
- **Roles:** en cada pareja, el primer jugador que queda esperando en el
  emparejamiento es el Puma y el segundo es el de Perros. Por lo tanto el rol
  depende del orden de llegada de los hilos, no del número de jugador.

## 7. Diseño (resumen)

- **N hilos cliente:** se emparejan con `emparejar_mutex` y `emparejar_cond`.
  El primero espera y el segundo crea la partida (reserva su memoria, inicializa
  el mutex y la variable de condición de la partida) y lanza el hilo Juez.
- **1 hilo Juez por partida:** primero espera una mesa libre con el semáforo
  `mesas_libres` (valor inicial M). Una vez asignada la mesa, inicializa el
  tablero, crea los 13 hilos pieza y vigila la partida. Al terminar hace `join`
  de las piezas, imprime el resultado y libera la mesa con `sem_post`. Así,
  nunca hay más de M partidas jugando a la vez.
- **13 hilos pieza por mesa:** duermen fuera de la sección crítica y luego
  mueven.
- **Exclusión mutua:** un mutex por partida (`p->mutex`) protege tablero,
  posiciones y contadores. La elección del destino, la validación de
  adyacencia/captura y el movimiento ocurren dentro del mismo lock, así que dos
  fichas nunca toman la misma casilla.
- `imprimir_mutex` evita que se mezclen los tableros de distintas mesas.
- Orden de locks fijo (`p->mutex` y luego `imprimir_mutex`) para evitar
  deadlock.
- **Esperas con sondeo:** las esperas usan `sem_timedwait` (cada 1 s, mesas) y
  `pthread_cond_timedwait` (cada 100 ms, partidas y emparejamiento) para poder
  revisar periódicamente la bandera `detener` (Ctrl+C) y la condición de
  abandono. No son esperas activas (el hilo duerme entre revisiones), pero
  tampoco son esperas puramente bloqueantes.

El detalle completo está en `informe.pdf`.

## 8. Resiliencia

### Abandono de un jugador

Los jugadores son hilos, así que una falla real no se puede provocar.
La falla se **simula** con la variable de entorno `FALLAR_JUGADOR`:

```
FALLAR_JUGADOR=3 ./torneo 4 2
```

Si el jugador 3 está en una mesa, el Juez lo detecta en su primer ciclo de
vigilancia, termina la partida (por lo que casi no llega a jugarse), declara
ganador al oponente **por abandono** (`gano X por abandono`) y libera la mesa.
Si la partida ya había terminado antes de la detección, se respeta el
resultado original.

Nota: el rol de cada jugador depende del orden de llegada al emparejamiento
(el primero en esperar es el Puma), por lo que quién gana por abandono puede
variar entre ejecuciones.

Los mutex y variables de condición de cada partida se destruyen en `main`
después de que todos los hilos (piezas, Juez y clientes) terminaron con
`join`, por lo que ningún hilo los está usando en ese momento y la
destrucción es segura.

### Ctrl+C (SIGINT)

Al presionar Ctrl+C:

1. Se activa la bandera `detener` y no entran jugadores nuevos.
2. Las partidas activas terminan de forma ordenada (las piezas salen de su ciclo
   y el Juez las espera con `join`).
3. `main` espera a todos los hilos, destruye mutex, variables de condición y
   semáforo, y libera toda la memoria.

No se usa memoria compartida ni archivos abiertos (solo el config, que se cierra
al leerlo), por lo que no quedan recursos huérfanos.

## 9. Pruebas sugeridas

```
# Ejecución normal
./torneo 4 2 config.txt

# Más jugadores que mesas (las parejas esperan una mesa libre)
./torneo 10 2 config.txt

# Partidas rápidas
./torneo 20 5 config_rapido.txt

# Abandono
FALLAR_JUGADOR=3 ./torneo 4 2

# Ctrl+C: ejecutar y presionar Ctrl+C a mitad del torneo
./torneo 20 3 config.txt

# Fugas de memoria
valgrind --leak-check=full ./torneo 4 2 config.txt

# Condiciones de carrera
gcc -Wall -Wextra -std=c17 -pthread -fsanitize=thread torneo.c -o torneo_tsan
./torneo_tsan 4 2 config.txt
```

## 10. Limitaciones conocidas

- La `seed` fija las decisiones aleatorias de cada ficha, pero el orden en que
  el sistema operativo ejecuta los hilos puede variar, así que **no se
  garantiza reproducir exactamente la misma partida**.
- No hay límite de movimientos: con movimientos aleatorios, una partida puede
  durar bastante.
- El abandono es simulado (jugadores = hilos, no procesos) y se detecta desde
  el inicio de la partida.
- Los mutex y semáforos POSIX no garantizan orden FIFO.
- Con N y M muy grandes se pueden crear miles de hilos a la vez (hasta unos
  26.000 hilos de piezas con N=M=2000). Si `pthread_create` falla, el torneo se
  detiene de forma ordenada.
- La validación del archivo de configuración es básica: los valores numéricos
  se leen con `atoi`, por lo que un valor no numérico se interpreta como 0. Los
  comentarios solo se ignoran si `#` es el primer carácter de la línea.
