# PRACTICA 4

Página de Enunciado: `https://jderobot.github.io/RoboticsAcademy/exercises/AutonomousCars/global_navigation/`

Vamos a explicar y entender el funcionamiento de GlobalNavigation. En esta práctica se implementa un sistema completo de planificación global y navegación local para un coche autónomo.


El objetivo es que el robot sea capaz de:

1. Recibir un destino seleccionado por el usuario.
2. Calcular en la malla de ocupación (grid) el camino óptimo hasta el destino.
3. Evitar obstáculos (bordes y esquinas) usando penalización de seguridad.
4. Generar un sub-destino local para navegar suavemente evitando mínimos locales.
5. Calcular velocidades lineales y angulares para dirigir el robot hasta la meta.


## Path Planning 

Lo primero es entender que el mapa está divido en una cuadricula donde `1` representa carretera navegable y `0` representa obstáculos. Nuestro objetivo será construir un campo (mapa de costes) donde el destino tendrá coste 0, cada celda trendrá un coste acumulado que indica cuanto cuesta atravesarla y las celdas proximas a los obstáculos tendrán una penalización mayor. Como resultado el robot seguirá el gradiente descendente hacia el destino.

Para conseguir esto hemos definido una serie de funciones que vamos a analizar para entender su funcionamiento: 

`def obstacle_penalty()`

Esta función se encarga de elevar el coste de las celdas que están cerca de los obstáculos. De esta forma evitamos que el robot circule pegado a los muros. Veamos más detenidamente su funcionamiento:

  1. Calcula, para cada celda, a cuántas celdas está del obstáculo más cercano usando `distance_transform_edt`
  2. Si una celda está a una distancia menor o igual que el radio indicado, se le asigna un coste extra
  3. Finalmente devuelve un mapa con la penalización: `penalty_map`

De esta forma conseguimos alejar al robot de los bordes, evitar atascos en giros cerrados y crear un mapa de navegación mucho más seguro.


`def wavefront()`

Esta función implementa un algoritmo de expansión de coste desde el destino. Con ella estamos creando un campo donde el destino tiene coste 0, las celdas cercanas al destino tienen coste bajo, cuanto más lejos estés, mayor será el coste y los obstáculos no se expanden ni tienen coste. De forma que todo esto lo cumplimos mediante un algoritmo tipo Dijkstra, usando una cola de prioridad. Veamos más detenidamente su funcionamiento: 

  1. Coloca el destino en una cola de prioridad con coste 0.
  2. Extrae siempre la celda de menor coste
  3. Expande a sus vecinos con un coste adicional: 1 para vecinos horizontales/verticales y √2 para diagonales 
  4. Ignora las celdas que son obstáculos (valor 0 en el mapa).
  5. Llena la cuadrícula con el valor mínimo necesario para llegar al destino.
  6. Se detiene cuando ha expandido suficientemente cerca del robot (hasta el robot y un poco más).
  7. Devuelve el mapa: `wave_map`

Como esta es una de las funciones más importantes de nuestro programa vamos a resaltar algunos puntos importantes: 

Vamos a estudiarlo a traves de un pseudo código para entenderlo mejor conceptualmente
```python
def wavefront(goal, robot, map):

    # Crear una matriz CAMPO del tamaño del mapa donde guardaremos el coste
    # Inicialmente todas las celdas tienen coste infinito (desconocido)

    # Asignar al destino un coste 0 porque es nuestro punto de referencia

    # Crear una cola de prioridad (min-heap)
    # Esta estructura siempre devuelve la celda con el coste más pequeño

    # Insertar en la cola la celda del destino con coste 0

    # Inicializar una variable que guardará el coste óptimo del robot
    # (cuando la expansión alcance su celda)

    # Comenzar el bucle principal:
    # Mientras queden celdas por procesar en la cola:

        # Extraer la celda con el coste más pequeño conocido

        # Si este coste es mayor del que ya tenemos almacenado en CAMPO,
        # significa que es una versión antigua y no se debe procesar

        # Si esta celda es la del robot:
            # Guardar su nivel de coste (nivel_robot)

        # Si ya conocemos el nivel del robot
        # y el coste actual supera nivel_robot + un margen extra:
            # Podemos detener la expansión porque ya no aporta información útil

        # Para cada vecino de la celda actual (hasta 8 movimientos posibles):
            
            # Ignorar vecinos fuera del mapa

            # Ignorar vecinos que sean obstáculos en el mapa
            
            # Calcular el nuevo coste acumulado para ese vecino
            # (coste actual + coste del movimiento, que puede ser 1 o √2)

            # Si este nuevo coste es menor que el almacenado en CAMPO:
                # Actualizar CAMPO con el nuevo coste
                # Insertar el vecino en la cola con su coste actualizado

    devolver CAMPO
    

```

