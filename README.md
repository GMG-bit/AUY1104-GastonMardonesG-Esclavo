# AUY1104-GastonMardonesG-Esclavo

Este es el workflow del repositorio SharedClient . Su responsabilidad es únicamente invocar al workflow reutilizable, delegando toda la lógica al repositorio
proveedor. Es intencionalmente minimalistas prueba de pipeline

##  Desglose del comando RUN npm ci --only=production 🛠️
Este comando es el estándar de oro para entornos de producción y CI/CD (Integración Continua).

RUN: Es una instrucción de Docker que le dice al motor que ejecute el comando que sigue durante la construcción de la imagen. Cada RUN crea una nueva capa en la imagen.

npm ci: Significa Clean Install. A diferencia de npm install, este comando:

Es estricto: Requiere que el archivo package-lock.json esté presente.

Es exacto: No intenta actualizar versiones; instala exactamente lo que está registrado en el lockfile.

Es limpio: Borra la carpeta node_modules si ya existe para evitar conflictos.

--only=production: Este argumento le dice a npm que ignore las devDependencies 🚫. Esto es vital en Docker para que la imagen final sea lo más pequeña posible, incluyendo solo lo necesario para que la app funcione (el "runtime").

| Característica | package.json (El Manifiesto) 📑 | package-lock.json (La Fotografía) 📸 |
| :--- | :--- | :--- |
| **Propósito** | Define las dependencias, scripts y metadatos del proyecto. | Registra la versión exacta de cada librería y sus sub-dependencias. |
| **Versiones** | Usa rangos (ej. `^4.17.1`), lo que permite actualizaciones menores. | Usa versiones fijas (ej. `4.17.1`) y hashes de seguridad. |
| **Flexibilidad** | **Alta**: es donde el desarrollador decide qué librerías usar. | **Nula**: es generado automáticamente para garantizar repetibilidad. |

Si moviéramos el paso de COPY package*.json ./ para que ocurra después de copiar todo el código de la carpeta src/, ¿qué crees que pasaría con el tiempo de espera cada vez que hagas un pequeño cambio en un mensaje de saludo de tu API?

 R: cada vez que corrijas un punto o una coma en tu código, Docker diría: "Oh, algo cambió en esta capa, así que tengo que invalidar todo lo que viene después". Y como la instalación de paquetes es pesada, perderíamos mucho tiempo.

## El concepto de "Capas" en Docker 🍰
Imagina que Docker es como una torre de panqueques:

Cada instrucción del Dockerfile es un panqueque nuevo que se pone encima del anterior.

Si cambias un panqueque de abajo (el código), Docker tiene que tirar todos los que estaban encima y cocinarlos de nuevo.

Al poner los archivos package.json primero, dejamos el panqueque que más tarda en cocinarse (npm ci) lo más abajo posible. Así, mientras no cambies las librerías, ese panqueque se queda ahí quietito y Docker solo cocina los de arriba (tu código).
=========================================================================================================
## .dockerignore 🚫
node_modules
npm-debug.log
.git
.gitignore

¿Por qué crees que es una mala idea copiar la carpeta node_modules de tu propia computadora directamente al contenedor, en lugar de dejar que el contenedor la instale con npm ci? 🤔

## R: 
El motivo técnico principal son los módulos nativos 🧩. Algunas librerías de Node.js (como las que manejan criptografía, bases de datos o procesamiento de imágenes) no son solo código JavaScript; se compilan en código binario específico para el sistema operativo durante la instalación.

Si instalas las dependencias en Windows y copias esa carpeta a un contenedor Linux Alpine, los binarios simplemente no funcionarán 🛑 porque hablan "idiomas" distintos a nivel de procesador.

Al usar el archivo .dockerignore, obligamos a Docker a ignorar tu carpeta local y a construir su propia versión de node_modules dentro de la imagen, garantizando que todo sea compatible con su entorno interno. 🛠️

## Conclusión sobre la Eficiencia 🚀
Ahora que hemos visto todo el proceso, podemos cerrar tu duda inicial: subir ambos archivos (package.json y package-lock.json) es mucho más eficiente y profesional que subir solo uno por estas razones:

Caché de Docker 🍰: Permite que Docker reutilice instalaciones previas si no has añadido librerías nuevas, ahorrando tiempo en cada construcción.

Velocidad con npm ci ⚡: Al tener el "candado" (lock), npm no pierde tiempo calculando versiones y va directo a la descarga.

Consistencia Total 🔒: Te aseguras de que las versiones de las librerías sean exactamente las mismas en tu computadora y en el servidor.