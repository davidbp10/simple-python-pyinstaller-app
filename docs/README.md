# Instrucciones del proceso completo

## Primer paso: Desplegar Contenedores dind y Jenkins

### Crear directorio de trabajo.

Lo primero que haremos será crear un directorio donde pondremos main.tf y Dockerfile.

**Main.tf** será una archivo terraform que tendrá la configuración para desplegar las imagenes de dind (Docker in Docker), y Jenkins.

**Dockerfile** lo que hará será configurar una imagen personalizada de Jenkins, en este claso **Jenkins-blueocean**.

### Personalización de jenkins-blueocean con Dockerfile

Lo que hacemos en este paso es crear una imagen personalizada de Jenkins. Para esto ejecutaremos el siguiente comando.

`docker build -t myjenkins-blueocean .`

Con el cual, habríamos creado la imagen personalizada de jenkins, que será la que usaremos posteriormente en el archivo de configuración de Terraform.


### Configurar Terraform.

Cuando estemos en el directorio, ejecutaremos el comando `terraform init`, que guardará la configuración. Esta configuración desplegará el contenedor con la imagen de dind y la imagen de jenkins-blueocean que hemos creado a partir del **Dockerfile** en el puerto 8080.

Luego, tendremos que aplicar la configuración con `terraform apply`, que nos pedirá afirmación.

## Segundo paso: Validar Jenkins

Tendremos que acceder al contenedor de Jenkins para acceder a la ruta `/var/jenkins_home/secrets/initialAdminPassword`, donde encontraremos la contraseña inicial de administrador para poder seguir con la configuración de Jenkins. Después de esto, podremos descargar los plugins y elegir nuestro usuario, nuestra propia contraseña y nuestro correo electrónico. 

Para acceder al contenedor usaremos el comando siguiente:

`docker exec -it jenkins bash`

Quizás debas reiniciar el contenedor de jenkins posteriormente para que se te muestren los plugins:

`docker restart jenkins`