De esta forma construimos un campo de gradiente descendente, que atrae al robot hacia su objetivo.

Finalmente debemos combinar los dos campos resultantes para obtener `combined_cost = wave_map + penalty_map` que es un mapa que marca el camino óptimo hacia el destino, penaliza las celdas que se encuentren cerca de los obstáculos, es suave y estable y asegura que el robot navegue por calles estrechas y curvas. 

### Vídeo del funcionamiento 
Finalmente obtenemos este funcionamiento como resultado: 

[PathPlaning.webm](https://github.com/user-attachments/assets/10c0ceff-a426-4f26-9dd0-bf7b30e3f5f6)

## Path Navigation

Al igual que en el Path Planning debemos entender primero lo fundamental. Una vez generado el campo de costes con la planificación global, el robot debe ser capaz de navegar dentro del mapa y dirigirse hacia el destino. Para ello debemos implementar una lógica de navegación que en nuestro caso se basa en:

  1. Selección de un sub-objetivo local (local goal)

Donde hacemos uso de la funcion: `def local_goal()`

De la misma forma que antes usaremos un pseudocodigo para entender su funcionamiento 
```python

def local_goal(campo_costes, robot):

    # buscar celdas dentro de un radio alrededor del robot

    # elegir la celda:
       # - con menor coste
       # - libre de obstáculos
       # - suficientemente alejada del robot
       # - dentro de zona segura

   devolver celda como sub-objetivo

```

  2. Cálculo de la orientación deseada

Una vez elegido el sub-objetivo, el robot calcula la dirección desde su posición actual hasta el subpunto objetivo el ángulo ideal de orientación y el error angular (cuanto debe girar para mirar hacia el subpunto objetivo)

  3. Cálculo de velocidad lineal y angular

Finalmente aplicamos un control propocional (P) de foma que: si el robot está bien oreintado avanza rápido, si está muy girado reduce su velocidad lineal y si necesita girar aplica velocidad angular proporcional al error. 

  4. Avance del robot suavemente hacia el destino final

De esta forma el código se repite hasta que llega al destino donde se detiene al detectar que ha llegado al final.


<br>
<br>

## Mejoras 

Una vez desarrollada la primera versión del algoritmo de planificación y navegación, hemos realizado una serie de mejoras con el objetivo de optimizar el funcionamiento del robot, reducir cálculos innecesarios, mejorar la seguridad de las trayectorias y conseguir una navegación más organizada y fluida. De esta forma vamos a explicar los principales cambios introducidos respecto a la implementación original:

### 1. Optimización del cálculo de distancias 

En la primera versión, la distancia a los obstáculos la calculábamos mediante `distance_transform_edt()` durante la ejecución del programa y la función `obstacle_penalty()` generaba el mapa de penalizaciones cada vez que se ejecutaba el bucle principal, pero como el mapa permanece estático durante la simulación, no necesitamos repetir estos cálculos continuamente. Por eso, hemos trasladado su calculo al inicio del programa:

- `DIST_TO_OBSTACLE`: almacena la distancia de cada celda al obstáculo más cercano.

- `safe_map`: identifica las celdas que cumplen el margen de seguridad establecido.

- `PENALTY_MAP`: almacena las penalizaciones de las celdas próximas a los obstáculos.

Además, hemos sustituido la función `obstacle_penalty()` por una operación directa, simplificando la generación del mapa de penalizaciones.

<br>

### 2. Centralización de parámetros y optimización del código

Hemos agrupado al principio del programa los parámetros principales de planificación y navegación, así como los radios de seguridad, las penalizaciones, las ganancias del controlador y los límites de velocidad. También hemos definido `VECINOS` como una constante global, evitando reconstruir la lista de movimientos posibles cada vez que se ejecuta Dijkstra.

<br>

### 3. Visualización de la expansión de Dijkstra

Hemos incorporado una animación a la función `wavefront()` para visualizar cómo se expande el algoritmo desde el destino seleccionado.Además, mediante un contador de celdas expandidas, actualizamos la imagen cada 500 celdas y añadimos una pequeña pausa de 0.08 segundos para poder observar el proceso.

También hemos incorporado `mostrar_campo()`, que convierte los valores del campo de costes a una imagen en escala de grises de tipo uint8, adecuada para su representación mediante WebGUI.showNumpy().

<br>

### 4. Generación de una ruta completa mediante sub-destinos

En la implementación original, el robot seleccionaba un sub-destino local y volvía a calcular el campo de costes continuamente durante la navegación, pero en nuestra versión final hemos incorporado `construir_ruta()`, que utiliza repetidamente `local_goal()` para generar una secuencia de puntos desde la posición inicial del robot hasta el destino final.

<br>

## 5. Suavizado de la trayectoria

Una vez construida la ruta, hemos incorporado un procedimiento de suavizado que permite reducir los cambios de dirección innecesarios. Para ello, utilizamos  `linea_libre()`, que comprueba si existe una trayectoria recta entre dos puntos sin atravesar celdas penalizadas por su proximidad a los obstáculos. Posteriormente, dentro de `construir_ruta()`, buscamos conectar cada punto con el más lejano al que podemos llegar en línea recta.


<br>

## 6. Gestión de posiciones y destinos inseguros

Hemos incorporado `celda_cercana()`, que permite localizar la celda libre o segura más próxima a una posición determinada. Esta función se utiliza en dos situaciones:

- Si el robot se encuentra inicialmente sobre una celda considerada obstáculo, buscamos una celda transitable cercana desde la que comenzar la planificación.

- Si el usuario selecciona un destino sobre un obstáculo o demasiado cerca de una pared, buscamos una celda que cumpla el margen de seguridad y planificamos hacia ella.

Además, hemos marcado los bordes del mapa como obstáculos, evitando que el planificador considere estas zonas como transitables.



<br>

## 7. Mejora de la navegación y del control del robot

También hemos reorganizado el sistema de navegación mediante las funciones `mover_hacia()`, `parar()` y `distancia()`. En la primera versión, el controlador calculaba la orientación deseada utilizando las coordenadas de la cuadrícula. Ahora utilizamos directamente la posición real del robot y las coordenadas del siguiente sub-destino en el mundo.

Además, hemos incorporado dos mejoras. En primer lugar el seguimiento de sub-destinos donde el robot avanza por los distintos puntos de la ruta y pasa al siguiente cuando se encuentra a menos de 1.5 metros del actual. Y en segundo lugar el control de velocidad donde la velocidad lineal se limita teniendo en cuenta la distancia restante hasta el destino final, permitiendo reducirla al aproximarse a la meta.

<br>

## 8. Simplificación del bucle principal y gestión de destinos

Por último, hemos simplificado el bucle principal para separar la planificación de la navegación sin necesidad de mantener una máquina de estados explícita. Lo conseguimos utilizando `ultimo_destino` para detectar cuándo el usuario selecciona un nuevo objetivo y `ruta` para controlar si existe una trayectoria pendiente de seguir.

El funcionamiento queda organizado en tres situaciones:

- `Sin destino`: el robot permanece detenido esperando una nueva selección.

- `Nuevo destino`: el robot se detiene, calcula la ruta completa y reinicia el índice de seguimiento.

- `Navegación`: sigue los sub-destinos calculados hasta alcanzar el objetivo final.

Cuando el robot se encuentra a menos de 0.3 metros del destino, se detiene, vacía la ruta y vuelve a esperar una nueva selección. Además, si el usuario selecciona otro destino mientras el robot está navegando, se detiene y calcula una nueva trayectoria.


<br>


En conjunto, estas modificaciones permiten pasar de una planificación repetitiva basada en sub-destinos locales a un sistema que calcula una ruta completa antes de iniciar el movimiento, con una estructura más sencilla y organizada.

## Video Funcionamiento FINAL


https://github.com/user-attachments/assets/2cfa314e-feb0-4d6a-8878-ce5d12db92dc











