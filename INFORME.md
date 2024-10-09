# Cinta-Transportadora
Vamos a diseñar una cinta transportadora que clasifique objetos según su altura. Utilizaremos servomotores para mover los objetos a distintas cajas de acuerdo con su tamaño. Además, implementaremos un servidor web que registrará la cantidad de objetos clasificados, utilizando comunicación MQTT a través de Node-RED.
Nuestro objetivo cómo proyecto es que podamos transportar 2 distintos tipos de objetos (alto y bajo) en un buen tiempo y con su dicha estabilidad.

Tenemos tres instancias, primero el modelado 3D y el esquemático del circuito en kicad, segundo preparación de la comunicación (I2C, SPI, MQTT con Node-Red) y por último el armado de la transportadora y su prueba final de funcionamiento.  

# INDICE
# Iniciación de nuestro proyecto
# Explicación de la tecnología que usamos
2 Motores paso a paso
1 Esp 32
2 Fotoresistencias
2 Láser
1 Pulsador
2 Servomotores
2 Driver A4988
Comunicación Node-Red
# Fotos de nuestro proceso
Lo primero que hicimos fue ingresar a Thingiverse para buscar diseños de cintas transportadoras y encontramos este: https://www.thingiverse.com/thing:3036233. Modificamos este diseño en Tinkercad, reduciendo sus dimensiones para que no fueran tan grandes. También hicimos lo mismo con los rodillos y los soportes.
# Fuentes 

