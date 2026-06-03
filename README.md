\# Unidad 10: Memorias del Computador

\## Laboratorio: Midiendo el Efecto de la Caché



\*\*Estudiante:\*\* Bryan Gelvez  

\*\*Fecha:\*\* Junio 2026  

\*\*Sistema:\*\* Ubuntu 26.04 LTS (WSL2) — Linux 6.6.114.1-microsoft-standard-WSL2 x86\_64



\---



\## 1. Objetivo



Implementar y ejecutar un benchmark en C que mide la latencia de acceso a memoria para arrays de distintos tamaños, observando empíricamente el efecto de la jerarquía de caché (L1, L2, L3, RAM) sobre el rendimiento de los programas, e interpretar los resultados en función de los conceptos de localidad espacial, líneas de caché y cache miss.



\---



\## 2. Conceptos Teóricos



\### 2.1 Jerarquía de Memoria

Los procesadores modernos no acceden directamente a la RAM para cada operación. En cambio, utilizan una jerarquía de memorias caché que actúan como intermediarios entre el procesador y la memoria principal:



\- \*\*L1 (Level 1):\*\* La caché más rápida y pequeña, integrada directamente en el núcleo del procesador. Tiene latencias de 1-4 ciclos de reloj. Se divide en L1 de datos (L1d) y L1 de instrucciones (L1i).

\- \*\*L2 (Level 2):\*\* Mayor capacidad que L1 pero más lenta (latencia \~10 ciclos). Generalmente privada por núcleo.

\- \*\*L3 (Level 3):\*\* Compartida entre todos los núcleos del procesador. Mucho más grande pero con latencia de \~40 ciclos.

\- \*\*RAM:\*\* La memoria principal del sistema. Latencia de \~100-200 ciclos. Cuando un dato no se encuentra en ningún nivel de caché ocurre un cache miss y el procesador debe esperar a que llegue desde RAM.



\### 2.2 Localidad Espacial y Temporal

\- \*\*Localidad temporal:\*\* Si se accede a un dato, es probable que se vuelva a acceder pronto. Las cachés aprovechan esto guardando datos recientemente usados.

\- \*\*Localidad espacial:\*\* Si se accede a una dirección de memoria, es probable que se acceda a direcciones cercanas pronto. Por esto, las cachés cargan bloques enteros llamados \*\*líneas de caché\*\* (cache lines) de 64 bytes típicamente.



\### 2.3 Cache Miss

Un \*\*cache miss\*\* ocurre cuando el dato que necesita el procesador no está en la caché y debe buscarse en un nivel inferior de la jerarquía. Existen tres tipos:

\- \*\*Compulsory miss:\*\* El primer acceso a un dato siempre es un miss.

\- \*\*Capacity miss:\*\* El array es más grande que la caché, no caben todos los datos.

\- \*\*Conflict miss:\*\* Dos datos compiten por el mismo conjunto de la caché.



\### 2.4 TLB Miss

El \*\*TLB (Translation Lookaside Buffer)\*\* es una caché especial que traduce direcciones virtuales a físicas. Cuando un programa accede a muchas páginas de memoria diferentes (como en acceso aleatorio a arrays grandes), el TLB se llena y ocurren \*\*TLB misses\*\*, añadiendo una penalización adicional de latencia sobre los cache misses.



\---



\## 3. Configuración del Entorno



\### 3.1 Sistema utilizado

\- \*\*OS:\*\* Ubuntu 26.04 LTS sobre WSL2 (Windows Subsystem for Linux 2)

\- \*\*Kernel:\*\* Linux 6.6.114.1-microsoft-standard-WSL2

\- \*\*Compilador:\*\* GCC (gcc -O0, sin optimizaciones)

\- \*\*Herramienta de tiempo:\*\* clock\_gettime(CLOCK\_MONOTONIC)



\### 3.2 Topología de Caché del Procesador



Se consultó la topología de caché mediante el sistema de archivos /sys:



```bash

for i in 0 1 2 3; do

&#x20; echo -n "Cache level $i: "

&#x20; cat /sys/devices/system/cpu/cpu0/cache/index${i}/size

done

```



\*\*Resultados:\*\*



| Índice | Nivel | Tamaño | Descripción |

|--------|-------|--------|-------------|

| index0 | L1d   | 48 KB  | Caché de datos de nivel 1 |

| index1 | L1i   | 32 KB  | Caché de instrucciones de nivel 1 |

