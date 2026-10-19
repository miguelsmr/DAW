# Arquitectura de despliegue de TaskFlow: instalación directa vs. contenedor

**Alumno:** Miguel Luzón Prado · **Práctica 1.4** · Despliegue de Aplicaciones Web (2º DAW)

---

## 1. Las dos arquitecturas en TaskFlow

### A) Instalación directa (práctica 1.3)

```
┌──────────────────────── Ubuntu Server ────────────────────────┐
│  Python 3.13 (deadsnakes)  +  venv  +  pip install            │
│  Gunicorn (127.0.0.1:8000) ─▶ worker Uvicorn ─▶ FastAPI       │
│  Dependencias, versiones y config. mezcladas con el sistema   │
└───────────────────────────────────────────────────────────────┘
```

### B) Contenedor Docker (práctica 1.4)

```
┌──────────────────────── Ubuntu Server ────────────────────────┐
│  Docker Engine                                                │
│  ┌──────────── contenedor "taskflow" (taskflowapi:1.0) ────┐  │
│  │ python:3.13-slim + requirements + app                   │  │
│  │ Gunicorn (0.0.0.0:8000) ─▶ Uvicorn ─▶ FastAPI           │  │
│  └─────────────────────────────────────────────────────────┘  │
│        ▲ -p 8000:8000                                          │
└───────────────────────────────────────────────────────────────┘
```

En la instalación directa tuve que instalar Python 3.13 desde un PPA porque Ubuntu trae la 3.12, crear el venv e instalar dependencias a mano. Con Docker, todo eso está descrito en el `Dockerfile` y basta con `docker build` y `docker run`: el anfitrión solo necesita Docker.

---

## 2. Comparativa

| Aspecto | Instalación directa | Contenedor Docker |
|---|---|---|
| Preparación del servidor | Instalar Python, venv, dependencias a mano | Solo Docker Engine |
| Reproducibilidad | Depende de lo que tenga cada máquina ("en mi máquina funciona") | La imagen es idéntica en desarrollo, clase, producción y nube |
| Aislamiento | Comparte librerías y Python del sistema | Sistema de ficheros, procesos y red propios (namespaces) |
| Conflictos de versiones | Posibles entre aplicaciones del mismo servidor | Cada contenedor lleva sus versiones |
| Rendimiento | Máximo, sin capas intermedias | Prácticamente igual (comparte el kernel), mínima sobrecarga en red/disco |
| Arranque | Inmediato | Segundos (imagen ya construida) |
| Actualizar / volver atrás | Reinstalar paquetes; *rollback* difícil | Nueva etiqueta de imagen; *rollback* = arrancar la anterior (`taskflowapi:1.0`) |
| Escalado | Más workers de Gunicorn en la misma máquina | Varios contenedores idénticos, en una o varias máquinas |
| Complejidad | Baja, se ve todo en el sistema | Hay que aprender Docker (imágenes, redes, volúmenes) |
| Depuración | Directa (logs, procesos del sistema) | `docker logs`, `docker exec`; una capa más |
| Persistencia | Ficheros en el disco del servidor | Lo que se escribe en el contenedor se pierde al borrarlo; hacen falta **volúmenes** |
| Seguridad | La app corre en el sistema real | Aislada, pero comparte kernel; el grupo `docker` equivale a root |

---

## 3. Contenedor vs. máquina virtual

```
   Máquina virtual (VirtualBox)              Contenedor (Docker)
┌──────────┐ ┌──────────┐              ┌─────────┐ ┌─────────┐
│  App     │ │  App     │              │  App    │ │  App    │
│  Libs    │ │  Libs    │              │  Libs   │ │  Libs   │
│ SO invitado│ │SO invitado│            └─────────┘ └─────────┘
│ (kernel) │ │ (kernel) │              ┌───────────────────────┐
└──────────┘ └──────────┘              │     Docker Engine     │
┌───────────────────────┐              ├───────────────────────┤
│      Hipervisor       │              │  SO anfitrión (kernel)│
├───────────────────────┤              ├───────────────────────┤
│  SO anfitrión / HW    │              │       Hardware        │
└───────────────────────┘              └───────────────────────┘
```

| | Máquina virtual | Contenedor |
|---|---|---|
| Qué virtualiza | El **hardware**: cada VM tiene su propio sistema operativo y kernel | El **sistema operativo**: comparte el kernel del anfitrión |
| Aislamiento | Total (otro kernel) | A nivel de proceso (namespaces y cgroups) |
| Peso | GB (SO completo). Mi Ubuntu Server ocupa varios GB | MB (`python:3.13-slim` ~ 150 MB + la app) |
| Arranque | Minutos (arranca un SO entero) | Segundos (arranca un proceso) |
| Recursos | RAM y CPU reservadas para cada VM | Consume solo lo que usa el proceso |
| Uso típico | Otro SO (Linux en Windows), aislamiento fuerte | Empaquetar y desplegar aplicaciones |

