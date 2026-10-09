# PRACTICA 2

Página de Enunciado: `https://jderobot.github.io/RoboticsAcademy/exercises/AutonomousCars/follow_line/`

Vamos a explicar y entender el funcionamiento de FollowLine. Donde el objetivo ha sido crear e implementar usa serie de PIDs para permitir el funcionamiento de un robot sigue lineas cuyo objetivo es completar una vuelta en el menor tiempo posible. (seguimos con la idea de sistema reactivo)

## Imagen  

En primer lugar vamos a estudiar como hemos capturado la imagen:

1. `image = HAL.getImage()` → Captura una imagen del simulador (una matriz con los valores de color de cada píxel).
2. `cv2.cvtColor(image, cv2.COLOR_BGR2HSV)` → Convierte la imagen de formato BGR (por defecto en OpenCV) a HSV, que separa mejor el color (H) del brillo (V), facilitando detectar el rojo.

3. `cv2.inRange()` → Crea una máscara binaria (blanco y negro) que marca en blanco los píxeles cuyo color esté dentro de un rango.
Como el rojo está en dos zonas del espectro, usamos dos rangos ([0–10] y [160–179]) y obtenemos dos máscaras.

4. `cv2.bitwise_or(mask1, mask2)` → Une esas dos máscaras para obtener una sola que contenga todas las zonas rojas.

5. `M = cv2.moments(mask)` → Calcula los momentos de la máscara (propiedades geométricas).

Si `M['m00'] > 0`, significa que hay área detectada, y podemos calcular el centroide de la línea roja, si
`M['m00'] < 0` quiere decir que no la está detectando y comienza a girar sobre si mismo para volver a encontrarla. 

## Funcionamiento PIDs (Genérico)

- CONTROLADOR → P

El controlador proporcional responde en proporción al error. Es importante tener en mente que con kp muy baja casi no compensamos el error y de la misma forma con kp muy alta corrige tanto que generamos error en el sentido opuesto.

Formula: `u = -kp * err`

- CONTROLADOR → PD

El controlador derivativo responde de forma proporcional a la derivada (tendencia en el tiempo) del error. En nuestro ejercicio, las derivadas son restas del error en la iteración actual menos el error en la iteración anterior. El controlador D actua como un amortiguador, es decir, reduce los picos y las oscilaciones. 

Formula: `u = -(kp * err + kd * d_err )`

- CONTROLADOR → PID

El controlador integrador va a acumular el error en el tiempo (cuanto más tiempo pasa más actua). Si la ganancia ki es demasiado alta, el integrador puede acumular demasiado error, provocando oscilaciones o respuesta lenta. Por eso normalmente se satura o limita el valor de la integral (como veremos más adelante)

Formula: `u = -(kp * err + kd * d_err + ki * integral)`

## Funcionamiento PIDs (Específico)

El codigo se basa en el funcionamiento de dos PIDs que trabajan de forma coordinada para guiar al robot por la linea. El primero de ellos (Direction PID) es el encargado de controlar la dirección del robot, corrigiendo el error lateral entre el centro de la imagen y la posición de la linea detectada. El segundo (Speed PID) se encarga de regular la velocidad lineal del robot en función del mismo error.

CALCULO DEL ERROR 

En esta practica nos referimos al error `err` como `err = (cx - (Width / 2)) / (Width / 2)`. 

Con esta definición: 
- `err = 0` → el centro de la línea coincide con el centro de la cámara, es decir, el robot está alineado correctamente.
- `err > 0` → la línea está desplazada a la derecha respecto al centro, por lo que el robot debe girar a la derecha.
- `err < 0` → la línea está desplazada a la izquierda, indicando un giro hacia la izquierda.

Este error es normalizado dividiendo entre W/2, lo que lo mantiene en un rango aproximado de [-1, 1], facilitando la sintonización de los parámetros PID y evitando que las ganancias dependan del tamaño de la imagen.

### Direction PID

Este PID ajusta la velocidad angular del robot en función del error (desalineación con la línea), la integral (error acumulado) y la derivada (variación del error). Gracias a este controlador, el robot mantiene la línea centrada en su campo de visión y gira de manera suave y estable.

Adjunto la carpeta (Direction-PID) donde se puede observar la implementación y la evolución del código en cuanto al Direction PID. En dicha carpeta aparecen varias soluciones donde se van añadiendo controladores para obtener un resultado final completo:

    Direction-PID/
    ├── FollowLine-P.py
    ├── FollowLine-PD.py
    └── FollowLine-PID.py

