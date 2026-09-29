# PRACITCA 1

Página de Enunciado: `https://jderobot.github.io/RoboticsAcademy/exercises/MobileRobots/vacuum_cleaner_loc`

Vamos a explicar y entender el funcionamiento de Localiced Vacuum Cleaner. Donde nuestro objetivo es implementar la lógica de navegación de un robot aspirador haciendo uso de la información de localización del robot dentro de un mapa conocido. El objetivo principal del algoritmo utilizado (Algoritmo BSA, en este caso) es cubrir la mayor superficie posible, tratando de recorrer de forma sistemática las zonas transitables del mapa y reduciendo al mínimo las áreas que quedan sin visitar.

## Funcionamiento 

El funcionamiento del sistema se divide en tres partes principales, donde encontramos el registro del mapa, la planificación y el movimiento. Primero, se relaciona la posición del robot con el mapa para saber en todo momento dónde se encuentra. Después, se realiza una planificación en estático, es decir, antes de que el robot comience a moverse, utilizando el algoritmo BSA y generando una ruta que permita cubrir la mayor parte del entorno. Finalmente, el robot ejecuta la ruta calculada, desplazándose por las distintas zonas y teniendo en cuenta los obstáculos y puntos de retorno.


### 1- Registro del mapa


### 2- Algoritmo BSA 


[Video_planBSA.webm](https://github.com/user-attachments/assets/8a60f8e1-c7a5-4b9a-80e9-58d7f5803611)


### 3 - Movimiento 



## Video
