---
layout: post
title: Odoo Bajo la Lupa ¿Están tus Instancias a salvo?
date: 2025-11-19 10:18:00
categories: [Odoo, Security, Exposed]
---

Que tal analistas que duermen menos que sus logs... En esta ocasión venga a hablarles de un tema de interés que vengo revisando desde
hace algún tiempo y no es nada menos que aspectos fundamentales de seguridad en ***Odoo, un ERP de código abierto 100% web***.

En la actualidad, ***Odoo*** se ha convertido en uno de los ERPs más populares entre empresas pequeñas y medianas, debido a su flexibilidad,
modularidad, y accesibilidad. Sin embargo, como cualquier herramienta, su seguridad es crucial. Asi que, si estas usando Odoo o piensas implementarlo en tu empresa sería conveniente tomar nota.

Y bueno, comenzamos haciendo una busqueda usando Shodan una herramienta util para el proposito de encontrar instancias de Odoo
usando una query como la que veremos acontinuación:

```title:"Odoo"```

a esta podemos refinarla un poco más filtrando por país, de la siguiente forma:
```title:"Odoo" country:"US"```

otra query que nos podria arrojar resultados interesantes, sería algo como esto:
```port:8069 http.html:"DEBUG" country:"US"```

para entender que hace cada query, desglosare parte por parte:
```port:8069``` -> Filtra por el puerto 8069, puerto predeterminado que utiliza Odoo para su interfaz web.

```http.html:"DEBUG"``` -> Busca instancias donde la palabra "DEBUG" aparece en el código HTML de la página. Esto puede indicar que la instancia de Odoo está corriendo en modo debug/depuración, lo que puede dar paso a exponer información sensible o detalles de desarrollo que no deberían ser visibles en producción.

```country:"US"``` -> Limita los resultados a servidores ubicados en Estados Unidos.

Algunas de las instancias corriendo Odoo que aparecen en los resultados pueden redireccionar a un sitio web. Algo totalmente normal, ya que Odoo también nos da la posibilidad de crear un sitio web, usando la herramienta de arrastrar y soltar (drag and drop), más info pueden encontrar en el siguiente enlace: 
- https://www.odoo.com/blog/business-hacks-1/how-to-create-your-own-website-for-free-and-easily-1717. 
Sin embargo, nuestro interes va en encontrar algo más jugoso, como listar las bases de datos disponibles, y acceder al panel para obtener un backup, eliminar o crear una nueva base de datos.

Basta con incluir la rutas siguientes rutas al final de la url que estamos examinando:
- ***https://example.com/web/database/manager*** -> esta ruta nos llevara al panel donde podemos crear una nueva base de datos, duplicar bases de datos existentes, eliminar una base de datos, restaurar un backup, y acceder a configuraciones sensibles de la base de datos.

- ***https://example.com/web/database/selector*** -> esta ruta nos permite ver la lista completa de las bases de datos disponibles, seleccionar y cambiar entre diferentes bases de datos, y potencialmente acceder a bases de datos de diferentes clientes o entornos.


![Gestor de base de datos de Odoo Expuesto sin clave](/assets/images/odoo-db-manager-exposed.png)

![Gestor de base de datos de Odoo](/assets/images/odoo-db-manager.png)

![Configurar Clave master de Odoo](/assets/images/odoo-master-pass.png)


Dato de color, puede que algunas de las instancias corriendo Odoo usen credenciales por defecto, facilitando la tarea a un atacante. Ahora, probar contra cada una de las intancias 1 a 1 sería una tarea que demandaría una buena cantidad de tiempo y no sería la mejor forma, de allí que, una herramienta como
lo es ***OdooMap*** desarrollada por ***Karrab*** nos facilite la tarea automatizando parte de las tareas a realizar {reconocimiento, recolección de información, enumeración de bases de datos, fuerza bruta de credenciales, enumeración de modelo, extracción de datos, obtener información de los plugins instalados}. (https://karrab7.com/articles/Pentesting-Odoo-Applications-with-OdooMap)

Hasta aca, se deben estar preguntando ***¿Por qué es peligroso que estén expuestas las rutas mencionadas previamente?***. 
Y bueno, aca viene lo bueno. Si existe una instancia que no tiene la autenticación activada, un atacante podría:
- Crear/administrar bases de datos, acceder a información de clientes, finanzas y empleados. 
- Impactar al negocio de forma crítica eliminando todas las bases de datos de la organización.
- Exfiltrar de forma masiva los datos del sistema.

Y bueno, se preguntaran que podemos hacer respecto a este punto?
- Podemos limitar restringir el acceso a la ruta /web/database/management.
- Aplicar parches y actualizaciones de seguridad regularmente para mantener la seguridad del servidor.
- Activar logs de accesso para rastrear los intentos de inicio de sesion, especialmente los fallidos.
- Limitar el numero de intentos de inicio de sesion para prevenir ataques de fuerza bruta.
- Activar politicas de experiracion de contraseñas para que los usuarios las renueven periódicamente.
- Activar el segundo factor de autenticacion

En una proxima entrega hablaremos de los plugines maliciosos Odoo...


**Referencias**
- https://www.odoo.com/security
- https://odoo-community.org/blog/news-updates-1/basic-guide-to-odoo-security-182
- https://www.odoo.com/forum/help-1/odoo-login-page-databases-administration-68558
- https://www.odoo.com/forum/help-1/securing-webdatabasemanager-121799
- https://github.com/MohamedKarrab/odoomap