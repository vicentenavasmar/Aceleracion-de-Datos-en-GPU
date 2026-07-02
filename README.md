# Aceleración de Datos en GPU - Procesamiento Distribuido con MPI y C++

Este proyecto implementa una solución de **procesamiento paralelo y distribuido de datos** en **C++20**, orientada al tratamiento eficiente de grandes conjuntos de datos geoespaciales en formato CSV mediante **MPI (Message Passing Interface)**. El programa principal genera un ejecutable llamado `computeDistributed`, construido con **CMake**, y utiliza una biblioteca auxiliar llamada `libDomain` para encapsular parte de la lógica del dominio.

## 🚀 Características Principales

- **Procesamiento Distribuido con MPI**: Ejecución paralela repartida entre múltiples procesos MPI, con comunicación mediante `MPI_Scatterv`, `MPI_Allreduce`, `MPI_Reduce`, `MPI_Send/Recv` y `MPI_Gather`.
- **Clúster Multi-Nodo**: Soporte para despliegue en un clúster real de 5 nodos accesibles por SSH, cada uno asignado a un continente geográfico.
- **Proyecto Modular**: Separación entre el ejecutable principal `computeDistributed` y la biblioteca `libDomain`, lo que facilita el mantenimiento y la reutilización del código.
- **Compilación con CMake y MPI**: Compiladores `mpicc`/`mpicxx` configurados directamente en el `CMakeLists.txt`, con optimizaciones `-O3 -march=native`.
- **Compatibilidad con C++20**: El proyecto está preparado para compilarse con los compiladores MPI del sistema sobre GCC.
- **Entrada basada en CSV**: El programa trabaja con un archivo de datos externo (avistamientos de Pokémon geolocalizados) pasado como argumento en la ejecución.
- **Fórmula de Haversine**: Cálculo preciso de distancias geográficas entre coordenadas en kilómetros.
- **TSP Greedy**: Resolución aproximada del Problema del Viajante (TSP) entre los Pokémon más raros de cada continente.

## ⚙️ Requisitos

Antes de compilar el proyecto, instala las dependencias necesarias:

```bash
sudo apt install build-essential git cmake libglm-dev libopenmpi-dev openmpi-bin
```

Para desplegar en el clúster, configura el acceso SSH con la clave correspondiente y añade las entradas del fichero `config` a tu `~/.ssh/config`:

```
Host aac05
    HostName 10.100.139.247
    User vmuser
    IdentityFile ~/.ssh/AAC05.key
```

## 📂 Preparación de Datos

El proyecto espera un archivo CSV externo con datos de avistamientos geolocalizados (latitud, longitud, continente, ID de Pokémon). El dataset debe colocarse en una ruta accesible, por ejemplo:

```bash
~/Practica1_memoria_compartida/300k.csv
```

El script `deploy_cluster.sh` automatiza el despliegue del código y del dataset al nodo principal del clúster:

```bash
scp -i ~/.ssh/AAC05.key -r Practica1_memoria_compartida vmuser@10.100.139.247:~
scp -i ~/.ssh/AAC05.key Zips/300k.csv vmuser@10.100.139.247:~/Practica1_memoria_compartida/
```

## 🛠️ Compilación

### Build de desarrollo

```bash
cd Practica1_memoria_compartida
mkdir build && cd build
cmake ..
make -j8
```

### Build optimizada (producción)

El `CMakeLists.txt` del ejecutable ya incluye `-O3 -march=native` por defecto, por lo que cualquier build genera código optimizado. Para controlar el tipo:

```bash
mkdir release && cd release
cmake .. -DCMAKE_BUILD_TYPE=Release
make -j8
```

## ▶️ Ejecución

El ejecutable se lanza con `mpirun` indicando el número de procesos y la ruta al fichero CSV:

```bash
mpirun -np 5 computeDistributed/computeDistributed <ruta_csv>
```

Ejemplo real con el dataset del clúster:

