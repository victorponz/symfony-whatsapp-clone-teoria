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
2. El servidor que sincroniza los mensajes entre todos los clientes usando websockets ya está implementado y puedes descargarlo [aquí](https://github.com/victorponz/whatsapp-server)

### Funcionamiento del cliente

#### Mensajes

* Cuando se activa uno de los chats, todos los mensajes de los participantes se deben cargar asíncronamente mediante una petición ajax a `localhost:8080/messages/from/{toUserId}`. El método que responde a dicha ruta debe devolver en formato `json`  todos los mensajes enviados o recibidos por `toUserId` y el usuario logeado en el sistema
* Cuando se envía un mensaje, se debe hacer a la ruta `/post/touser/{toUserId}`. Como todavía no hemos conectado el servidor, para ver el nuevo mensaje hemos de recargar el chat o la página.
* Os dejo un [esqueleto](https://github.com/victorponz/whatsapp-clone-skeleton) de la aplicación cliente, que contiene todo lo necesario para poder empezar, incluida una plantilla, y las entidades `User` y `Message` junto con sus repositorios.
Una vez descargado el esqueleto debes instalar las dependencias de paquetes con el comando `composer install` que debes ejecutar en la raíz del proyecto y realizar una migración para que se creen las tablas. 
* El servidor está continuamente escuchando cambios en la base de datos y cuando encuentra un mensaje nuevo lo envía al emisor y al receptor que deben estar escuchando para incorporarlo al chat entre ellos. Si se encuentra un nuevo usuario, lo envía a todos los usuarios, que deben estar escuchando y agregarlo a la lista de mensajes.

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

## Plantillas
Para pintar los contactos y los mensajes vamos a usar plantillas que definan el aspecto y donde definimos clases css para poder usar un selector. Por ejemplo, esta puede ser la plantila para el contacto:
```html
<script id="templateContact" type="x-template">    
<div data-id="-1" class="contact contact.id  px-3 flex bg-white border-2 border-gray-darker items-center cursor-pointer">
    <div>
        <img  class="contact.image h-12 w-12 rounded-full"
                src="/img/avatar.webp"/>
    </div>
    <div class="ml-4 flex-1 border-b border-grey-lighter py-4">
        <div class="flex items-bottom justify-between">
            <p class="contact.userName text-grey-darkest">
                name
            </p>
            <div class="text-xs text-grey-darkest">
                <p class="contact.lastTimestamp text-xs text-grey-darkest">
                12:45 pm
            </p>
            <p class="contact-numMessages hidden text-xs text-grey-darkest" style="background-color: #04AA6D; color: white; padding: 4px 8px;text-align: center; border-radius: 5px;">
                0
            </p> 
            </div>          
           
        </div>
        <p class="contact.info text-grey-dark mt-1 text-sm">
            Get Andrés on this movie ASAP!
        </p>
    </div>
</div>
</script>
```
Por ejemplo, tenemos la clase `contact.userName` que es donde va a ir el nombre del contacto.

Al cargar la página asignamos esta plantilla a una variable:
```javascript
const contacts =  document.getElementById("contacts"); //div donde vamos a pintar los contactos
const templateContact = document.getElementById("templateContact").innerHTML;
```
Y luego, cuando nos lleguen los datos de un contacto hacemos lo siguiente:
```javascript
function createDOMContact(m){
    var el = document.createElement('span');
    el.innerHTML = templateContact;
    el.getElementsByClassName("contact.userName")[0].innerHTML = m.userName;
    el.getElementsByClassName("contact")[0].setAttribute("data-id", m.id);
    el.getElementsByClassName("contact.info")[0].innerHTML = m.info;
    contacts.appendChild(el);
    //Ya solo te falta añadir los eventos clic
  }
```