| index2 | L2    | 2048 KB (2 MB) | Caché de nivel 2 |

| index3 | L3    | 266240 KB (\~260 MB) | Caché de nivel 3 (servidor) |



\---



\## 4. Implementación del Benchmark



\### 4.1 bench\_seq() — Acceso Secuencial

Recorre el array de forma secuencial (índice 0, 1, 2, ..., N). Este patrón aprovecha al máximo la localidad espacial: cada vez que se carga una línea de caché de 64 bytes, se usan todos los bytes consecutivos antes de necesitar una nueva línea. Esto minimiza los cache misses.



\### 4.2 bench\_rand() — Acceso Aleatorio

Recorre el array en orden aleatorio usando el algoritmo Fisher-Yates shuffle para mezclar los índices. Este patrón destruye la localidad espacial: cada acceso probablemente apunta a una línea de caché diferente, causando un cache miss en cada acceso cuando el array supera el tamaño de la caché.



\### 4.3 Compilación

Se compila con -O0 (sin optimizaciones) para evitar que el compilador elimine los accesos a memoria:



&#x20;   gcc -O0 -o cache\_bench cache\_bench.c



\---



\## 5. Resultados



\### 5.1 Tabla de Resultados



| Array (KB) | Seq ns/byte | Rand ns/elem | Ratio Rand/Seq |

|------------|-------------|--------------|----------------|

| 4          | 1.082       | 1.372        | 1.27x          |

| 8          | 1.544       | 1.909        | 1.24x          |

| 16         | 1.510       | 2.041        | 1.35x          |

| 32         | 1.146       | 1.948        | 1.70x          |

| 64         | 1.209       | 1.475        | 1.22x          |

| 128        | 1.158       | 1.532        | 1.32x          |

| 256        | 1.029       | 1.509        | 1.47x          |

| 512        | 0.976       | 2.310        | 2.37x  (salto L2) |

| 1024       | 0.967       | 2.705        | 2.80x  (salto L3) |

| 4096       | 1.072       | 3.199        | 2.98x  (salto RAM) |



\### 5.2 Análisis de los Puntos de Inflexión



\*\*Transición L1 → L2 (alrededor de 48 KB)\*\*

Cuando el array supera los 48 KB de L1d, los datos ya no caben en la caché más rápida. En el acceso aleatorio se observa un incremento de latencia desde 1.37 ns (4 KB) hasta 2.04 ns (16 KB). El acceso secuencial se mantiene estable gracias a las líneas de caché que traen datos contiguos.



\*\*Transición L2 → L3 (alrededor de 2 MB)\*\*

Al superar los 2 MB de L2, el acceso aleatorio muestra un salto notable: de 1.509 ns (256 KB) a 2.310 ns (512 KB), un incremento del 53%. El procesador ya no puede servir los datos desde L2 y debe acceder al L3.



\*\*Transición L3 → RAM (mayor a 4 MB)\*\*

A partir de 1 MB el acceso aleatorio sube hasta 3.199 ns en 4 MB. La penalización adicional se atribuye también a TLB misses: con arrays grandes y acceso aleatorio, se tocan muchas páginas de memoria distintas agotando las entradas del TLB.



\---



\## 6. Conclusiones



1\. El acceso secuencial es consistentemente más rápido que el aleatorio en todos los tamaños, gracias a la localidad espacial y al prefetching de líneas de caché de 64 bytes.



2\. La diferencia aumenta con el tamaño del array: para arrays pequeños (menor a L1) el ratio Rand/Seq es \~1.2x. Para arrays grandes (mayor a L2) el ratio supera 2.5x-3x, evidenciando el alto costo de los cache misses.



3\. Los cache misses son el cuello de botella principal en programas que acceden a memoria de forma irregular.



4\. Los TLB misses agravan el problema para arrays muy grandes con acceso aleatorio, añadiendo latencia extra por la traducción de direcciones virtuales a físicas.



5\. Estructuras de datos contiguas en memoria (arrays) son preferibles a estructuras enlazadas (listas enlazadas, árboles) cuando el rendimiento es crítico, ya que aprovechan mejor la jerarquía de caché.



\---



\## 7. Archivos del Repositorio



| Archivo | Descripción |

|---------|-------------|

| cache\_bench.c | Código fuente del benchmark en C |

| README.md | Este informe con análisis y resultados |



\---