```bash
mpirun -np 5 computeDistributed/computeDistributed ~/Practica1_memoria_compartida/300k.csv
```

Para buscar un Pokémon concreto por ID (Ejercicio 2), se puede pasar el argumento opcional `--id`:

```bash
mpirun -np 5 computeDistributed/computeDistributed 300k.csv --id 25
```

> ⚠️ El Ejercicio 3 requiere **exactamente 5 procesos MPI** (uno por continente). Con menos, el programa avisa y finaliza correctamente.

## 🧮 Lógica del Programa

El programa implementa tres ejercicios principales que se ejecutan secuencialmente:

### Ejercicio 2 — Conteo distribuido por ID
Cada proceso recibe una fracción proporcional del dataset mediante `MPI_Scatterv`. Cada uno construye un histograma local de frecuencias por ID de Pokémon. Los histogramas se reducen al proceso root con `MPI_Reduce`, y el conteo del ID objetivo se difunde con `MPI_Allreduce`.

### Ejercicio 3 — Pokémon más raro por continente + TSP Greedy
Los 5 procesos reciben cada uno los avistamientos de su continente asignado (America, Europe, Africa, Asia, Pacific). Cada proceso calcula el Pokémon con menor número de avistamientos en su continente. Los resultados se recogen en el root con `MPI_Gather`. Finalmente, el root resuelve un TSP greedy (vecino más cercano) entre las coordenadas de los 5 Pokémon raros usando la **distancia de Haversine**, imprimiendo la ruta óptima y la distancia total recorrida en km.

## 📦 Estructura del Proyecto

```
Aceleracion-de-Datos-en-GPU/
├── CSVLoader.h                          # Cabecera auxiliar para carga de CSV
├── config                               # Configuración SSH del clúster de nodos
├── deploy_cluster.sh                    # Script de despliegue al clúster
├── README.md                            # Documentación base
└── Practica1_memoria_compartida/
    ├── CMakeLists.txt                   # Configuración raíz de CMake (C++20)
    ├── computeDistributed/
    │   ├── CMakeLists.txt               # Build del ejecutable MPI
    │   └── main.cpp                     # Lógica principal distribuida (MPI)
    ├── computeShared/                   # Módulo de memoria compartida (en desarrollo)
    ├── libDomain/                       # Biblioteca auxiliar de dominio
    ├── build/                           # Carpeta de build de desarrollo
    └── release/                         # Carpeta de build optimizada
```

## 🧰 Tecnologías Utilizadas

| Tecnología | Rol en el proyecto |
|---|---|
| **C++20** | Lenguaje principal del proyecto |
| **MPI (OpenMPI)** | Comunicación entre procesos distribuidos |
| **CMake 3.16+** | Sistema de configuración y generación de builds |
| **GLM** | Tipos de vectores (`glm::dvec2`) y operaciones trigonométricas |
| **GCC/mpicxx** | Compiladores utilizados para construir el proyecto |
| **Haversine** | Fórmula para distancias geográficas precisas en km |

## 🌍 Distribución de Procesos MPI

Cada uno de los 5 procesos MPI se asocia a un continente geográfico:

| Rango MPI | Continente |
|---|---|
| 0 | America (proceso root) |
| 1 | Europe |
| 2 | Africa |
| 3 | Asia |
| 4 | Pacific |

## 🧪 Uso Previsto

Este proyecto está orientado a prácticas de **Arquitectura y Aceleración de Computadores (AAC)**, enfocadas en **computación de altas prestaciones** y **procesamiento paralelo distribuido**. El diseño sobre clúster real con nodos SSH y la necesidad de evaluar tiempos de ejecución por ejercicio apuntan a un enfoque centrado tanto en la corrección como en el rendimiento y la escalabilidad.

***
*Desarrollado como práctica de procesamiento distribuido de datos en C++ con MPI, CMake y despliegue sobre clúster multi-nodo.*
