# Tarea2_SO
# Torneo de Komikan Concurrente — Tarea 2 SO

Integrantes: Nombre 1 (rol), Nombre 2 (rol)

## Compilación
```
make
```
Usa `gcc -Wall -Wextra -std=c17 -pthread`. Para limpiar: `make clean`.

## Uso
```
./torneo N M [config.txt]
```
- `N`: cantidad de jugadores (par, entre 2 y 2000)
- `M`: máximo de mesas simultáneas (entre 1 y 2000)
- `config.txt`: opcional

Ejemplo: `./torneo 8 3 config.txt`

## Archivo de configuración
```
min_ms=50
max_ms=300
seed=1234
puma=12
perros=0,1,2,3,4,5,6,7,8,9,10,11
```
- `min_ms` / `max_ms`: retardo aleatorio entre movimientos
- `seed`: semilla para las decisiones aleatorias
- `puma`: casilla inicial del Puma (0 a 27)
- `perros`: casillas iniciales de los 12 perros (distintas entre sí y del Puma)

Si falta el archivo, se usan esos mismos valores por defecto.

## Tablero
Casillas 0–24 son la cuadrícula 5x5 (fila = n/5, columna = n%5).
Casillas 25, 26 y 27 son el trono (27 es la punta).
Se imprime con `P` = Puma, `D` = perro, `.` = vacía.

## Estructura
- Un hilo por cliente (N hilos).
- El segundo jugador de cada pareja crea el hilo Juez de la mesa.
- El Juez espera una mesa libre (semáforo con valor M), crea 13 hilos pieza (1 Puma y 12 Perros) y vigila la partida.
- Un mutex por partida protege el tablero.

## Pruebas especiales
- **Abandono:** `FALLAR_JUGADOR=3 ./torneo 4 2` simula que el jugador 3 se cae. Su oponente gana por abandono. (Los jugadores son hilos, así que la falla se simula con esta variable.)
- **Ctrl+C:** detiene nuevos jugadores, termina las partidas activas y libera todo.
- **Fugas:** `valgrind --leak-check=full --tool=memcheck ./torneo 4 2`

## Limitaciones
- La seed no garantiza resultados idénticos, porque el orden de los hilos depende del SO.
- No hay límite de movimientos, así que una partida podría alargarse mucho.
