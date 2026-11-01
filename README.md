# Tarea 2 SO — Torneo de Komikan Concurrente

**Integrantes:** Valentina Mella y Paulette Saavedra

Implementación de un torneo de Komikan en C utilizando hilos POSIX (`pthread`), mutex, variables de condición y semáforos para manejar la ejecución concurrente de las partidas.

## 1. Requisitos

Para ejecutar el programa se necesita:
- Sistema operativo Linux/Unix.
- GCC.
- Make.
- Biblioteca POSIX Threads (`pthread`).
- Valgrind (opcional, para revisar memoria).

## 2. Compilación

Desde la carpeta del proyecto, ejecutar:

```bash
make
```

Se utiliza el siguiente comando de compilación:

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
- `N`: cantidad de jugadores. Debe ser un número par entre 2 y 2000.
- `M`: cantidad máxima de mesas simultáneas. Debe estar entre 1 y 2000.
- `config.txt`: archivo de configuración opcional.

Ejemplo:

```bash
./torneo 8 3 config.txt
```

En este caso participan 8 jugadores, que forman 4 parejas, con un máximo de 3 partidas simultáneas.

## 4. Archivo de configuración

El archivo `config.txt` permite definir los valores iniciales de las partidas.

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
| `seed` | Semilla utilizada para las decisiones aleatorias. |
| `puma` | Posición inicial del Puma (0 a 27). |
| `perros` | Posiciones iniciales de los 12 perros. |

Las posiciones deben estar dentro del tablero y no pueden repetirse entre las piezas.

Si no se proporciona un archivo de configuración, se utilizan los valores predeterminados definidos en el programa.

## 5. Tablero

El tablero contiene 28 casillas:
- Casillas 0 a 24: cuadrícula principal de 5 × 5.
- Casillas 25, 26 y 27: posiciones correspondientes al trono.

Representación en consola:
- `P`: Puma.
- `D`: Perro.
- `.`: casilla vacía.

## 6. Estructura del programa

El programa utiliza tres tipos de hilos:

- **Clientes:** se crean N hilos, uno por jugador. Se emparejan de dos en dos, asignando el rol de Puma al primero y el de Perros al segundo.
- **Juez:** cada pareja cuenta con un hilo Juez, encargado de administrar la partida, controlar su finalización y gestionar la mesa disponible.
- **Piezas:** el Juez crea 13 hilos por partida: uno para el Puma y doce para los perros.

Para coordinar los hilos se utilizan:
- `pthread_mutex_t`: protege el tablero y los datos compartidos de cada partida.
- `pthread_cond_t`: permite coordinar el emparejamiento y avisar sobre cambios en las partidas.
- `sem_t`: controla la cantidad máxima de mesas simultáneas.

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

La variable de entorno permite simular el abandono de un jugador. El Juez detecta la situación y declara ganador al oponente por abandono.

**Interrupción con Ctrl+C:**

Presionar `Ctrl+C` durante la ejecución para solicitar la interrupción del torneo.

**Revisión de memoria con Valgrind:**

```bash
valgrind --leak-check=full --tool=memcheck ./torneo 4 2
```

## 8. Limitaciones

- La semilla permite inicializar las decisiones aleatorias, pero no garantiza resultados idénticos debido al orden de ejecución de los hilos.
- No existe un límite máximo de movimientos por partida, por lo que algunas partidas podrían prolongarse considerablemente.
- El abandono se simula mediante una variable de entorno, ya que los jugadores están representados por hilos y no por procesos independientes.
