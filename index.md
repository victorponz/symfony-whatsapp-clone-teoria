---
typora-copy-images-to: ../../assets/img/primeros-pasos/
typora-root-url: ../../
title: Symfony Whatsapp clone
layout: post
categories: parte1
conToc: true
---
En esta parte del módulo vamos a implementar un clon de la aplicación Whatsapp.

Constará de dos partes diferenciadas:

1. El cliente, que es tarea vuestra
2. El servidor que ya está implementado y puedes descargarlo [aquí](https://github.com/victorponz/whatsapp-server)

### Funcionamiento del cliente

#### Mensajes

* Cuando se activa uno de los chats, todos los mensajes de los participantes se cargan asíncronamente mediante una petición ajax a `localhost:8080/messages/from/{toUserId}`. El método que responde a dicha ruta debe devolver en formato `json`  todos los mensajes enviados o recibidos por `toUserId` y el usuario logeado en el sistema
* Cuando se envía un mensaje, se debe hacer a la ruta `/post/touser/{toUserId}`. Como todavía no hemos conectado el servidor, para ver el nuevo mensaje hemos de recargar el chat o la página.
* Os dejo un [esqueleto](https://github.com/victorponz/whatsapp-clone-skeleton) de la aplicación cliente, que contiene todo lo necesario para poder empezar, incluida una plantilla, y las entidades `User` y `Message` junto con sus repositorios.
  Una vez descargado el esqueleto, realiza una migración para que se creen las tablas.

#### Contactos

Los contactos también se recuperan mediante una llamada asíncrona a `/contacts/{userId}`. En esta primera versión vamos a mostrar **todos** los usuarios registrados.

### Funcionamiento del servidor

Como conocéis, Whatsapp actualiza automáticamente los mensajes cuando recibe uno nuevo. Pero, ¿cómo implementar esta funcionalidad en una aplicación web?

En una aplicación web, el inicio de la comunicación entre un cliente y un servidor en el protocolo `HTTP` **siempre** lo realiza el cliente. Una vez finalizada esta comunicación se **cierra** la conexión.

Así, podemos montar una función en javascript que mediante uso de `setTimeOut` llamase a un api que recuperase los nuevos mensajes. Pero claro, en cuánto tiempo fijo el `timeout`: 1/2 segundo, 1 segundo.... Seguramente si llamo muy deprisa, la mayoría de veces no habrá mensajes nuevos mientras que si lo hago más lento los recuperaré en paquetes. Este tipo de funcionamiento se denomina [polling](https://en.wikipedia.org/wiki/Polling_(computer_science))

Es mejor usar una tecnología llamada [websockets](https://developer.mozilla.org/es/docs/Web/API/WebSockets_API). En este caso, se inicia un canal de comunicación bidireccional que se mantiene **siempre abierto**. Ahora el servidor y el cliente están continuamente *escuchando* la llegada de nuevos mensajes.

Para lanzar el servidor desde la raíz del proyecto, usa este comando:

```
 php -q bin/server port:9000
```

Esto iniciará el servidor y empezará a escuchar la conexión de nuevos usuarios.

Conectar el cliente es muy sencillo:

```javascript
$(document).ready(function(){
	//Open a WebSocket connection.
	websocket = new WebSocket("ws://localhost:9000/");
	
    //Connected to server
	websocket.onopen = function(ev) {
		console.log('Connected to server ');
	}
    
    //Connection close
	websocket.onclose = function(ev) { 
    	console.log('Disconnected');
    };
    websocket.onmessage = function(evt) { 
        var response 		= JSON.parse(evt.data); //PHP sends Json data
        //hacer lo que corresponda con response
    };
     
    //Error
	websocket.onerror = function(ev) { 
    	console.log('Error '+ev.data);
    };
    
});
```

Ahora, cada vez que el servidor nos envíe un mensaje llegará al evento `websocket.onmessage`

El servidor envía dos tipos de mensajes que debéis tratar:

* Cuando se da de alta un nuevo usuario, envía el siguiente mensaje mediante `broadcasting`:

  ```json
  {
    "type": "usermsg",
    "id": 12,
    "userName": "sonia",
    "info": "soy Sonia",
    "image": "719SdJJgEoL-AC-SX425-63c6deb1056ed.jpg"
  }
  ```

* Cuando llega un mensaje nuevo, envía el siguiente mensaje al emisor y al receptor:

  ```json
  {
    "type": "chatmsg",
    "toUserId": 1,
    "fromUserId": 2,
    "text": "hola",
    "timestamp": {
      "date": "2023-01-17 10:42:44.000000",
      "timezone_type": 3,
      "timezone": "Europe/Berlin"
    },
    "fromUserName": "Pepe"
  }
  ```


> -alert-Este servidor es de _andar por casa_ y hecho con el sólo  propósito de crear un clon casero de Whatsapp.
>
>  En una aplicación real usaríamos otras aproximaciones, [https://www.rabbitmq.com/](como colas de mensajes), [socket.io](https://socket.io/), etc

  