En esta práctica se usan **los dos a la vez**: VirtualBox virtualiza un Ubuntu Server en mi Windows, y dentro de ese Ubuntu, Docker ejecuta el backend como contenedor.

---

## 4. Caso TaskFlow: ¿por qué Docker para el backend y no para el servidor web?

```
Cliente ─HTTPS─▶ Apache (anfitrión :80/:443) ─proxy─▶ 127.0.0.1:8000 ─▶ contenedor taskflow
                 · TLS / certificados                                 (Gunicorn + Uvicorn + FastAPI)
                 · estáticos, logs, cabeceras
```

**Backend en contenedor**, porque:
- Es lo que **cambia constantemente** (nuevas versiones de la API). Cada versión es una imagen etiquetada; desplegar o volver atrás es cambiar de imagen.
- Tiene **dependencias de Python concretas** (Python 3.13, FastAPI, Uvicorn, Gunicorn) que no queremos instalar ni mezclar en el sistema. En la 1.3 tuve que añadir un PPA solo para tener Python 3.13; en Docker viene en la imagen base.
- Se puede **escalar** lanzando más contenedores idénticos detrás del proxy.

**Servidor web (Apache) en el anfitrión**, porque:
- Es la **puerta de entrada** fija: escucha en los puertos 80/443, gestiona los certificados TLS (que se renuevan en el sistema) y casi nunca cambia.
- Se instala y actualiza con `apt` y se gestiona con `systemd`, como un servicio más del sistema operativo; contenerizarlo añade complejidad (volúmenes para certificados y configuración) sin ganar mucho.
- Hace de proxy hacia uno o varios contenedores, y puede seguir sirviendo una página de mantenimiento aunque el contenedor caiga.

Además, el contenedor se publica en el anfitrión solo en local en producción (`-p 127.0.0.1:8000:8000`), por la misma razón que en la 1.3 Gunicorn solo escuchaba en 127.0.0.1: que todo el tráfico entre por Apache.

---

## 5. Criterios de decisión: cuándo usar cada opción

**Instalación directa** si:
- Es una aplicación pequeña, un único servidor y un único proyecto.
- El equipo no conoce Docker o el servidor no permite instalarlo.
- Se necesita el máximo rendimiento o acceso directo a hardware.
- Son servicios del sistema que cambian poco: el servidor web, SSH, el firewall.

**Contenedor** si:
- Hay varios entornos (desarrollo, pruebas, producción) que deben ser idénticos.
- Conviven varias aplicaciones con versiones distintas en el mismo servidor.
- Se despliega a menudo y se quiere *rollback* rápido.
- Se va a desplegar en la nube o a escalar horizontalmente.

**Para TaskFlow:** Apache instalado directamente en el servidor y el backend en Docker.

---

## 6. ¿Qué aporta desplegar en la nube?

La imagen Docker es el **paquete de despliegue**: la misma `taskflowapi:1.0` que funciona en mi VM funciona en cualquier proveedor.

| Aporte | Explicación |
|---|---|
| Sin hardware propio | No hay que comprar ni mantener servidores; se paga por uso (AWS, Azure, Google Cloud) |
| Escalabilidad | Se añaden instancias o contenedores según la carga, incluso automáticamente (*autoscaling*) |
| Alta disponibilidad | Varias zonas/regiones, balanceadores de carga; si una máquina cae, el tráfico va a otra |
| Servicios gestionados | Bases de datos, certificados HTTPS, copias de seguridad y monitorización ya resueltos por el proveedor |
| Despliegue de contenedores | Servicios como AWS ECS/Fargate, Azure Container Apps o Google Cloud Run ejecutan directamente la imagen: `docker push` a un registro y desplegar |
| CI/CD | Integración con GitHub: cada `push` puede construir la imagen y desplegarla |

**Inconvenientes:** coste recurrente que crece con el uso, dependencia del proveedor (*vendor lock-in*), menos control sobre la infraestructura y datos alojados fuera (protección de datos, RGPD).

**Relación con Docker:** contenedor y nube se complementan. Docker resuelve "que funcione igual en todas partes" y la nube resuelve "dónde se ejecuta, cuánto escala y cómo se mantiene disponible".
