# Torneo de Komikan Concurrente — Tarea 2 SO

Implementación de un torneo de Komikan en C utilizando programación concurrente mediante hilos POSIX (`pthread`), mutex, variables de condición y semáforos.

## Integrantes
- Valentina Mella
- Paulette Saavedra

## 1. Requisitos

Para compilar y ejecutar el programa se necesita:
- Sistema operativo Unix/Linux.
- GCC.
- Make.
- Biblioteca POSIX Threads (`pthread`).
- Valgrind (opcional, para pruebas de memoria).

## 2. Compilación

Desde el directorio del proyecto, ejecutar:

```bash
make
```

La compilación utiliza:

```bash
gcc -Wall -Wextra -std=c17 -pthread torneo.c -o torneo
```

Para eliminar el ejecutable:

```bash
make clean
```

## 3. Ejecución

```bash
./torneo N M [config.txt]
```

Parámetros:
- `N`: cantidad total de jugadores. Debe ser un número par entre 2 y 2000.
- `M`: cantidad máxima de mesas simultáneas. Debe estar entre 1 y 2000.
- `config.txt`: archivo de configuración opcional.

Ejemplo:

```bash
./torneo 8 3 config.txt
```

En este caso, participan 8 jugadores, que forman 4 parejas, con un máximo de 3 partidas simultáneas.

## 4. Archivo de configuración

El archivo `config.txt` permite establecer los parámetros iniciales de las partidas.

Ejemplo:

```ini
min_ms=50
max_ms=300
seed=1234
puma=12
perros=0,1,2,3,4,5,6,7,8,9,10,11
```

Descripción de los parámetros:

| Parámetro | Descripción |
|---|---|
| `min_ms` | Tiempo mínimo de espera entre movimientos, en milisegundos. |
| `max_ms` | Tiempo máximo de espera entre movimientos, en milisegundos. |
| `seed` | Semilla utilizada para inicializar las decisiones aleatorias. |
| `puma` | Casilla inicial del Puma (0 a 27). |
| `perros` | Posiciones iniciales de los 12 perros. |

Las posiciones deben estar dentro del tablero y no pueden repetirse entre las piezas.

Si no se proporciona un archivo de configuración, se utilizan los valores predeterminados definidos en el programa.

## 5. Tablero y representación

El tablero contiene 28 casillas:
- Casillas 0 a 24: cuadrícula principal de 5 × 5.
- Casillas 25, 26 y 27: posiciones correspondientes al trono.

Representación en consola:
- `P`: Puma.
- `D`: Perro.
- `.`: casilla vacía.

## 6. Arquitectura concurrente

El programa utiliza tres niveles de hilos:

- **Clientes:** se crean N hilos, uno por jugador. Se emparejan de dos en dos, asignando el rol de Puma al primero y Perros al segundo.
- **Juez:** cada pareja crea un hilo juez, responsable de administrar la partida, controlar las condiciones de término y gestionar la mesa.
- **Piezas:** el juez crea 13 hilos por partida: uno para el Puma y doce para los perros.

La sincronización se implementa mediante:
- `pthread_mutex_t`: protege el estado compartido del tablero.
- `pthread_cond_t`: permite coordinar los hilos y notificar cambios.
- `sem_t`: limita la cantidad de mesas activas simultáneamente.

## 7. Pruebas

**Ejecución normal:**

```bash
./torneo 8 3
```

**Ejecución con configuración personalizada:**

```bash
./torneo 8 3 config.txt
```

**Simulación de abandono:**

```bash
FALLAR_JUGADOR=3 ./torneo 4 2
```

La variable de entorno permite simular el abandono de un jugador durante una partida. El oponente es declarado ganador por abandono.

**Interrupción con Ctrl+C:**

Presionar `Ctrl+C` durante la ejecución para solicitar la interrupción del torneo y el cierre de las partidas activas.

**Revisión de memoria con Valgrind:**

```bash
valgrind --leak-check=full --tool=memcheck ./torneo 4 2
```

## 8. Limitaciones conocidas

- La semilla aleatoria permite inicializar las secuencias de decisión, pero no garantiza resultados idénticos debido al orden de ejecución de los hilos.
- No existe un límite máximo de movimientos por partida, por lo que una partida podría prolongarse considerablemente.
- El abandono se simula mediante una variable de entorno, ya que los jugadores están representados por hilos y no por procesos independientes.
