# Instrucciones del proceso completo

Participantes: David Brea Piñero y Jesús Alberto Mellado Montes.

## Primer paso: Desplegar Contenedores dind y Jenkins

### 1. Crear directorio de trabajo.

Lo primero que haremos será crear un directorio donde pondremos main.tf y Dockerfile.

- **Main.tf** será una archivo terraform que tendrá la configuración para desplegar las imagenes de dind (Docker in Docker), y Jenkins.

- **Dockerfile** lo que hará será configurar una imagen personalizada de Jenkins, en este claso **Jenkins-blueocean**.

### 2. Personalización de jenkins-blueocean con Dockerfile

Lo que hacemos en este paso es crear una imagen personalizada de Jenkins en un archivo Dockerfile con el siguiente contenido:

```
FROM jenkins/jenkins:2.426.2-jdk17
USER root
RUN apt-get update && apt-get install -y lsb-release
RUN curl -fsSLo /usr/share/keyrings/docker-archive-keyring.asc \
    https://download.docker.com/linux/debian/gpg
RUN echo "deb [arch=$(dpkg --print-architecture) \
    signed-by=/usr/share/keyrings/docker-archive-keyring.asc] \
    https://download.docker.com/linux/debian \
    $(lsb_release -cs) stable" > /etc/apt/sources.list.d/docker.list
RUN apt-get update && apt-get install -y docker-ce-cli
USER jenkins
RUN jenkins-plugin-cli --plugins "blueocean:1.27.9 docker-workflow:572.v950f58993843"
```

Y luego crearemos la imagen de **Jenkins-blueocean** ejecutando el siguiente comando.

`docker build -t myjenkins-blueocean .`

