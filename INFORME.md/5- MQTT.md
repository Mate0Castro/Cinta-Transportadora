# MQTT
Se trata de un protocolo de mensajería ligero para usar en casos de clientes que necesitan una huella de código pequeña, que están conectados a redes no fiables o con recursos limitados en cuanto al ancho de banda.
Arquitectura de MQTT:
MQTT se ejecuta sobre TCP/IP utilizando una topología PUSH/SUBSCRIBE. En la arquitectura MQTT, existen dos tipos de sistemas: clientes y brókeres. Un bróker es el servidor con el que se comunican los clientes: recibe comunicaciones de unos y se las envía a otros. Los clientes no se comunican directamente entre sí, sino que se conectan con el bróker. Cada cliente puede ser un editor, un suscriptor o ambos.
MQTT es un protocolo controlado por eventos, donde no hay transmisión de datos periódica o continua. Así se mantiene el volumen de transmisión al mínimo. Un cliente sólo publica cuando hay información para enviar, y un bróker sólo envía información a los suscriptores cuando llegan nuevos datos.
Arquitectura de los mensajes:
QoS 0: ofrece la cantidad mínima de transmisión de datos. Con este nivel, cada mensaje se entrega a un suscriptor una vez, sin confirmación, por lo que no hay forma de saber si los suscriptores recibieron el mensaje. Este método a veces se denomina “lanzar y olvidar” o “una entrega como máximo”. Debido a que este nivel asume que la entrega está completa, los mensajes no se almacenan para entregarlos a los clientes desconectados que luego se vuelven a conectar.

![image](https://github.com/user-attachments/assets/f2a9b622-0bb0-4c30-b239-507222b99958)
