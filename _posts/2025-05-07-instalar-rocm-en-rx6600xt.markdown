## Introducción


OLlama es una plataforma que puede ser desplegada en infraestructura o en la nube que permite cargar modelos grandes de lenguaje sin la necesidad de tener que importar todos los objetos y bibliotecas de Python de forma manual, haciendo más sencillo el uso de modelos de lenguaje de forma local. La problemática en el uso de modelos comerciales mediante sus planes SaaS es que los datos utilizados para la generación, ya sea texto o como archivos quedan disponibles y sin cifrar en los sistemas de archivos de la empresa que provee el LLM (Large Language Model), así como que los planes de cobro son por número de _tokens_, lo cual puede añadirse de forma abrupta según el caso de uso.

El escenario más común es el uso de la plataforma de GPGPU (Cómputo General en Unidades de Procesamiento Gráfico por sus siglas en Inglés) CUDA. Hoy día CUDA representa el estándar, sin embargo esto ata las cargas de trabajo a hacerse de equipo NVidia. Menos conocida está la plataforma ROCm de AMD (Radeon Open Compute), que permiten a usuarios de tarjetas Radeon aprovecharlas para carga de trabajo AMD.

Uno de los retos más grandes con ROCm, es que a pesar que bibliotecas como PyTorch y Tensorflow tienen soporte para ella, el soporte de AMD para con sus tarjetas gráficas es límitado. En mi caso, tengo una tarjeta RX 6600 XT que no cuenta con soporte oficial de AMD para ROCm. Adelante discutiremos cómo utilizar esta tarjeta que cuenta con 8 GB de VRAM para OLlama, se detallará cómo crear y configurar una instalación con Docker y cómo utilizar [Open Web UI](https://openwebui.com/) para tener una experiencia similar a los chatbots de grado consumidor que se han popularizado en los últimos años.

## Preparación

Una de las formas más sencillas es con contenedores de Docker, que será el método a seguir.
Asumiremos la instalación de un sistema operativo Linux con Debian por distribución, en caso que se trate de una distribución distinta, Digital Ocean tiene una buena [guia](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-debian-10) vigente al tiempo de escritura de este artículo.
También requerimos que el plugin  `docker compose` esté instalado. Una instalación de Docker habitual suele incluirlo.

## Instalación

Crearemos primero la siguiente estructura de directorio:

```
|ROOT/
|-- compose.yaml
|-- .env
|-- ollama/
    |-- Dockerfile
    |-- override.conf
```

Creamos el siguiente Dockerfile:
```
Dockerfile
----------
FROM ollama/ollama:rocm
COPY override.conf /etc/systemd/system/ollama.service/override.conf
```

Y llenamos el `override.conf` de la siguiente manera:
```
Enviroment=HSA_OVERRIDE_GFX_VERSION=10.3.0
```
Esto permitirá pasar la sobreescritura al proceso de OLlama en este contenedor de Docker.

## Deployment

Los modelos suelen ser intensivos en memoria (finalmente buscamos alojarlos en la vRAM de la tarjeta de videos), entonces recomiendo crear un lugar en el disco, o en un punto de montaje de un disco dedicado a modelos. En esta guía la ubicación que usaremos será `/opt/ollama/models` . De igual forma buscamos mapear el usuario dentro del contenedor al usuario que gestiona esta ruta. Asumimos que es el usuario con `UID:GID=1000:1000`.
```
mkdir -p /opt/ollama/models
chown -R 1000:1000 /opt/ollama/
```

Creamos entonces la especificación para nuestro contenedor de OLlama:

```yaml
networks:
  common:
services:
###########################################
#                OLLAMA                   #
###########################################
  ollama:
    build:
      ./ollama
		container_name: ollama
    cap_add:
      - SYS_PTRACE
    networks:
      - common
    security_opt:
      - "seccomp=unconfined"
    volumes:
      - /library/models:/root/.ollama
    devices: # Necesitamos pasar los dispositivos de video al contenedor
      - "/dev/kfd:/dev/kfd" 
      - "/dev/dri:/dev/dri"
    group_add:
      - video
    shm_size: "8G"
    environment:
      - HSA_OVERRIDE_GFX_VERSION=10.3.0
    ports:
      - 11434:11434
    restart: unless-stopped
```

Podemos entonces hacer el build de nuestro contenedor:
```bash
docker compose build
docker compose up -d
```

Probamos la instalación con un modelo, en este ejemplo `llama3.2`

```bash
docker exec -it ollama bash
llama run llama3.2:latest
```

Y comenzamos a interactuar:

```bash
root@7f6ad5f203c4:/# ollama run llama3.2:latest
>>> Manda un saludo a los lectores
>>> ¡Hola! ¡Bienvenidos a nuestro espacio de conocimiento!
Quiero extenderte mis más sinceras felicitaciones por haber llegado hasta aquí. Estoy emocionado de poder compartir contigo contenido valioso y útil en este viaje juntos.
Recuerda que la vida es un proceso continuo de aprendizaje y crecimiento, así que no tengas
miedo de explorar nuevos horizontes y desafíos. Aquí encontrarás información actualizada y
relevante para ayudarte a navegar por los retos y oportunidades que te rodean.
Espero poder conectarme contigo en este viaje y compartir mis conocimientos con ti. ¡No dudes en hacerme preguntas o dejar tus comentarios!
¡Saludos de la parte más profunda del corazón!
>>> Send a message (/? for help)
```

Existe un amplio catálogo de modelos en el sitio de [ollama](https://ollama.com/search). En el [repositorio](https://github.com/ollama/ollama/blob/main/docs/import.md) de OLlama existe una guía de cómo empaquetar un modelo propio.

## Open Web UI

Una de las mayores ventajas de usar un modelos autoalojado es poder integrarlo. Las personas han mostrado gran aceptación a ChatGPT de OpenAI. Open Web UI provee una experiencia muy similar. Primero añadimos el manifiesto de Docker Compose que refiere a Open Web UI. La imagen de Docker ya está empaquetada.

```yaml
###########################################
#              OPEN WEBUI                 #
###########################################
  open-webui:
    image: ghcr.io/open-webui/open-webui:main
		container_name: webui
    networks:
      - common
    volumes:
      - open-webui-data:/app/backend/data
    environment:
      - OLLAMA_BASE_URL=http://ollama:11434
      - PDF_EXTRACT_IMAGES=true
    ports:
      - 80:8080
    restart: always
```

Al final nuestro `docker-compose.yaml` quedará de la siguiente forma:

```yaml
networks:
  common:
services:
###########################################
#                OLLAMA                   #
###########################################
  ollama:
    build:
      ./ollama
		container_name: ollama
    cap_add:
      - SYS_PTRACE
    networks:
      - common
    security_opt:
      - "seccomp=unconfined"
    volumes:
      - /library/models:/root/.ollama
    devices: # Necesitamos pasar los dispositivos de video al contenedor
      - "/dev/kfd:/dev/kfd" 
      - "/dev/dri:/dev/dri"
    group_add:
      - video
    shm_size: "8G"
    environment:
      - HSA_OVERRIDE_GFX_VERSION=10.3.0
    ports:
      - 11434:11434
    restart: unless-stopped
		
  open-webui:
    image: ghcr.io/open-webui/open-webui:main
		container_name: webui
    networks:
      - common
    volumes:
      - open-webui-data:/app/backend/data
    environment:
      - OLLAMA_BASE_URL=http://ollama:11434
      - PDF_EXTRACT_IMAGES=true
    ports:
      - 80:8080
    restart: always

volumes:
  - open-webui-data:
		```

Open Web UI estará disponible en `http://localhost:80`, ahí podremos ver una interfaz similar a la siguiente:
![Open Web UI en interfaz móvil](/assets/img/instalar-rocm-en-rx6600xt/img1.png)
Pedirá llenar datos para un usuario administrativo. Una vez realizado se podrá interactuar con los modelos de la misma manera que con ChatGPT.

## Conclusiones

En los últimos días he integrado `gemma3`, un modelo abierto de Google a mi día a día. Una de las ventajas de usar Open Web UI es que cuenta con una API que cumple con el estándar de Open AI, de forma que puedo integrarla con algunas de mis herramientas de uso diario, en particular el plugin de Obsidian [BMO](https://github.com/longy2k/obsidian-bmo-chatbot) (nombrado como el personaje de hora de aventura) me ayuda a salir del bloqueo de escritura.

Al momento de escribir este articulo también me enteré que [PyTorch en ROCm](https://rocm.docs.amd.com/projects/install-on-linux/en/latest/install/3rd-party/pytorch-install.html#docker-image-support) ya está sobre Python 3.12, lo cual nos habla de que hay trabajo activo para las tarjetas gráficas AMD en el espacio del cómputo general con GPU. Finalmente, las tarjetas RX 6XX0 de AMD están pensadas para un público consumidor y con aplicaciones como juegos o edición de imágenes y video en mente, con lo que cargas de trabajo de grado industrial son difíciles de alcanzar. Al final, el objetivo de este articulo es incentivar el uso de hardware que ya se podría tener para aplicaciones de ML e inteligencia artificial.


