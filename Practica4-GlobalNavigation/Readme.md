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

## Conclusión 

Nuestra solución cumple el enunciado porque generamos un campo de costes global usando una expansión tipo Dijkstra sobre la cuadrícula, asignando pesos adecuados y evitando obstáculos. Además, añadimos una penalización cerca de muros, tal como se solicita para prevenir colisiones. Por otro lado, durante la navegación, seleccionamos metas locales dentro de un radio seguro y finalmente, el robot avanza usando control lineal y angular hacia estos sub-objetivos, completando la planificación y navegación exigidas.

## Vídeo Funcionamiento FINAL 

https://github.com/user-attachments/assets/503bbb39-c83c-4a72-91ec-eeb4d59bc863









