# Microservicio de Órdenes ("Orders") - TechMarket

Este repositorio contiene el código fuente, pruebas unitarias y archivos de configuración para el microservicio crítico **"Orders"** (representado académicamente por la API de Node.js/Express `demo-api`) de la organización **TechMarket**. 

El objetivo de este proyecto es transformar un pipeline básico de despliegue en una **"Fortaleza Digital"** altamente disponible, resiliente y auto-recuperable sobre clústeres de Kubernetes (Amazon EKS / K3s).

---

## 1. Arquitectura de Despliegue en Kubernetes (EKS / K3s)

El microservicio se ejecuta de manera contenerizada y es administrado en el clúster usando los manifiestos ubicados en el directorio [k8s/](file:///k8s):
*   [deployment.yaml](file:///k8s/deployment.yaml): Define las especificaciones de ejecución de los pods de la API, las réplicas, puertos y sondas de salud.
*   [service.yaml](file:///k8s/service.yaml): Configura un Service de tipo `NodePort` mapeando el puerto interno de la API (3000) al puerto público accesible del clúster (30090).
*   **demo-blue-green/**: Manifiestos de apoyo (`blue.yaml`, `green.yaml`, `configmaps.yaml` y `service.yaml`) utilizados para la demostración manual de despliegues avanzados.

---

## 2. Estrategia de Despliegue Avanzada: Blue-Green

Para evitar caídas de servicio (*downtime*) durante la actualización del microservicio, se implementó la estrategia **Blue-Green** utilizando recursos nativos de Kubernetes:

1.  **Entornos Aislados:** Coexisten dos despliegues idénticos en el clúster: el entorno productivo actual (**Blue**, corriendo la versión estable v1) y el nuevo entorno a desplegar (**Green**, corriendo la nueva versión v2).
2.  **Enrutamiento Dinámico:** Un único Service de Kubernetes (`nginx-blue-green-service` o similar) expone la aplicación. Éste dirige las conexiones entrantes basándose en un selector de etiqueta (`color: blue` o `color: green`).
3.  **Switch de Tráfico sin Downtime:** El despliegue de la versión Verde se realiza sin afectar a los usuarios. Una vez verificado el estado de salud de los pods Verdes, el tráfico se redirige al 100% en caliente con el comando:
    ```bash
    sudo k3s kubectl patch service nginx-blue-green-service -p '{"spec":{"selector":{"color":"green"}}}'
    ```
4.  **Reversión Instantánea:** Si se detectan anomalías post-despliegue, se revierte el tráfico al entorno Blue de inmediato:
    ```bash
    sudo k3s kubectl patch service nginx-blue-green-service -p '{"spec":{"selector":{"color":"blue"}}}'
    ```

---

## 3. Configuración de Sondas de Salud (Health Checks)

Los pods de la API integran sondas automáticas para controlar su estado, apuntando al endpoint `/health` expuesto en [index.js](file:///src/index.js):

*   **Readiness Probe (Sonda de Disponibilidad):** Verifica si el pod está listo para recibir peticiones externas.
    *   `initialDelaySeconds: 30`: Otorga un margen de 30 segundos para que la aplicación cargue recursos y conecte bases de datos antes de consultar su salud.
    *   `timeoutSeconds: 10`: Límite máximo de espera por respuesta antes de considerar la consulta fallida.
*   **Liveness Probe (Sonda de Vida):** Supervisa que la aplicación no haya entrado en congelamiento (*deadlock*).
    *   `initialDelaySeconds: 30` y `timeoutSeconds: 10`. Si falla repetidas veces, Kubernetes destruye el pod y lo reinicia automáticamente.

---

## 4. Pipeline de CI/CD y Remediación Automática (Rollback)

El flujo de integración y entrega continua se define en [.github/workflows/client.yaml](file:///.github/workflows/client.yaml). 
Cuando se sube una nueva etiqueta de versión (`git push origin v*.*.*`), la pipeline del cliente se dispara y delega la ejecución al **workflow reutilizable maestro** (`deploy-api.yaml`).

### Reacción ante Fallos de Salud:
Si una versión defectuosa (que no pasa la sonda de salud) se despliega, ocurre la siguiente remediación automática:
1.  El comando `kubectl rollout status` se queda esperando a que los pods estén en estado *Ready*.
2.  Al expirar el timeout configurado de **60 segundos**, el paso de despliegue principal falla.
3.  La pipeline intercepta la falla mediante la condicional `if: failure()` y ejecuta de forma autónoma el paso de **Rollback automático** (`kubectl rollout undo`), devolviendo el clúster a la versión anterior de forma segura.

---

## ANEXO A: Eficiencia en la Construcción de Docker (Dockerfile & Layers)

La optimización de la construcción de la imagen Docker en este microservicio se fundamenta en los siguientes conceptos técnicos:

### Desglose del comando `RUN npm ci --only=production`
Este comando es la mejor práctica para entornos de producción y CI/CD:
*   **npm ci (Clean Install):** Borra la carpeta `node_modules` local (si existe) y realiza una instalación exacta y estricta basada únicamente en el archivo `package-lock.json`, impidiendo actualizaciones de versiones accidentales en producción.
*   **--only=production:** Excluye las dependencias de desarrollo (`devDependencies` como Jest o Supertest), reduciendo significativamente el peso de la imagen final del runtime.

| Característica | package.json (El Manifiesto) | package-lock.json (La Fotografía) |
| :--- | :--- | :--- |
| **Propósito** | Define las dependencias generales y scripts. | Registra la versión exacta de cada librería y sub-dependencia. |
| **Versiones** | Permite rangos dinámicos (ej: `^4.17.1`). | Utiliza versiones fijas y hashes SHA de seguridad. |

### Caché y Capas en Docker
En el [Dockerfile](file:///Dockerfile), se copian primero los archivos `package*.json` y se ejecuta `npm ci` **antes** de copiar el código fuente de `src/`. 
Esto se realiza debido a la caché de capas de Docker. Si realizamos un cambio menor en el código de nuestra API, Docker invalidará únicamente la capa correspondiente al código (`COPY src/ ./src/`), pero reutilizará la capa de dependencias compilada previamente. Esto reduce el tiempo de compilación de minutos a segundos en cada ejecución de la pipeline.

### Uso de `.dockerignore`
El archivo [.dockerignore](file:///.dockerignore) obliga a Docker a ignorar carpetas locales pesadas como `node_modules`. Esto evita importar binarios nativos compilados en la computadora de desarrollo (por ejemplo, con sistema operativo Windows) al contenedor basado en Linux (Alpine), garantizando compatibilidad multiplataforma y una construcción limpia.

---

## 5. Declaración de Uso de Inteligencia Artificial

De acuerdo con las directrices académicas y de integridad de Duoc UC, se declara el uso guiado de inteligencia artificial generativa (**Antigravity, por Google DeepMind**) para asistir en la redacción técnica del informe de arquitectura y estructuración del presente README.md.

---

## 6. Referencias Bibliográficas (Norma APA)

*   Express. (2024). *Express production best practices: Performance and reliability*. Express Docs. https://expressjs.com/en/advanced/best-practice-performance.html
*   Kubernetes. (2023). *Kubernetes Documentation: Configure Liveness, Readiness and Startup Probes*. Kubernetes IO. https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/
*   Docker. (2024). *Docker documentation: Optimize builds with caching*. Docker Docs. https://docs.docker.com/build/cache/