![Construcción Jenkins-blueocean](https://github.com/davidbp10/simple-python-pyinstaller-app/blob/main/docs/img/blueocean.png)

Con el cual, habríamos creado la imagen personalizada de jenkins, que será la que usaremos posteriormente en el archivo de configuración de Terraform.


### 3. Configurar Terraform.

Para la configuración de Terraform en el que desplegaremos Jenkins y Dind, crearemos un archivo **.tf**, en nuestro caso **main.tf**, donde tendremos el siguiente contenido, el especifíca los volúmenes, los puertos y crea una red a dónde se conectarán los contenedores.

```
terraform {
  required_providers {
    docker = {
      source  = "kreuzwerker/docker"
      version = "~> 3.0.1"
    }
  }
}

provider "docker" {}

resource "docker_network" "jenkins" {
  name = "jenkins-network"
}

resource "docker_volume" "jenkins_certs" {
  name = "jenkins-docker-certs"
}

resource "docker_volume" "jenkins_data" {
  name = "jenkins-data"
}

resource "docker_container" "jenkins_docker" {
  name       = "jenkins-docker"
  image      = "docker:dind"
  restart    = "unless-stopped"
  privileged = true
  env = [
    "DOCKER_TLS_CERTDIR=/certs"
  ]

  volumes {
    volume_name    = docker_volume.jenkins_certs.name
    container_path = "/certs/client"
  }

  volumes {
    volume_name    = docker_volume.jenkins_data.name
    container_path = "/var/jenkins_home"
  }

  ports {
    internal = 2376
    external = 2376
  }

  ports {
    internal = 3000
    external = 3000
  }

  ports {
    internal = 5000
    external = 5000
  }

  networks_advanced {
    name    = docker_network.jenkins.name
    aliases = ["docker"]
  }

  command = ["--storage-driver", "overlay2"]
}

resource "docker_container" "jenkins_blueocean" {
  name    = "jenkins-blueocean"
  image   = "myjenkins-blueocean"
  restart = "unless-stopped"
  env = [
    "DOCKER_HOST=tcp://docker:2376",
    "DOCKER_CERT_PATH=/certs/client",
    "DOCKER_TLS_VERIFY=1",
  ]

  volumes {
    volume_name    = docker_volume.jenkins_certs.name
    container_path = "/certs/client"
    read_only      = true
  }

  volumes {
    volume_name    = docker_volume.jenkins_data.name
    container_path = "/var/jenkins_home"
  }

  ports {
    internal = 8080
    external = 8080
  }

  ports {
    internal = 50000
    external = 50000
  }

  networks_advanced {
    name = docker_network.jenkins.name
  }
}
```

Cuando estemos en el directorio, ejecutaremos el comando `terraform init`, que guardará la configuración. Esta configuración desplegará el contenedor con la imagen de dind y la imagen de jenkins-blueocean que hemos creado a partir del **Dockerfile** en el puerto 8080.

Luego, tendremos que aplicar la configuración con `terraform apply`, que nos pedirá afirmación.

## Segundo paso: Validar Jenkins

Tendremos que acceder al contenedor de Jenkins para acceder a la ruta `/var/jenkins_home/secrets/initialAdminPassword`, donde encontraremos la contraseña inicial de administrador para poder seguir con la configuración de Jenkins. Después de esto, podremos descargar los plugins y elegir nuestro usuario, nuestra propia contraseña y nuestro correo electrónico. 

Para acceder al contenedor usaremos el comando siguiente:

`docker exec -it jenkins bash`

Quizás debas reiniciar el contenedor de jenkins posteriormente para que se te muestren los plugins:

`docker restart jenkins`

![Página Principal de Jenkins](https://github.com/davidbp10/simple-python-pyinstaller-app/blob/main/docs/img/ppJenkins.png)

## Tercer paso: Pipeline en Jenkins.

En la página principal de Jenkins, pulsaremos nueva tarea, donde crearemos un pipeline. En la configuración de éste, lo que cambiaremos será la ruta de donde coger el archivo **Jenkinsfile**, que contendrá el pipeline. Seleccionaremos ***Pipeline script from SCM***, con SCM git, y pondremos la ruta de nuestra raíz de nuestro repositorio de github, donde deberá estar el archivo **Jenkinsfile**. Además, tendremos que aclarar que queremos que lo seleccione de la rama ***main***. Finalmente pulsamos en guardar.

![Configuración Pipeline](https://github.com/davidbp10/simple-python-pyinstaller-app/blob/main/docs/img/Ruta_pipeline.png)

El contenido del archivo **Jenkinsfile** será el siguiente:

```
pipeline {
    agent none 
    stages {
        stage('Build') { 
            agent {
                docker {
                    image 'python:3.12.1-alpine3.19' 
                }
            }
            steps {
                sh 'python -m py_compile sources/add2vals.py sources/calc.py' 
                stash(name: 'compiled-results', includes: 'sources/*.py*') 
            }
        }
        stage('Test') {
            agent {
                docker {
                    image 'qnib/pytest'
                }
            }
            steps {
                sh 'py.test --junit-xml test-reports/results.xml sources/test_calc.py'
            }
            post {
                always {
                    junit 'test-reports/results.xml'
                }
            }
        }
        stage('Deliver') {
            agent any
            environment {
                VOLUME = '$(pwd)/sources:/src'
                IMAGE = 'cdrx/pyinstaller-linux:python2'
            }
            steps {
                dir(path: env.BUILD_ID) {
                    unstash(name: 'compiled-results')
                    sh "docker run --rm -v ${VOLUME} ${IMAGE} 'pyinstaller -F add2vals.py'"
                }
            }
            post {
                success {
                    archiveArtifacts "${env.BUILD_ID}/sources/dist/add2vals"
                    sh "docker run --rm -v ${VOLUME} ${IMAGE} 'rm -rf build dist'"
                }
            }
        }
    }
    
}

```

## Cuarto paso: Desplegar y comprobar.

Ya lo que nos queda es desplegar el archivo Jenkins y comprobar que funciona correctamente. Para ello, lo único que tenemos que hacer es pulsar en construir ahora y observar que todo sucede sin problemas y pasa por todas las etapas.

![Pipeline realizado](https://github.com/davidbp10/simple-python-pyinstaller-app/blob/main/docs/img/Pipeline_realizado.png)

Después de hacerlo, y con visión de comprobar aún mejor que todo ha ido como se esperaba, pulsaremos en **Artefactos** y veremos los archivos generados de la ejecución.

![Archivos generados](https://github.com/davidbp10/simple-python-pyinstaller-app/blob/main/docs/img/Archivos_generados.png)