Lo más destacable es la limitación del valor de la integral para evitar acomular demasiado error, tal y como hemos mencionado antes: `integral = max(min(integral, 0.5), -0.5)`

   VIDEO 
(Funcionamiento usando unicamente el Direction PID)

https://github.com/user-attachments/assets/eda7276b-c614-4f80-a860-4b68de7393e8


### Speed PID

Cuando el error aumenta (al tomar una curva), el controlador reduce la velocidad para evitar salidas de trayectoria, mientras que en zonas rectas (error pequeño) aumenta la velocidad, y de esta forma, el robot adapta dinámicamente su velocidad a las condiciones del recorrido, combinando precisión y rapidez.

De esta forma podemos decir que cuanto mayor es el error, menor es la velocidad.

Adjunto la carpeta (Speed-PID) donde se puede observar la implementación y la evolución del código en cuanto al Speed PID funcionando junto a Direction PID. En dicha carpeta aparecen varias soluciones donde se van añadiendo controladores para obtener un resultado final completo:

    Speed-PID/
        ├── FollowLine-P.py
        ├── FollowLine-PD.py
        └── FollowLine-PID.py


Es importante destacar la limitación de la velocidad en un rango `[Vmin, Vmax]` a través de: `V = max(Vmin, min(V, Vmax))`, lo que evita valores de velocidad fuera del rango físico permitido y protege el sistema frente a oscilaciones o errores acumulados.



   VIDEO 
(Funcionamiento  Direction PID + speed PID)

https://github.com/user-attachments/assets/03842e09-dfcc-4f0a-98ce-a11bbd44b107

Es sorprendente observar la diferencia de tiempo con el uso de ambos PIDs en lugar de usar unicamente uno.


## Mejoras 

En esta última versión hemos introducido una serie de mejoras sobre la implementación anterior con el objetivo de aumentar la velocidad del coche y mejorar la estabilidad durante el seguimiento de la línea roja, reduciendo las oscilaciones sin renunciar a un comportamiento rápido.

1. `Modificación de la región de interés (ROI)`

Antes calculábamos el centroide utilizando todos los píxeles rojos de la imagen mediante `cv2.moments(mask)`. Esto permitía considerar toda la línea visible, pero también hacía que el cálculo dependiera de los tramos que aparecían a diferentes distancias del vehículo.

En esta versión mejorada hemos incorporado una región de interés (ROI) situada entre el 45 % y el 70 % de la altura de la imagen. De esta forma, el centroide se calcula exclusivamente a partir de la línea detectada dentro de esta franja. La finalidad es controlar el vehículo utilizando una región concreta de la imagen, dándonos más margen para anticiparse y maniobrar. 

Además, hemos incorporado un rectángulo verde para visualizar la ROI.

2. `Filtrado de la derivada del Direction PID`

También hemos mejorado la derivada del Direction PID, ya que antes la calculabamos directamente mediante la diferencia entre el error actual y el anterior, pero ahora hemos incorporado un filtro de primer orden para suavizar sus variaciones.ç


Esta modificación reduce la influencia de los cambios bruscos de la derivada entre iteraciones, lo que puede ayudar a disminuir las correcciones excesivas del volante. El filtrado también introduce cierto retraso, por lo que hemos tenido que comprobar su efecto con cuidado.


3. `Limitación de la velocidad angular`

Antes no imponíamos un límite explícito a la velocidad angular calculada por el PID. Pero, en esta nueva versión hemos establecido un rango de actuación.

Con ello evitamos que el controlador genere órdenes angulares superiores a los límites establecidos. Esta medida permite controlar la intensidad máxima de las correcciones, aunque el límite lo hemos ajustado según el comportamiento observado en las curvas.

4. `Modificación del control adaptativo de velocidad`

Hemos reducido el coeficiente del factor de velocidad, de modo que la velocidad se reduce menos para un mismo error lateral. Por tanto, el coche conserva una mayor velocidad en las curvas cuando existe una desviación moderada respecto al centro de la imagen.





Conclusión: El propósito de estas mejoras es conseguir un equilibrio entre velocidad, precisión y estabilidad. La efectividad de cada modificación debe evaluarse mediante pruebas en el simulador, comparando el tiempo por vuelta, las oscilaciones y las posibles salidas de trayectoria.



## Resultado final 

[Grabación de pantalla desde 2026-10-09 11-53-45.webm](https://github.com/user-attachments/assets/90cf6984-08fb-46ee-9485-a9ef70005061)








