# CI-Jenkins-Docker

<p align="center">
  <img src="./images/image1.png">
</p>

En este proyecto vamos a realizar un despliegue de pipelines CI/CD usando jenkins y ArgoCD, automatizado mediante webhooks de github y jobs creados en Jenkins, se desplegara internamente de forma automatica el helm contenedor de los microservicios y a su vez se realizara monitoreo de estos mediante prometheus y grafana

# Requisitos

Se debe tener instalado

Docker
Kubernetes
helm



# Tabla de Contenido
- [Flujo de aplicación](#flujo-de-aplicación)
- [Jenkins](#jenkins)
  - [Instalación Jenkins](#instalación-jenkins)
  - [Ngrok con Jenkins](#ngrok-con-jenkins)
  - [Configuración Jenkins](#configuración-jenkins)
    - [Plugins](#plugins)
    - [Credenciales](#credenciales)
    - [Jobs](#jobs)
- [GitHub](#github)
  - [Pipelines CI/CD](#pipelines-cicd)
  - [WebHooks](#webhooks)
- [ArgoCD](#argocd)
    - [Instalación ArgoCD](#instalación-argocd)
    - [Configuración ArgoCD](#configuración-argocd)
- [Conclusión](#conclusión)
- [Creditos](#creditos)


# Flujo de aplicación

<p align="center">
  <img src="./images/image21.png">
</p>

Podemos ver el flujo de la aplicacion en el que se comienza realizando un cambio en algun archivo del repositorio de CI, luego esto acciona el webhook asociado a jenkins el cual acciona el pipeline que añade un nuevo versionamiento a la imagen de Docker y la sube a un container registry el cual es Dockerhub, luego se invoca automaticamente el pipeline de CD el cual actualiza este versionamiento en el pod asociado al helm y luego, argoCD verifica dichos cambios y actualiza el servicio con la nueva imagen para añadir dicha version al pod de forma local. 

# Jenkins
### Instalación Jenkins

Para la instalacion de Jenkins se procedera primero a realizar una clonacion del siguiente repo

git clone https://github.com/DanielRamirez1901/CD-Jenkins-Kubernetes.git

Este repo descargara una carpeta llamada JenkinsConfig, en jenkins-ns.yaml cambiaremos el nombre del namespace a nuestro gusto y luego ejecutamos los siguientes comandos
```
cd CD-Jenkins-Kubernetes
cd JenkinsConfig
kubectl apply -f jenkins-ns.yaml
kubectl apply -f jenkins-pv.yaml
kubectl apply -f jenkins-pvc.yaml
helm repo add jenkins https://charts.jenkins.io
helm repo update
helm install jenkins --namespace parcial3 --values values.yaml jenkins/jenkins
```
<p align="center">
  <img src="./images/image2.png">
</p>

Con el comando

    kbc get all -n parcial3
Podremos ver la lista de los servicios asociados a ese namespace el cual en este caso son los de jenkins

<p align="center">
  <img src="./images/image3.png">
</p>

Ahora ejecutamos el comando para hacer accesible el servicio de jenkins localmente

    minikube service jenkins --url -n parcial3

<p align="center">
  <img src="./images/image4.png">
</p>


### Ngrok con Jenkins

Ahora, debemos crear un tunel para exponer jenkins que se encuentra ejecutandose localmente a traves de internet para poder luego configurarlo con los webhooks de github, para ello se puede realizar con ngrok o con serveo, el comando para serveo es el siguiente

ssh -R jenkins-example.serveo.net:80:`<Localhost>`:`<Puerto-del-Jenkins>` serveo.net

- `Localhost:` Se puede reemplazar con la ip asociada al servicio, en este caso 127.0.0.1
- `Puerto-del-Jenkins:` Se reemplaza con el puerto expuesto por el servicio, en este caso 39965

O en este caso, con Ngrok, para ello se procede primero a instalar ngrok
```
wget https://bin.equinox.io/c/bNyj1mQVY4c/ngrok-v3-stable-linux-amd64.tgz
tar -xvzf ngrok-v3-stable-linux-amd64.tgz
sudo mv ngrok /usr/local/bin
```

Ahora, procedemos a ir a la pagina [oficial de registro de ngrok](https://dashboard.ngrok.com/signup)
Iniciamos sesion y copiamos el token para posteriormente usar en el siguiente comando

    ngrok config add-authtoken YOUR_AUTHTOKEN

Y luego creamos el tunel para acceder a jenkins por medio de internet

    ngrok http 127.0.0.1:39965

<p align="center">
  <img src="./images/image5.png">
</p>

### Configuración Jenkins

Ahora que Jenkins es accesible en internet, procedemos a configurarlo para su posterior uso, para obtener la contraseña y acceder al login se usa el siguiente comando

 printf$(kubectl get secret --namespace parcial3 jenkins -o jsonpath="{.data.jenkins-admin-password}"| base64 --decode);echo

#### Plugins

Se deben primero actualizar los plugins necesarios, para ello vamos a

Administrar Jenkins>Plugins>Updates   y actualizamos todo lo que aqui se encuentre

<p align="center">
  <img src="./images/image6.png">
</p>

Y en esa misma pagina, en Available Plugins vamos a buscar los plugins relacionados a Github y relacionados a Docker y luego se reinicia jenkins

<p align="center">
  <img src="./images/image7.png">
</p>


#### Credenciales

Ahora, se agregaran credenciales que se usaran en el pipeline de jenkins para por ejemplo logearse en github y hacer pull o en dockerhub para subir la imagen creada con su nuevo tag

Para ello se accede a 
Panel de control>Administrar Jenkins>Credentials>system>Global credentials>Add credentials

Se deben usar los tokens generados por dockerhub y github ya que estas plataformas ya no permiten el uso de contraseñas para el inicio de sesion por consola, cabe recalcar que el token de github se debe crear permitiendole el uso de webhooks

Para este ejercicio se debe tener como minimo estos dos credentials

<p align="center">
  <img src="./images/image8.png">
</p>


#### Jobs

Ahora creamos las tareas necesarias, en este caso se crearan dos, una para el CI y otra para el CD

CI

Para este caso, creamos una tarea de tipo pipeline

<p align="center">
  <img src="./images/image9.png">
</p>

Habilitamos el trigger github trigger for GITScm polling

<p align="center">
  <img src="./images/image10.png">
</p>

Seleccionamos en el apartado de pipelines para que se use el de Jenkins que se encuentra en el repo

<p align="center">
  <img src="./images/image11.png">
</p>

Guardamos todo este Job

Ahora cremos una nueva tarea, esta se nombrara updatemanifest

<p align="center">
  <img src="./images/image12.png">
</p>

Se selecciona en la configuracion general, esta opcion debe parametrizarse, se añade un parametro de cadena titulado DOCKERTAG con value latest

<p align="center">
  <img src="./images/image13.png">
</p>

Se habilita el trigger como en el Job pasado

<p align="center">
  <img src="./images/image10.png">
</p>

Ahora configuramos el pipeline de la misma forma que el Job anterior pero usando como url de github contenedor del Jenkins, el repo creado para CD

<p align="center">
  <img src="./images/image14.png">
</p>


# GitHub

En este apartado se mostrara el pipeline creado para CI/CD con lo que tendra mas sentido lo configurado anteriormente en Jenkins

### Pipelines CI/CD

CI

En este apartado se usara el repo siguiente que se puede clonar con el comando

git clone https://github.com/DanielRamirez1901/CI-Jenkins-Docker.git

En este repo se encuentra un proyecto de ejemplo, un proyecto n-tier, y se configurara el jenkins en base a dicho proyecto usando un dockerfile como base de imagen

```
podTemplate(label: 'build', containers: [
    containerTemplate(name: 'docker', image: 'docker', command: 'cat', ttyEnabled: true)
  ],
  volumes: [
    hostPathVolume(mountPath: '/var/run/docker.sock', hostPath: '/var/run/docker.sock'),
  ]
) {
  node('build') {
    container('docker') {
  
      def app

      stage('Clone repository') {
          checkout scm
      }

      stage('Build image') {
          app = docker.build("ventana1901/knote")
      }

      stage('Test image') {
          app.inside {
              sh 'echo "Tests passed"'
          }
      }

      stage('Push image') {
          docker.withRegistry('https://registry.hub.docker.com', 'dockerhub') {
              app.push("${env.BUILD_NUMBER}")
          }
      }

      stage('Trigger ManifestUpdate') {
          echo "triggering updatemanifestjob"
          build job: 'updatemanifest', parameters: [string(name: 'DOCKERTAG', value: env.BUILD_NUMBER)]
      }
    }        
  }  
}
```

### Estructura del Pipeline

El pipeline utiliza un pod template para ejecutar contenedores Docker en Jenkins. A continuación se detallan las etapas del pipeline:

### Pod Template

El `podTemplate` define un pod con un contenedor Docker y un volumen montado para Docker socket.

```groovy
podTemplate(label: 'build', containers: [
    containerTemplate(name: 'docker', image: 'docker', command: 'cat', ttyEnabled: true)
  ],
  volumes: [
    hostPathVolume(mountPath: '/var/run/docker.sock', hostPath: '/var/run/docker.sock'),
  ]
)
```

### Node and Container

El pipeline se ejecuta en un nodo etiquetado como `build` y dentro de un contenedor Docker.

```groovy
node('build') {
    container('docker') {
```

### Clone Repository

Esta etapa clona el repositorio especificado en la configuración de Jenkins.

```groovy
stage('Clone repository') {
    checkout scm
}
```

### Build Image

Esta etapa construye la imagen Docker utilizando el Dockerfile presente en el repositorio. La imagen se etiqueta como `ventana1901/knote`.

```groovy
stage('Build image') {
    app = docker.build("ventana1901/knote")
}
```

### Test Image

En esta etapa, la imagen Docker construida se ejecuta y se ejecutan las pruebas dentro del contenedor.

```groovy
stage('Test image') {
    app.inside {
        sh 'echo "Tests passed"'
    }
}
```

### Push Image

La imagen Docker se empuja a Docker Hub. La imagen se etiqueta con el número de build de Jenkins.

```groovy
stage('Push image') {
    docker.withRegistry('https://registry.hub.docker.com', 'dockerhub') {
        app.push("${env

```

### Trigger Manifest Update

Finalmente, se dispara otro trabajo de Jenkins (`updatemanifest`) para actualizar el manifiesto de despliegue con el nuevo tag de la imagen.

```groovy
stage('Trigger ManifestUpdate') {
    echo "triggering updatemanifestjob"
    build job: 'updatemanifest', parameters: [string(name: 'DOCKERTAG', value: env.BUILD_NUMBER)]
}
```

CD

Para este apartado, se usa otro repo que se puede obtener mediante el comando

git clone https://github.com/DanielRamirez1901/CD-Jenkins-Kubernetes.git

En este repo se encuentra el Jenkinsfile asociado al CD

```
node {
    def app

    stage('Clone repository') {
        checkout scm
    }

    stage('Update GIT') {
        script {
            catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                withCredentials([usernamePassword(credentialsId: 'github', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                    sh "git config user.email danirg2222@outlook.es"
                    sh "git config user.name DanielRamirez1901"
                    
                    // Path to the values.yaml in the Helm chart directory
                    def valuesYamlPath = 'helm-project/values.yaml'
                    
                    // Update the image tag in the values.yaml file
                    sh "sed -i 's+tag:.*+tag: ${DOCKERTAG}+g' ${valuesYamlPath}"
                    
                    // Display the updated file for verification
                    sh "cat ${valuesYamlPath}"
                    
                    // Commit and push the changes to the repository
                    sh "git add ${valuesYamlPath}"
                    sh "git commit -m 'Update image tag to ${DOCKERTAG} by Jenkins Job: ${env.BUILD_NUMBER}'"
                    sh "git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/${GIT_USERNAME}/CD-Jenkins-Kubernetes.git HEAD:main"
                }
            }
        }
    }
}
```


---

## Jenkins Pipeline for Updating Helm Chart Values

Este Jenkins Pipeline está diseñado para clonar un repositorio, actualizar el archivo `values.yaml` de un chart de Helm con un nuevo tag de imagen, y luego hacer commit y push de los cambios al repositorio.

### Estructura del Pipeline

El pipeline consta de varias etapas para realizar estas tareas:

### Node

El pipeline se ejecuta en un nodo Jenkins.

```groovy
node {
    def app
```

### Clone Repository

Esta etapa clona el repositorio especificado en la configuración de Jenkins.

```groovy
stage('Clone repository') {
    checkout scm
}
```

### Update GIT

En esta etapa, se actualiza el archivo `values.yaml` en el chart de Helm con el nuevo tag de imagen proporcionado en la variable `DOCKERTAG`.

#### Configuración de Git

Se configura el usuario de Git con el correo electrónico y el nombre proporcionados.

```groovy
sh "git config user.email danirg2222@outlook.es"
sh "git config user.name DanielRamirez1901"
```

#### Ruta del Archivo `values.yaml`

Se define la ruta al archivo `values.yaml` en el directorio del chart de Helm.

```groovy
def valuesYamlPath = 'helm-project/values.yaml'
```

#### Actualización del Tag de la Imagen

Se utiliza `sed` para buscar y reemplazar la línea que contiene el tag de la imagen con el nuevo tag proporcionado en `DOCKERTAG`.

```groovy
sh "sed -i 's+tag:.*+tag: ${DOCKERTAG}+g' ${valuesYamlPath}"
```

#### Verificación de la Actualización

Se muestra el contenido del archivo `values.yaml` actualizado para verificar los cambios.

```groovy
sh "cat ${valuesYamlPath}"
```

#### Commit y Push de los Cambios

Se hacen commit y push de los cambios al repositorio utilizando las credenciales de Git configuradas en Jenkins.

```groovy
sh "git add ${valuesYamlPath}"
sh "git commit -m 'Update image tag to ${DOCKERTAG} by Jenkins Job: ${env.BUILD_NUMBER}'"
sh "git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/${GIT_USERNAME}/CD-Jenkins-Kubernetes.git HEAD:main"
```

### WebHooks

Ahora, para añadir webhooks en el CI para que cualquier cambio, genere un nuevo tag y este se actualice en el deploy, se debe hacer lo siguiente

Vamos a settings del repo CI, luego en webhooks y creamos uno, este webhook se configurade la siguiente manera

<p align="center">
  <img src="./images/image15.png">
</p>


se debe añadir en payload URL /github-webhook, dicha url es la dada por ngrok o serveo

Ahora, si hacemos cualquier cambio en el repo del CI, se ejecutaran los Jobs

<p align="center">
  <img src="./images/image16.png">
</p>

Podemos ver que el CI invoco el Job del CD y ambos se ejecutaron correctamente

<p align="center">
  <img src="./images/image17.png">
</p>

# ArgoCD

Ahora vamos a usar argoCD el cual se conectara al repositorio contenedor del CD e instalara localmente los recursos asociados al helm en nuestro cluster

### Instalación ArgoCD

Se instala el ArgoCD con los siguientes comandos

    kubectl create namespace argocd 
    kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd stable/manifests/install.yaml

Luego, se accede a él con el siguiente comando

    kubectl port-forward svc/argocd-server -n argocd 8080:443

Y para acceder se usa este comando para obtener la contraseña

    kbc -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo 

<p align="center">
  <img src="./images/image18.png">
</p>
    
### Configuración ArgoCD

En argo cd ejecutamos el siguiente .yaml para que cree un proyecto asociado al repositorio de CD y utilice los valores del Helm para levantar el proyecto

se ejecuta el comando

kubectl apply -f argocd.yaml

Luego se da click en sync para sincronizar los servicios de forma local con el argoCD

<p align="center">
  <img src="./images/image19.png">
</p>

Y ahora podemos ver la aplicacion en ejecucion y con su estado de salud asociado a cada microservicio 

<p align="center">
  <img src="./images/image20.png">
</p>

# Conclusión
# Creditos


