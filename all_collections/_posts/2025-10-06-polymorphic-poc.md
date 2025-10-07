---
layout: post
title: Poc de un infostealer polimórfico con IA que roba tus credenciales desde VSCode
date: 2025-10-06 10:18:00
categories: [PoC, Research, Extension, Supply Chain Attacks]
---

Hace algún tiempo estuve trabajando en una PoC para demostrar cómo un atacante puede obtener credenciales de servicios como AWS, llaves SSH, y otros recursos sensibles atacando la cadena de suministro, y en esta ocasión me he puesto manos a la obra con una totalmente diferente.

Recientemente, se ha visto un incremento considerable en este tipo de ataques. Plataformas como NPM, PyPI e incluso Crates han sido afectadas por el volumen de malware que se distribuye a través de ellas. Aunque es cierto que se han tomado ciertas medidas para mitigar problemas como los Domain Resurrection Attacks (https://blog.pypi.org/posts/2025-08-18-preventing-domain-resurrections/), Starjacking, Typosquatting, entre otros, todavía existen amenazas importantes.

Mi enfoque actual está centrado en crear una extensión para Visual Studio Code que filtre datos sensibles del usuario que la instala. Para lograrlo, debemos entender qué son las extensiones de VSCode, cómo crear una, y de qué forma un atacante puede valerse de una para comprometer su objetivo.

Entrando en materia, las extensiones son add-ons que permiten expandir la funcionalidad del editor de código. En el caso de VSCode, estas se distribuyen a través de la Extension Marketplace, propiedad de Microsoft, de forma gratuita, siguiendo un proceso detallado en la web oficial. 

Antes de que una extensión se haga pública, pasa por un escáner automático para detectar posibles problemas de seguridad, y también se verifica la identidad del publisher (esto último no es obligatorio). Sin embargo, es preocupante la cantidad de extensiones que poseen componentes con vulnerabilidades críticas reportadas, tal como muestra el investigador de seguridad John Hammond (https://docs.google.com/spreadsheets/d/12GIzrSzzU-_Ok4pPigUJYSxKO2ZYSmDwr1OJy6T2X40/edit?gid=1397736002#gid=1397736002).

De manera similar, la empresa de seguridad Koi, en una prueba de concepto explicarón cómo lograron vulnerar en "30 minutos" varias compañías multimillonarias a través de una extensión falsa, un copycat llamado "Darcula Official", que aparecía como verificada y sin repositorio en el marketplace. Pueden leer más aquí: https://www.koi.security/blog/1-6-how-we-hacked-multi-billion-dollar-companies-in-30-minutes-using-a-fake-vscode-extension.

Podríamos seguir enumerando uno por uno los incidentes relacionados con las extensiones de VSCode, pero nos tomaría un largo tiempo.

Ahora, revisemos los vectores de ataque para comprometer Visual Studio Code de forma remota
 * Un adversario compromete el proyecto open source de VSCode o una de sus dependencias para introducir código malicioso.
 * Un adversario clona el repositorio de VSCode, introduce código malicioso e intenta engañar a los usuarios para que descarguen la versión comprometida.
 * Un adversario crea una extensión maliciosa y publicarla en el VSCode Marketplace.
 * Un adversario crea una tarea maliciosa y la commitea a un repositorio público utilizado por desarrolladores.

Para esta PoC, simularemos ser un atacante cuya intención principal es robar credenciales. Para esto, crearemos una extensión maliciosa con fines educativos en un **entorno controlado** e instalandola desde un archivo vsix.

**Dato interesante:** Podemos crear extensiones y otros artefactos como *temas*, *code snippets*, *keymaps*, *extension packs*, *web extensions*, *paquetes de lenguaje* y *notebook renderers* usando la herramienta recomendada por VSCode llamada Yeoman.

En un escenario real, un atacante podría usar técnicas como la ingeniería social para hacer que desarrolladores instalen nuestra extensión maliciosa, valiendose del hecho de que muchas personas no realizan due diligence y se dejan llevar por métricas de vanidad, como el número de descargas, el nombre del autor o incluso por tener un badge de verificación (esto último no garantiza que no puedas ser engañado).

A través de datos sintéticos —que se pueden comprar a un bajo costo en ciertos portales especializados en vender estrellas falsas en GitHub, reseñas, etc.— pueden generarse señales de confianza en las posibles víctimas.

Ahora queda pensar qué tipo de extensión y nombre usar... 
La extensión a desarrollar tendrá el nombre "Qwen Plus", que hace referencia al modelo de Alibaba. 

**Nota:** Se puede usar un paquete NPM en una extensión de VSCode; sin embargo, me decidí a probar algo diferente: desarrollar lo que sería un AI-SYNTHESIZED, POLYMORPHIC INFOSTEALER, inspirado en un paper muy interesante: AI-SYNTHESIZED, POLYMORPHIC KEYLOGGER WITH ON-THE-FLY PROGRAM MODIFICATION de HYAS Labs.

**¿Por qué un atacante se enfocaría en los desarrolladores?**
Porque, al comprometer a un desarrollador, podría ganar acceso a recursos críticos como el código fuente, credenciales sensibles y, lo que es aún más peligroso, a entornos de producción.

Crear una extensión para VSCode no es complicado. Lo realmente retador viene después, cuando tienes que pasar por las medidas de seguridad del marketplace que se detallan en su web oficial. entre ellas:

 * Tanto las extensiones nuevas como las actualizadas en su versión deben pasar por un análisis de malware antes de ser publicadas, para garantizar su seguridad. Si todo está en orden, entonces se hace disponible para el público.

 * Si una extensión es reportada y verificada como maliciosa, o se encuentra una vulnerabilidad en alguna de sus dependencias, entonces se remueve del marketplace y se agrega a una lista negra ("kill list"), de modo que se desinstale automáticamente en caso de estar instalada en VSCode. De lo anterior me surge una gran duda: ¿por qué existen algunas extensiones en el marketplace con vulnerabilidades reportadas en sus dependencias? ¿No deberían haberse eliminado ya? Vaya dato perturbador...

 * Se impide que los autores de extensiones roben los nombres de editores oficiales, como Microsoft o RedHat, y de extensiones populares, como GitHub Copilot.

 * La funcionalidad Workspace Trust te permite decidir si el código en la carpeta de tu proyecto puede ser ejecutado por VS Code y las extensiones sin tu aprobación explícita. De alli que el equipo de Microsoft nos comente: "Solo deberías instalar y ejecutar aquellas extensiones de las que confíes en el publisher.". Entonces "You click 'Trust', you lose." Sin embargo, el problema radica en que la extensión se ejecuta antes del modal de Workspace Trust. Esto significa que incluso si el usuario nunca da permiso explícito, el código malicioso podría haberse ejecutado ya.

**¿Qué hace exactamente Workspace Trust?**
Permite desactivar funcionalidades como tareas, debugging, ciertas configuraciones y extensiones, reduciendo así el riesgo cuando se abre código no confiable. Sin embargo, una forma de saltarse esta protección es instalando una extensión directamente desde un archivo .vsix ...lo que utilicé para desarrollar la PoC.

Otro detalle interesante que encontré en otros investigaciones, es el hecho que una extensión tiene los mismos permisos que cualquier otro proceso de usuario; es decir, no existe sandboxing. Es decir que una extensión es libre de leer y escribir archivos en disco, hacer peticiones de red, o llamar a otras APIs y librerías disponibles. Además, todas las extensiones instaladas en VSCode se actualizan automáticamente.

Para ilustrar mejor la PoC, les comparto un gráfico...
![Prueba de concepto de infostealer polimórfico VSCode Extension ](/assets/images/aipoci.png)


En este punto se preguntarán, ¿por qué usar inteligencia artificial?
Porque permite generar variaciones rápidas del código malicioso (simulando comportamientos tipo polymorphic), lo que dificulta su detección por los programas antivirus que dependen de firmas conocidas.

Ahora bien, ¿qué implica que sea una extensión asistente de código basada en IA?
El tráfico de red se mezcla con comunicaciones legítimas. Entonces ¿Quién sospecharía de una petición hacia una API de modelo de lenguaje?

En resumen, estas son las principales funcionalidades de la extensión::
 * Evita la detección mediante generación dinámica de código y ejecución en memoria.
 * Genera, ofusca o modifica su propio código en tiempo de ejecución.
 * Codifica en Base64 y ejecuta la carga útil sin dejar rastro en disco.

Mientras que las acciones que ejecuta van desde:
 * Recolección de información (footprinting) antes de lanzar cualquier tipo de ataque, ya que al tener datos del sistema permite al malware-AI tomar mejores decisiones y responder de forma correcta.
 * Solicitar el código necesario para exfiltrar credenciales a un endpoint de un servidor (en este caso local). También se podría usar un canal de Slack o Teams (no aplica en este PoC).

**Nota:** Las peticiones para generar el código malicioso se hacen simulando ser una petición correspondiente a telemetría luego de unos cuantos minutos y saltandose los filtros del modelo empleado.

**Limitantes**
* Se pueden dar errores en el código generado por IA que generan comportamientos inconsistentes durante las ejecuciones.

**Conclusión**
Para esta PoC, se creo una prueba de concepto (PoC) capaz de generar dinámicamente un infostealer malicioso basado en código generado por OpenAI y ejecutarlo directamente en memoria, sin crear ningún archivo en disco. Dada su naturaleza polimórfica y el uso de técnicas de evasión, logra pasar inadvertido ante controles tradicionales de seguridad. 
Esto demuestra cómo una extensión aparentemente inocente puede convertirse en una amenaza real si se aprovechan tecnologías emergentes como la IA combinadas con técnicas clásicas de evasión.

**Referencias**
- [VS Code Extension Runtime Security](https://code.visualstudio.com/docs/configure/extensions/extension-runtime-security)
- [Getting Started with VS Code Extension Anatomy](https://code.visualstudio.com/api/get-started/extension-anatomy)
- [Publishing Your VS Code Extension](https://code.visualstudio.com/api/working-with-extensions/publishing-extension)
- [GitHub Issue #52116](https://github.com/microsoft/vscode/issues/52116)
- [Malicious Helpers in VS Code Extensions](https://www.reversinglabs.com/blog/malicious-helpers-vs-code-extensions-observed-stealing-sensitive-information)
- [Can You Trust Your VS Code Extensions?](https://www.aquasec.com/blog/can-you-trust-your-vscode-extensions/)
- [Malicious Pull Request Infects VS Code Extension](https://www.reversinglabs.com/blog/malicious-pull-request-infects-vscode-extension)
- [HYAS AI-Augmented Cyber Attack White Paper](https://www.hyas.com/hubfs/Downloadable%20Content/HYAS-AI-Augmented-Cyber-Attack-WP-1.1.pdf)