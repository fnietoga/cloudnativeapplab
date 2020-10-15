# Cloud Native Application Lab

La finalidad de este laboratorio es la práctica en la creación de una Cloud Native Application, orquestando su proceso de Integración Contínua (CI) y Despliegue Continuo (CD) utilizando las funcionalidad ofrecida por GitHub para la ejecución de workflows denominada `GitHub Actions`.

Se utilizará como base para las prácticas el código fuente de la aplicación de ejemplo creada por Ben Coleman y sobre la que encontrareis mas información en [este repositorio GitHub](https://github.com/benc-uk/dotnet-demoapp). 

Esta aplicación web de ejemplo permite monitorizar en tiempo real los recursos de CPU y memoria utilizados por la aplicación, así como forzar el uso de recursos y lanzar excepciones que serán recogidas por aplicaciones de monitorización, como App Insights.

En el laboratorio empaquetaremos la aplicación web en una imagen [Docker](https://docs.docker.com/), la publicaremos en un [Azure Container Registry](https://docs.microsoft.com/en-us/azure/container-registry/) y lo desplegaremos sobre un cluster de [Azure Kubernetes Service (AKS)](https://docs.microsoft.com/en-us/azure/aks/). 
Toda la infraestructura será desplegada haciendo uso de un fichero de definición de [Terraform](https://www.terraform.io/docs/index.html).

## Requisitos
1. Una suscripción de Azure, de pago por uso, MSDN o Trial.
   - Para completar este laboratorio, asegúrate de que tu cuenta tenga los siguientes roles y permisos:
     - El rol [Owner](https://docs.microsoft.com/en-us/azure/role-based-access-control/built-in-roles#owner) para la suscripción que vayas a utilizar.
     - Ser un usuario [Miembro](https://docs.microsoft.com/en-us/azure/active-directory/fundamentals/users-default-permissions#member-and-guest-users) del directorio Azure AD que vayas a utilizar. (Los usuarios invitados no tendrán los permisos necesarios).
     - No tener restringida la creación de Aplicaciones Azure AD. Es posible que el administrador del Directorio Azure AD lo haya limitado.  

     > **Nota** Si no cumples estos requisitos, es posible que tengas que pedir a otro usuario miembro con derechos de propietario en la suscripción que inicie sesión en el portal y ejecute con antelación el paso de creacion de la aplicación Azure AD.
  
2. Máquina local o una máquina virtual (Windows o Linux) configurada con:
   - Un navegador, preferiblemente Chrome o Microsoft Edge.
   - [dotnet core sdk](https://dotnet.microsoft.com/download) instalado, para las pruebas de compilación y ejecucion locales.
   - [Terraform CLI](https://www.terraform.io/downloads.html). Para las pruebas de ejecución local de los despliegues de infraestructura.
   - [Docker CLI](https://docs.docker.com/get-docker/). Para probar la generación de imagenes docker en local. En Windows es recomendable configurarlo para el uso de Linux containers.
   - [Kubernetes CLI](https://kubernetes.io/docs/tasks/tools/install-kubectl/). Para ejecutar algunos comandos de validación y monitorización sobre el cluster AKS.
   - [Helm](https://helm.sh/docs/intro/install/). Para las pruebas de creación y despliegue en local del Chart para despliegue en kubernetes
   - [Visual Studio Code](https://code.visualstudio.com/download) o cualquier otro IDE o editor, incluso `vim` es válido.
  
>**NOTA** Desde el Portal de Azure existe la posibilidad de abrir una ventana de [Azure Cloud Shell](https://shell.azure.com/) (linux Bash o Powershell) que es totalmente operativa con nuestras suscripciones de Azure y con todas estas herramientas ya preinstaladas.

# Ejercicio 1: Preparar proyecto y prueba local

**Duracion**: 10 minutes

Como primer ejercicio prepararemos un proyecto en GitHub donde versionaremos nuestro código y desde el que realizaremos las tareas de compilación y despliegue.

### Tarea 1: Descargar y probar aplicación web
Para ello empezaremos descargando el codigo fuente de ejemplo desde el repositorio de referencia para este laboratorio, ejecutando el siguiente comando.
```
git clone https://github.com/fnietoga/cloudnativeapplab.git cloudnativelab
```

Eliminaremos las referencias al repositorio Git desde el que hemos descargado el contenido, ya que no utilizaremos este repositorio para subir nuestros cambios en lo sucesivo..
```
Remove-Item -Recurse -Force ./cloudnativelab/.git 
```

> **Nota** En entornos linux el comando a ejecutar sería
> ```
> rm -rf cloudnativelab/.git
> ```

Realizaremos una compilacion de la aplicacion web, y la ejecutaremos de forma local, ejecutando:
```
cd cloudnativelab/src
dotnet restore
dotnet run
```

La aplicación web escuchará en los puertos 5000 (http) y 5001 (https), habituales de Kestrel, pero esto puede cambiarse configurando la variable de entorno `ASPNETCORE_URLS` o con el parámetro `--urls` ([ver docs](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/servers/kestrel?view=aspnetcore-3.1)).  

Probaremos la aplicacion web en el navegador, accediendo a alguna de las siguientes URLs:
```
http://localhost:5000
https://localhost:5001
```

Probaremos la opciones de la aplicación:
- **'Info'** - Mostrará información del sistema y del entorno de ejecución, incluyendo las variables de entorno.
- **'Tools'** - Algunas herramientas útiles en demos, como la carga forzada de CPU, y las páginas de error/excepción para su uso con App Insights u otra herramienta de monitorización.
- **'Monitoring'** - Muestra la carga de la CPU y memoria en tiempo real en formato gráfico, obtenidos utilizando una API REST (/api/monitoringdata)y mostrados utilizando chart.js. Recomendable abrirlo en una pestaña adicional del navegador para poder observar en tiempo real la carga generada utilizando las herramientas anteriores.

### Tarea 2: Crear repositorio en GitHub
A continuación procederemos a crear un repositorio en GitHub y subiremos el codigo fuente de la aplicacion de ejemplo, para lo que realizaremos las siguientes acciones:

1. Accederemos a [GitHub](https://github.com) y nos identificaremos ([Sign in](https://github.com/login)) o registraremos ([Sign up](https://github.com/join)) si es la primera vez que creamos un repositorio.  
    <kbd>![GitHub_Repositories_New](https://user-images.githubusercontent.com/4158659/93994508-413f3200-fd90-11ea-89a7-970458a441e8.png)</kbd>

2. En la parte izquierda se mostrarán nuestros repositorios, y pulsaremos sobre el botón `New` para crear uno nuevo.  
   Le asignaremos el nombre `cloudnativelab`, y dejaremos sin marcar todas las opciones para inicializarlo.  
   Finalmente pulsaremos en el botón `Create repository` 
    <kbd>![GitHub_Repository_Create](https://user-images.githubusercontent.com/4158659/93994522-43a18c00-fd90-11ea-98db-21ff8fc07288.png)</kbd>

3. En nuestro equipo, nos posicionaremos en la carpeta raiz del proyecto descargado (cloudnativelab) 
   ```
   cd ..
   ```

4. Configuraremos nuestro entorno local de git, especificando los datos del usuario que serán incluidos en los cambios realizados.
   ```
   git config --global user.email "you@example.com"
   git config --global user.name "Your Name"
   git config --global credential.helper cache
   ```
   <kbd>![Git_Config](https://user-images.githubusercontent.com/4158659/93996915-1dc9b680-fd93-11ea-8008-92d8448d646e.png)</kbd>

5. Inicializaremos un repositorio git local
   ```
   git init
   ```

6. Añadiremos a nuestro repositorio local todo el contenido de nuestra carpeta actual y subcarpetas
   ```
   git add .
   git commit -m "Initial Commit"
   ```

7. Configuraremos el repositorio remoto de git, apuntando al repositorio de GitHub creado
    ```
    git branch -M master
    git remote add origin https://github.com/[GitHubUserName]/cloudnativelab.git
    ```
    > **Nota** Atención a sustituir el valor entre corchetes con nuestro usuario en GitHub. También podremos copiar la url del repositorio, o el comando completo, desde la pantalla que se nos muestra tras crear el repositorio.
    <kbd>![Git_RemoteAdd](https://user-images.githubusercontent.com/4158659/93996968-2b7f3c00-fd93-11ea-9c4f-7f16c2cb3a26.png)</kbd>

8. Subiremos todos los cambios al repositorio remoto
   ```
   git push -u origin master
   ```
9. Por ultimo, comprobaremos a través del navegador que los ficheros se han subido correctamente a nuestro repositorio en GitHub.
   <kbd>![GitHub_FilesPushed](https://user-images.githubusercontent.com/4158659/93997264-7ef18a00-fd93-11ea-8286-9c31addf92db.png)</kbd>

# Ejercicio 2: Despliegue de infraestructura con Terraform

**Duracion**: 45 minutes

En este ejercicio desplegaremos toda la infraestructura Azure necesaria.  
Desplegaremos los siguientes elementos:
- [Azure Container Registry](https://docs.microsoft.com/en-us/azure/container-registry/). Nos permitirá contar con un repositorio privado para las imágenes generadas.
- [Azure Kubernetes Service](https://docs.microsoft.com/en-us/azure/aks/). Entorno de ejecución basado en el orquestador [kubernetes](https://kubernetes.io/docs/home/) sobre el que ejecutaremos nuestra aplicación incluida en un contenedor.

### Tarea 1: Preparar contexto para Terraform
Antes de empezar con el despliegue de infraestructura hay que hacer un par de preparativos para el uso de Terraform.
1. Se requiere crear un [Service Principal](https://www.terraform.io/docs/providers/azurerm/guides/service_principal_client_secret.html) con permisos para crear recursos en la suscripción de Azure, que será utilizado para la autenticacion de Terraform al acceder al cloud.  
   Desde una ventana de PowerShell, ejecutaremos los siguientes comandos para autentificarnos y seleccionar la suscripción deseada:
   ```
   az login (Finalizaremos el proceso de autenticación en el navegador web)
   az account list (para visualizar las suscripciones a las que tenemos acceso)
   az account set --subscription [GUID] (para seleccionar utilizando el subscriptionId)
   az account show (para confirmar la suscripción actualmente seleccionada)
   ```

   A continuación, los comandos de creación
   ```
   $SUBSCRIPTION_ID=$(az account show --query id -o tsv)
   $TENANT_ID=$(az account show --query tenantId -o tsv)

   #Create the Service Principal
   $SP=$(az ad sp create-for-rbac --sdk-auth --name cloudnativelab --role="Contributor" --scopes="/subscriptions/$SUBSCRIPTION_ID")
   
   $CLIENT_ID=($SP | ConvertFrom-Json).clientId
   $CLIENT_PASSWORD=($SP | ConvertFrom-Json).clientSecret

   #display values   
   echo "ARM_TENANT_ID: $TENANT_ID"  
   echo "ARM_SUBSCRIPTION_ID: $SUBSCRIPTION_ID"
   echo "ARM_CLIENT_ID: $CLIENT_ID"
   echo "ARM_CLIENT_SECRET: $CLIENT_PASSWORD" 
   echo "AZURE_CREDENTIALS: $SP"

   #export some values as environment variables
   $Env:ARM_TENANT_ID=$TENANT_ID
   $Env:ARM_SUBSCRIPTION_ID=$SUBSCRIPTION_ID
   $Env:ARM_CLIENT_ID=$CLIENT_ID
   $Env:ARM_CLIENT_SECRET=$CLIENT_PASSWORD   
   ```

   > **Nota** Los comandos a ejecutar desde una consola linux, o desde el shell del Portal de Azure, serían los siguientes:
   ```
   SUBSCRIPTION_ID=$(az account show --query id -o tsv)
   TENANT_ID=$(az account show --query tenantId -o tsv)

   #Create the Service Principal
   SP=$(az ad sp create-for-rbac --sdk-auth --name cloudnativelab --role="Contributor" --scopes="/subscriptions/$SUBSCRIPTION_ID")
   
   CLIENT_ID=$(echo $SP | jq -r ".clientId")
   CLIENT_PASSWORD=$(echo $SP | jq -r ".clientSecret")

   #display values
   echo "ARM_TENANT_ID: $TENANT_ID"  
   echo "ARM_SUBSCRIPTION_ID: $SUBSCRIPTION_ID"
   echo "ARM_CLIENT_ID: $CLIENT_ID"
   echo "ARM_CLIENT_SECRET: $CLIENT_PASSWORD"
   echo "AZURE_CREDENTIALS: $SP"

   #export some values as environment variables
   export ARM_TENANT_ID=$TENANT_ID
   export ARM_SUBSCRIPTION_ID=$SUBSCRIPTION_ID
   export ARM_CLIENT_ID=$CLIENT_ID
   export ARM_CLIENT_SECRET=$CLIENT_PASSWORD
   ``` 
   > **Importante** Anotar y conservar los valores de salida mostrados para utilizarlos mas adelante, incluido la cadena JSON con nombre de parámetro `AZURE_CREDENTIALS`.  

2. Para permitir la colaboración y el despliegue desde distintos origenes con Terraform se recomienda el uso de un [Backend](https://www.terraform.io/docs/backends/index.html) en el que almacenar el fichero que utiliza Terraform para almacenar el estado de los recursos desplegados. Para ello utilizaremos un [Storage Account de Azure](https://www.terraform.io/docs/backends/types/azurerm.html), que debemos crear previamente, ya que no podemos incluir su despliegue en el mismo fichero que lo utilizará.  
   Desde la ventana de PowerShell con la sesion en Azure ya iniciada, ejecutaremos los siguientes comandos: 
   ```
   $RANDOM=Get-Random -Maximum 9999
   $RESOURCE_GROUP_NAME="tstate"
   $STORAGE_ACCOUNT_NAME="tstatecnl$RANDOM"
   $CONTAINER_NAME="cloudnativelab"


   # Create resource group
   az group create --name $RESOURCE_GROUP_NAME --location westeurope

   # Create storage account
   az storage account create --resource-group $RESOURCE_GROUP_NAME --name $STORAGE_ACCOUNT_NAME --sku Standard_LRS --encryption-services blob

   # Get storage account key
   $ACCOUNT_KEY=$(az storage account keys list --resource-group $RESOURCE_GROUP_NAME --account-name $STORAGE_ACCOUNT_NAME --query [0].value -o tsv)

   # Create blob container
   az storage container create --name $CONTAINER_NAME --account-name $STORAGE_ACCOUNT_NAME --account-key $ACCOUNT_KEY

   #display values
   echo "resource_group_name: $RESOURCE_GROUP_NAME"
   echo "storage_account_name: $STORAGE_ACCOUNT_NAME"
   echo "container_name: $CONTAINER_NAME"
   echo "ARM_ACCESS_KEY: $ACCOUNT_KEY"

   #export some values as environment variables
   $Env:ARM_ACCESS_KEY=$ACCOUNT_KEY 

   ```

   > **Nota** Los comandos a ejecutar desde una consola linux, o desde el shell del Portal de Azure, serían los siguientes:
   ```
   RESOURCE_GROUP_NAME=tstate
   STORAGE_ACCOUNT_NAME=tstatecnl$RANDOM
   CONTAINER_NAME=cloudnativelab


   # Create resource group
   az group create --name $RESOURCE_GROUP_NAME --location westeurope

   # Create storage account
   az storage account create --resource-group $RESOURCE_GROUP_NAME --name $STORAGE_ACCOUNT_NAME --sku Standard_LRS --encryption-services blob

   # Get storage account key
   ACCOUNT_KEY=$(az storage account keys list --resource-group $RESOURCE_GROUP_NAME --account-name $STORAGE_ACCOUNT_NAME --query [0].value -o tsv)

   # Create blob container
   az storage container create --name $CONTAINER_NAME --account-name $STORAGE_ACCOUNT_NAME --account-key $ACCOUNT_KEY

   #display values
   echo "resource_group_name: $RESOURCE_GROUP_NAME"
   echo "storage_account_name: $STORAGE_ACCOUNT_NAME"
   echo "container_name: $CONTAINER_NAME"
   echo "ARM_ACCESS_KEY: $ACCOUNT_KEY"

   #export some values as environment variables
   export ARM_ACCESS_KEY=$ACCOUNT_KEY
   ```
   > **Importante** Anotar y conservar los valores de salida mostrados para utilizarlos mas adelante.  
> **Observación** En los dos pasos anteriores se han exportado como variables de entorno los valores necesarios para la autenticación por parte de Terraform al conectarse a Azure y al Storage Account, para permitir las pruebas locales sin tener que codificarlos de forma estática en el fichero de Terraform. Cuando realicemos el despliegue desde una `GitHub Action` habrá que configurar de forma similar esos valores.  

### Tarea 2: Fichero Terraform
Crearemos un fichero que contendrá la definición de la infraestructura a desplegar en la notación admitida por Terraform

1. Crearemos un fichero denominado `main.tf` en la raiz de nuestro proyecto. 
   El contenido del fichero será el siguiente (reeemplazando el nombre de `storage_account_name` con el valor de salida anterior):
   ```javascript
   # providers
   provider "azurerm" {
      features {}
   }

   # terraform backend in azure storage account
   terraform {
      backend "azurerm" {
         resource_group_name  = "tstate"
         storage_account_name = "tstatecnl[REPLACE]"
         container_name       = "cloudnativelab"
         key                  = "terraform.tfstate"
      }
   }

   #deployment
   resource "azurerm_resource_group" "cloudnativelab" {
      name     = "cloudnativelab"
      location = "West Europe"
   }

   resource "azurerm_container_registry" "cloudnativelab" {
      name                = "cnlacr"
      resource_group_name = azurerm_resource_group.cloudnativelab.name
      location            = azurerm_resource_group.cloudnativelab.location
      sku                 = "Standard"
      admin_enabled       = true
   }

   resource "azurerm_kubernetes_cluster" "cloudnativelab" {
      name                = "cnlaks"
      location            = azurerm_resource_group.cloudnativelab.location
      resource_group_name = azurerm_resource_group.cloudnativelab.name
      dns_prefix          = "cloudnativelab"
      node_resource_group = "cloudnativelab_cnlaks_resources"

      default_node_pool {
         name       = "default"
         node_count = 2
         vm_size    = "Standard_D2_v2" 
      }

      identity {
         type = "SystemAssigned"
      }
      role_based_access_control {
         enabled = true
      } 
   }

   #output values
   output "ACR_NAME" {
      value = azurerm_container_registry.cloudnativelab.login_server
   }
   output "ACR_USERNAME" {
      value = azurerm_container_registry.cloudnativelab.admin_username
   }
   output "ACR_PASSWORD" {
      value = azurerm_container_registry.cloudnativelab.admin_password
   }
   ```

   Analicemos el contenido del fichero.
   >. En el primer apartado definimos los providers que vamos a utilizar, en este caso unicamente especificaremos el de `azurerm`. No incluiremos valores de configuración ya que han sido configurados como variables de entorno anteriormente.  
   <kbd>![terraform_1](https://user-images.githubusercontent.com/4158659/94116312-2a114a80-fe4b-11ea-9172-ee8828fd3acf.png)</kbd>

   >. A continuación definimos el backend utilizado para almacenar el estado de los servicios desplegados, indicando los valores del storage account creado.  
   **OJO** Tendremos que modificar el nombre del recurso Storage Account, ya que una parte de su nombre de compone con un valor numérico aleatorio  
   <kbd>![terraform_2](https://user-images.githubusercontent.com/4158659/94116322-2d0c3b00-fe4b-11ea-9af7-a4988d364526.png)</kbd>

   >. En el apartado principal del fichero se describen los distintos recursos a desplegar, asignando un "alias" a cada uno de ellos y sus propiedades.  
   <kbd>![terraform 3](https://user-images.githubusercontent.com/4158659/94402667-9ef6c400-016c-11eb-8272-fb3618d9e498.png)</kbd>

   >. Por ultimo definimos los valores de salida, que podrán ser capturados tras el despliegue para su uso posterior.  
   <kbd>![terraform 4](https://user-images.githubusercontent.com/4158659/94402669-9f8f5a80-016c-11eb-9b98-940d756ecbf1.png)</kbd>

2. En la ventana de Powershell o linea de comandos anterior, probaremos a inicializar el contexto local de Terraform, ejecutando
   ```
   terraform init
   ``` 
   <kbd>![terraform init](https://user-images.githubusercontent.com/4158659/95560828-199bcb00-0a1a-11eb-96a1-dcc45616cb61.png)</kbd>

3. Probaremos a crear el plan de ejecucion de Terraform, ejecutando:
   ```
   terraform plan
   ```
   <kbd>![terraform plan](https://user-images.githubusercontent.com/4158659/95560829-1a346180-0a1a-11eb-930b-2d844c7cfd40.png)</kbd>

4. Realizaremos el despliegue, ejecutando
   ```
   terraform apply --auto-approve
   ```   

5. Transcurridos unos minutos y tras terminar el despliegue sin errores de ejecución, anotaremos los valores de salida
   <kbd>![terraform outputs](https://user-images.githubusercontent.com/4158659/95560830-1accf800-0a1a-11eb-98d3-2378847b4ca1.png)</kbd>
   
6. revisaremos a través del portal de Azure que se han creado los recursos correctamente
   <kbd>![PortalAzure_ Recursos](https://user-images.githubusercontent.com/4158659/94403073-2b08eb80-016d-11eb-839c-058e80b4774e.png)</kbd>

7. Procederemos a eliminar los recursos del despliegue, ya que solamente se trataba de una prueba. Ejecutaremos:
   ```
   terraform destroy --auto-approve
   ```

8. Tras confirmar que el fichero de definición Terraform es correcto, lo subiremos al repositorio de código en `GitHub`
   ```
   git add .
   git commit -m "Added terraform file"
   git push
   ```

### Tarea 3: Desplegar desde GitHub Action
En esta tarea preparemos el despliegue de la infraestructura para que se realice de forma automática desde GitHub Actions cada vez que se realice un cambio en el repositorio de la aplicación, como parte del proceso de integración y despliegue continuo.

1. Crearemos la carpeta para contener los ficheros de definición de workflows de GitHub, y la primera definicion de workflow.
   ```
   mkdir ./.github/workflows
   code ./.github/workflows/cloudnativelab.yml
   ```
   <kbd>![powershell_CreateWorkflow](https://user-images.githubusercontent.com/4158659/94408134-ac17b100-0174-11eb-81ce-4ff2e4227409.png)</kbd>

   >**Nota** Si se está utilizando otro editor, como `vim`, el comando equivalente sería el siguiente
   ```
   vim ./.github/workflows/cloudnativelab.yml
   ```


2. El contenido del fichero denominado `cloudnativelab.yml` es el  siguiente
   ```yaml
   name: cloudnativelab

   # Controls when the action will run. 
   # Triggers the workflow on push or pull request events,
   # but only for the master branch
   on:
      push:
         branches: [ master ]
      pull_request:
         branches: [ master ]

   # Configure workflow to also support triggering manually
      workflow_dispatch:
         inputs:
            logLevel:
               description: 'Log level'
               required: true
               default: 'warning'
         
   # Jobs define the actions that take place when code is pushed to the master branch
   jobs:
      deploy-infra-with-terraform:
         name: Deploy infrastructure with Terraform
         runs-on: ubuntu-latest
         env:
            ARM_ACCESS_KEY: ${{ secrets.ARM_ACCESS_KEY }}
            ARM_CLIENT_ID: ${{ secrets.ARM_CLIENT_ID }}
            ARM_CLIENT_SECRET: ${{ secrets.ARM_CLIENT_SECRET }}
            ARM_SUBSCRIPTION_ID: ${{ secrets.ARM_SUBSCRIPTION_ID }}
            ARM_TENANT_ID: ${{ secrets.ARM_TENANT_ID }}

         steps:
         # Checkout the repo
         - uses: actions/checkout@v2
            
         # Run Terraform commands
         - name: Terraform Init
           id: init
           run: terraform init

         - name: Terraform Plan
           id: plan
           run: terraform plan

         - name: Terraform Apply
           id: apply
           run: terraform apply -auto-approve
   ```
   >**ATENCION** al copiar y pegar es posible que no se respete la identación del documento. Al tratarse de un documento YAML es muy importante que los espacios al principio de cada linea queden exactamente como se muestran

   Analicemos el contenido del fichero.
   >. En la primera sección definiremos el nombre para el workflow y que se lance automáticamente tras una subida de código, o tras finalizar un pull request, en la rama master.  
   <kbd>![workflow_1](https://user-images.githubusercontent.com/4158659/94411607-0f0b4700-0179-11eb-8824-0520591c12a5.png)</kbd>  
  
   >. El siguiente bloque es para especificar valores por defecto que permitan la ejecución manual del workflow desde el portal de GitHub. Mas información en [este enlace](https://github.blog/changelog/2020-07-06-github-actions-manual-triggers-with-workflow_dispatch/)   
   <kbd>![workflow_2](https://user-images.githubusercontent.com/4158659/94411610-0fa3dd80-0179-11eb-8609-6c3626b1e2e3.png)</kbd>  
  
   >. Por ultimo definiremos un job que realizará el despliegue, definiendo algunas variables de entorno para configurar el entorno de ejecucion de `terraform` y ejecutando las operaciones de despliegue utilizando el `terraform CLI`, de igual forma que hicimos en las pruebas locales.  
   <kbd>![workflow_3](https://user-images.githubusercontent.com/4158659/94411611-0fa3dd80-0179-11eb-8ed7-a1085c0e64f1.png)</kbd>   


3. En el proyecto GitHub crearemos los secretos utilizados en el workflow, en el apartado Settings -> Secrets, con los valores que anotamos durante el despliegue del `Service Principal` y el `Storage Account`.
<kbd>![GitHub_Secrets1](https://user-images.githubusercontent.com/4158659/94407800-390e3a80-0174-11eb-8c33-b4f919f85ae1.png)</kbd>

Deben quedar correctamente configurados los siguientes secretos:
- ARM_ACCESS_KEY
- ARM_CLIENT_ID
- ARM_CLIENT_SECRET
- ARM_SUBSCRIPTION_ID
- ARM_TENANT_ID
<kbd>![GitHub_Secrets2](https://user-images.githubusercontent.com/4158659/94408424-10d30b80-0175-11eb-9113-ac21ea9ec143.png)</kbd>

4. Subiremos nuestros cambios al repositorio de código.
   ```
   git add .
   git commit -m "Added terraform file"
   git push
   ```

5. A través del portal de GitHub observaremos que se arranca nuestro workflow de despliegue, en la opción `Actions` de nuestro proyecto. Esperaremos a que termine su ejecución. Revisando los logs observaremos mensajes de ejecución similares a los que observamos en la prueba local.
   <kbd>![GitHub_ActionCompleted](https://user-images.githubusercontent.com/4158659/94411206-97d5b300-0178-11eb-956a-6ad642a98e42.png)</kbd>

6. Revisaremos de nuevo a través del Portal de Azure que se han desplegado los recursos correctamente.
   <kbd>![PortalAzure_ResourcesDeployed](https://user-images.githubusercontent.com/4158659/94411214-9a380d00-0178-11eb-98a0-3f19479b886a.png)</kbd>


# Ejercicio 3: Compilación y creación de imagen Docker

**Duracion**: 15 minutes

En este ejercicio preparemos nuestra aplicación para ser empaquetada en una imagen Docker, procediendo posteriormente a automatizar las tareas de creación de la imagen y su publicación en [Azure Container Registry](https://docs.microsoft.com/en-us/azure/container-registry/) desde [GitHub Actions](https://docs.github.com/en/actions).  

### Tarea 1: Crear imagen y ejecutar en local
1. El primer paso es crear un fichero `Dockerfile` que definirá las acciones necesarias para crear la imagen de nuestra aplicación, lo crearemos en la carpeta raiz de nuestro proyecto y con el siguiente contenido
   ```
   # =======================================================
   # Stage 1 - Build/compile app using container
   # =======================================================

   # Build image has SDK and tools (Linux)
   FROM mcr.microsoft.com/dotnet/core/sdk:3.1-alpine as build
   WORKDIR /build

   # Copy project source files
   COPY src ./src

   # Restore, build & publish
   WORKDIR /build/src
   RUN dotnet restore
   RUN dotnet publish --no-restore --configuration Release

   # =======================================================
   # Stage 2 - Assemble runtime image from previous stage
   # =======================================================

   # Base image is .NET Core runtime only (Linux)
   FROM mcr.microsoft.com/dotnet/core/aspnet:3.1-alpine
   
   # Seems as good a place as any
   WORKDIR /app

   # Copy already published binaries (from build stage image)
   COPY --from=build /build/src/bin/Release/netcoreapp3.1/publish/ .

   # Expose port 5000 from Kestrel webserver
   EXPOSE 5000

   # Tell Kestrel to listen on port 5000 and serve plain HTTP
   ENV ASPNETCORE_URLS http://*:5000

   # Run the ASP.NET Core app
   ENTRYPOINT dotnet dotnet-demoapp.dll
   ```

   Analicemos el contenido del fichero.
   >. En la primera parte se toma como base la imagen de `dotnet core` basada en alpine con el SDK instalado. Es algo mas pesada pero es ideal para las labores de compilación, al tener ya instaladas la mayor parte de las dependencias y frameworks necesarios.  
      <kbd>![DockerFile_1](https://user-images.githubusercontent.com/4158659/94113292-f92f1680-fe46-11ea-8f3b-03caf28724e7.png)</kbd>

   >. Sobre esa imagen se copia el contenido de nuestro proyecto y se realizan las tareas `dotnet restore` y `dotnet publish`, dejando el contenido de la aplicación compilada y lista para su publicación en una carpeta.  
      <kbd>![Dockerfile_2](https://user-images.githubusercontent.com/4158659/94113300-fb917080-fe46-11ea-999b-63d04f6a110f.png)</kbd>

   >. Sobre una segunda imagen `dotnet core` basada también en alpine, pero algo mas ligera y especialmente pensada para la ejecución de aplicaciones, se copia todo el contenido publicado desde la imagen anterior.  
      <kbd>![Dockerfile_3](https://user-images.githubusercontent.com/4158659/94113305-fd5b3400-fe46-11ea-8143-fd687a44aeba.png)</kbd>

   >. Se realizan los ultimos preparativos, como exponer el puerto que utiliza la aplicacion web y definir el punto de ejecución de nuestra aplicación.  
      <kbd>![Dockerfile_4](https://user-images.githubusercontent.com/4158659/94113311-ff24f780-fe46-11ea-8ca2-17032e585b43.png)</kbd>

2. Procederemos a crear la imagen docker ejecutando el comando
   ```
   docker image build -t cloudnativelab .
   ```
   <kbd>![Docker_ImageBuild](https://user-images.githubusercontent.com/4158659/94113561-60e56180-fe47-11ea-8a55-64ec3600f09b.png)
</kbd>

3. Revisaremos las imágenes docker locales
   ```
   docker image ls 
   ```
   Podremos observar las dos imágenes dotnet core descargadas y la diferencia de tamaño entre ambas, la imagen `cloudnativelab` generada con un tamaño de 118Mb y una imagen temporal sin nombre, que se corresponde con la utilizada para el proceso de compilación, de mayor tamaño.  
   <kbd>![Docker_imageList](https://user-images.githubusercontent.com/4158659/94113595-6e9ae700-fe47-11ea-8029-1adfc02aaa06.png)</kbd>
   
4. Pondremos en ejecución un contenedor a partir de la imagen recien creada
   ```
   docker container run --name demoapp -p 5000:5000 -d cloudnativelab
   ```

5. Revisaremos los logs del contenedor en ejecución para ver que todo ha ido bién.
   ```
   docker container logs demoapp
   ```

6. y probaremos desde el navegador que la aplicación es accesible correctamente. 
   Al acceder al apartado de `Info` nos indicará que se está ejecutando desde un contenedor Docker, y que el sistema Operativo es Linux. 
   <kbd>![Demoapp_RunningContainer](https://user-images.githubusercontent.com/4158659/94113643-7fe3f380-fe47-11ea-9fb9-0a61ba45c902.png)</kbd>

8. Por ultimo, detendremos y eliminaremos el contenedor creado para la prueba.
   ```
   docker container stop demoapp
   docker container rm demoapp
   ``` 
 
### Tarea 2: Ejecutar desde GitHub Action
En esta parte del ejercicio editaremos el workflow de GitHub creamos anteriormente para incorporar las acciones de creación de la imagen docker, de igual forma a como lo hemos realizado localmente, y su posterior publicación en el Container Registry desplegado.

1. Editaremos el fichero ./.github/workflows/cloudnativelab.yml para añadir al final el siguiente contenido 
   ```yaml
   build-and-publish-docker-image:
      needs: deploy-infra-with-terraform
      name: Build and Push Docker Image
      runs-on: ubuntu-latest
      env:
         ACR_NAME: cnlacr.azurecr.io
         IMAGE_NAME: cloudnativelab
         IMAGE_VERSION: ${{ github.run_id }}

      steps:
      # Checkout the repo
      - uses: actions/checkout@v2 
         
      # Login to ACR
      - name: Login to Azure container Registry
        uses: docker/login-action@v1 
        with:
         registry: ${{ env.ACR_NAME }}
         username: ${{ secrets.ACR_USERNAME }}
         password: ${{ secrets.ACR_PASSWORD }}

      # Build & push image
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v1

      - name: Build and push an image to container registry
        uses: docker/build-push-action@v2
        with:
         push: true
         context: .
         file: ./Dockerfile
         tags: |
            ${{ env.ACR_NAME }}/${{ env.IMAGE_NAME }}:latest
            ${{ env.ACR_NAME }}/${{ env.IMAGE_NAME }}:${{ env.IMAGE_VERSION }}
   ```
   > **Importante** La definición del nuevo job `build-and-publish-docker-image` debe quedar alineada en la misma columna con el job ya existente. 

2. Debemos crear dos nuevos secretos en el proyecto de GitHub, para la conexión con el Azure Container Registry. Los valores los encontraremos en los valores de salida del despliegue en los logs del workflow, o bien los relativos al ACR los podemos consultar directamente a través del Portal Azure.
   - ACR_USERNAME
   - ACR_PASSWORD
   - AZURE_CREDENTIALS
  <kbd>![GitHub_Secrets_2](https://user-images.githubusercontent.com/4158659/95969415-f9e81680-0e0e-11eb-8d9f-57074990aa4d.png)</kbd>

3. Subiremos los cambios al repositorio de código
   ```
   git add .
   git commit -m "Added docker build to workflow"
   git push
   ```

4. Revisaremos que el workflow se ejecuta automáticamente, y que concluye de forma correcta.
   <kbd>![GitHub_workflow](https://user-images.githubusercontent.com/4158659/94416098-b2128f80-017e-11eb-9b9b-0a21676071be.png)</kbd>

5. Comprobaremos que la imagen ha quedado desplegada en el Azure Container Registry, a través del Portal Azure, en el apartado `Repositories` del recurso `cnlacr`.
   <kbd>![Portal_Acr_Imagen](https://user-images.githubusercontent.com/4158659/94416102-b2ab2600-017e-11eb-940b-dfd88ab11c82.png)</kbd>

# Ejercicio 4: Despliegue en AKS con Helm Chart

**Duracion**: 20 minutes

### Tarea 1: Configurar cluster AKS

Para poder realizar las tareas de despliegue y utilizar la herramienta de linea de comandos de kubernetes `kubectl` tenemos que configurar nuestro entorno para que pueda conectarse al cluster AKS desplegado.

Para ello realizaremos las siguientes acciones:
1. Configurar en nuestro contexto de ejecucion local de kubernetes la informacion del cluster AKS `cnlaks` desplegado
   ```powershell
   az aks get-credentials --name cnlaks --resource-group cloudnativelab
   ```

2. Verificar que actualmente tenemos seleccionado el contexto de cluster de kubernetes `cnlaks`, ejecutando:
   ```powershell
   kubectl config current-context
   ``` 

3. Probaremos la correcta conexión con el cluster ejecutando algunos comandos adicionales de `kubectl` 
   ```powershell
   kubectl get nodes -o wide
   kubectl get pods,services --all-namespaces
   ```

4. Configuraremos el cluster de AKS `cnlaks` para que pueda descargarse de forma autóinoma y sin tener que especificar autenticacion adicional, las imagenes existentes en el Container Registry `cnlacr`, ejecutando el comando:
   ```powershell
    az aks update -n cnlaks -g cloudnativelab --attach-acr cnlacr
   ```
   
### Tarea 2: Creación del Helm Chart

En esta tarea crearemos y probaremos en local un chart de Helm, con el que realizaremos el despliegue al cluster de Kubernetes.
El chart de Helm se caracteriza por diferenciar en su implementación el fichero de definición de los servicios Kubernetes, incluidos en una template y los valores de configuración que son fusionados con las plantillas para la generación del fichero de definición resultante.

Para ello realizaremos las siguientes acciones:

1. Sobre el directorio raiz de nuestro proyecto, en una ventana Powershell o Bash Shell, usaremos el comando `helm create` para crear una plantilla de chart que despues podremos empaquetar y utilizar para el despliegue. Usaremos los siguientes comandos para crear un nuevo chart llamado `demoapp` en un nuevo directorio:
   ```powershell
   mkdir charts
   cd charts
   helm create demoapp
   ```

2. Ahora tenemos que actualizar el conjunto de plantillas y ficheros generados para que coincida con nuestros requisitos.  Primero actualizaremos el archivo llamado `values.yaml`
   ```powershell
   cd demoapp
   code values.yaml
   ```

3. Busca la definicion de `image`  y actualiza los valores para que coincidan con lo siguiente:
   ``` yaml
   image:
     repository: cnlacr.azurecr.io/cloudnativelab
     pullPolicy: Always
   ```

4. Busca las propiedades `nameOverride` and `fullnameOverride` y actualiza los valores para que coincidan con los siguientes:
   ```yaml
   nameOverride: "demoapp"
   fullnameOverride: "demoapp"
   ```

5. Busca la definicion de `service` y actualiza los valores para que coincidan con los siguentes:
   ```yaml
   service:
     type: LoadBalancer
     port: 80
   ```

6. Busca la definición de `resources` y actualiza los valores para que coincidan con los siguentes:
   ```yaml
   resources:
     # We usually recommend not to specify default resources and to leave this as a conscious
     # choice for the user. This also increases chances charts run on environments with little
     # resources, such as Minikube. If you do want to specify resources, uncomment the following
     # lines, adjust them as necessary, and remove the curly braces after 'resources:'.
     limits:
       cpu: 300m
       memory: 256Mi
     requests:
       cpu: 100m
       memory: 100Mi
   ```
   > **NOTE** Recuerda eliminar las llaves `{}` existentes despues de la definición `resources:`. 

7. Guarda los cambios y cierra el editor.

8. Ahora actualizaremos el archivo llamado `Chart.yaml`.
   ```powershell
   code Chart.yaml
   ```

9. Busca la propiedad `appVersion` y actualiza su valor para que coincidan con el siguente:
    ```yaml
    appVersion: latest
    ```

10. Ahora actualizaremos el archivo llamado `deployment.yaml`.
   ```powershell
   cd templates
   code deployment.yaml
   ```

11. Busca la definicion de `containers` y actualiza el valor de las propiedades `image` y `containerPort` para que coincidan con las siguentes:
    ```yaml
    containers:
      - name: {{ .Chart.Name }}
        securityContext: 
          {{- toYaml .Values.securityContext | nindent 12 }}
        image: "{{ .Values.image.repository }}:{{ .Chart.AppVersion }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
          - name: http
            containerPort: 5000
            protocol: TCP
    ```

12. Guarda los cambios y cierra el editor.

13. Ahora actualizaremos el archivo llamado `service.yaml`.
    ```powershell
    code service.yaml
    ``` 

14. Busca la definición `ports` y actualiza los valores para que coincidan con los siguientes:
    ```yaml
    ports:
      - port: {{ .Values.service.port }}
        targetPort: 5000
        protocol: TCP
        name: http
    ``` 

15. Guarda los cambios y cierra el editor.

16. El chart está ahora configurado para ejecutar nuestra aplicacion web. Escribe el siguiente comando para desplegar la aplicación descrita por los archivos YAML. Recibirá un mensaje indicando que `Helm` ha creado un despliegue y un servicio web.
   ```powershell
   cd ../..
   helm install demoapp ./demoapp
   ``` 

17. Comprobaremos que nuestra aplicacion se ha desplegado correctamente en el cluster de AKS, ejecutando el siguiente comando:
    ```powershell
    kubectl get pods,services,deployments -o wide
    ```
    Obtendremos un resultado similar al siguiente:
   <kbd>![kubectl get](https://user-images.githubusercontent.com/4158659/95560832-1accf800-0a1a-11eb-8e8a-1b89d5063675.png)</kb

18. Tomaremos nota de la IP pública del servicio `demoapp` y el puerto, y prepararemos la URL para acceder a través del navegador web. 
    <kbd>![demoapp_browser](https://user-images.githubusercontent.com/4158659/95850963-e2972380-0d51-11eb-983d-ff3345a265bf.png)</kbd>
   
   >**NOTA** En el apartado `Info` podremos observar que nuestra aplicación se está ejecutando en un contenedor y en Kubernetes.

### Tarea 3: Despliegue automatizado de Helm Chart 

En esta tarea automatizaremos el despliegue del Helm chart que hemos creado y probado en el paso anterior para que se ejecute de forma autónoma en el entorno de desarrollo tras la creación de una nueva versión de la imagen de la aplicación. Los pasos a seguir son los siguientes:

1. El primer paso es incluir en el archivo de definicion del workflow de GitHub las tareas necesarias para el empaquetado y publicación del Chart de Helm en el repositorio de imágenes, y para su posterior despliegue en el cluster AKS. 
   Editaremos el fichero cloudnativelab.yml en la carpeta de workflows de GitHub
   ```powershell
   cd ..
   code ./.github/workflows/cloudnativelab.yml
   ```

   Para incluir el siguiente contenido al final
   ```yaml
   package-and-publish-helm-chart:
      needs: [deploy-infra-with-terraform, build-and-publish-docker-image]
      name: Package and Push Helm Chart
      runs-on: ubuntu-latest
      env:
         ACR_NAME: cnlacr.azurecr.io
         CHART_NAME: demoapp         
         CHART_VERSION: v${{ github.run_id }}
         IMAGE_VERSION: ${{ github.run_id }}
         HELM_EXPERIMENTAL_OCI: 1

      steps:
      # Checkout the repo
      - uses: actions/checkout@v2
        
      # Install last Helm CLI
      - name: Install Helm
        uses: azure/setup-helm@v1
        id: installHelm
        with:
         version: 'latest'
      
      # Helm Login to ACR
      - name: Helm login to Azure Container Registry         
        run: | 
           echo ${{ secrets.ACR_PASSWORD }} |  helm registry login ${{ env.ACR_NAME }} --username ${{ secrets.ACR_USERNAME }} --password-stdin
         
      #Package and Publish Helm Chart to ACR
      - name: Publish Helm Chart
        run: | 
         helm package ./charts/${{ env.CHART_NAME }} --app-version ${{ env.IMAGE_VERSION }} --version ${{ env.IMAGE_VERSION }} --destination ./
         helm chart save ${{ env.CHART_NAME }}-${{ env.IMAGE_VERSION }}.tgz ${{ env.ACR_NAME }}/helm/${{ env.CHART_NAME }}:${{ env.CHART_VERSION }}
         helm chart save ${{ env.CHART_NAME }}-${{ env.IMAGE_VERSION }}.tgz ${{ env.ACR_NAME }}/helm/${{ env.CHART_NAME }}:latest
         helm chart push ${{ env.ACR_NAME }}/helm/${{ env.CHART_NAME }}:${{ env.CHART_VERSION }}
         helm chart push ${{ env.ACR_NAME }}/helm/${{ env.CHART_NAME }}:latest
         
   deploy-using-helm-chart:
      needs: [package-and-publish-helm-chart]
      name: Deploy to AKS cluster using Helm Chart
      runs-on: ubuntu-latest
      env:
         CHART_NAME: demoapp
         CHART_VERSION: v${{ github.run_id }}
         RESOURCE_GROUP_NAME: cloudnativelab
         ACR_NAME: cnlacr.azurecr.io
         AKS_CLUSTER_NAME: cnlaks
         AKS_NAMESPACE: default
         HELM_EXPERIMENTAL_OCI: 1

      steps:
      # Install last Helm CLI
      - name: Install Helm
        uses: azure/setup-helm@v1
        id: installHelm
        with:
         version: 'latest'

      # Helm Login to ACR
      - name: Helm login to Azure Container Registry         
        run: | 
           echo ${{ secrets.ACR_PASSWORD }} |  helm registry login ${{ env.ACR_NAME }} --username ${{ secrets.ACR_USERNAME }} --password-stdin

      # Set AKS Context
      - name: AKS config
        uses: azure/aks-set-context@v1
        with:
         creds: '${{ secrets.AZURE_CREDENTIALS }}'
         resource-group: ${{ env.RESOURCE_GROUP_NAME }}
         cluster-name: ${{ env.AKS_CLUSTER_NAME }}

      # Download and Publish Helm Chart
      - name: Publish Helm Chart
        run: |
            helm chart pull ${{ env.ACR_NAME }}/helm/${{ env.CHART_NAME }}:${{ env.CHART_VERSION }}
            helm chart export ${{ env.ACR_NAME }}/helm/${{ env.CHART_NAME }}:${{ env.CHART_VERSION }} --destination ./install
            helm upgrade \
               --namespace ${{ env.AKS_NAMESPACE }} \
               --create-namespace \
               --install \
               --wait \
               --version ${{ env.CHART_VERSION }} \
               ${{ env.CHART_NAME }} \
               ./install/${{ env.CHART_NAME }}
   ```
   >**NOTA** Los nombres de los dos nuevos jobs  `package-and-publish-helm-chart` y `deploy-using-helm-chart` deben quedar alienados verticalmente con los otros jobs existentes, y el resto del contenido debe mostrarse con la misma identación.

   Analicemos el contenido añadido al fichero de definicion del workflow de GitHub:
   >.Se ha incorporado un nuevo job denominado `package-and-publish-helm-chart` que empaqueta localmente en un archivo `tgz` el contenido del chart, y durante ese proceso ya modifica parte de la metadata del chart almacenada en el fichero `Chart.yaml`, utilizando el tag utilizado para etiquetar la imagen docker para el parámetro `--app-version` (tag de la imagen docker que se utilizará para el despliegue) y `--version` (versión del chart).  
   Con los comandos `helm chart save` se etiquetará la version del chart generada con la version `latest` y la equivalente a la versión de la imagen docker (hay que anteponer algun caracter,como la v, o especificarla en formato semVer).  
   Y con los comandos `helm chart push` se subirán al Container Registry ambas versiones del chart.  
   <kbd>![workflow_helm_1](https://user-images.githubusercontent.com/4158659/95977592-56e8ca00-0e19-11eb-8981-2a21e734971a.png)</kbd>

   >.  En el job denominado `deploy-using-helm-chart` se incluyen las tareas necesarias para descargar y extraer el chart preparado en el paso anterior, y proceder a su publicacion en el cluster AKS mediante el comando `helm upgrade` 
   <kbd>![workflow_helm_2](https://user-images.githubusercontent.com/4158659/95977594-57816080-0e19-11eb-9746-7edf2c2decbd.png)</kbd>

   >. En ambos jobs se han incluido algunas tareas adicionales para configurar la conexion con el Container Registry, o para configurar en el entorno local de ejecución la conectividad con el cluster de AKS.
   <kbd>![workflow_helm_3](https://user-images.githubusercontent.com/4158659/95977589-56503380-0e19-11eb-8076-af41e3008149.png)</kbd>
   
   >. Se ha prestado especial cuidado al parámetro `needs:` de cada job, para asegurar que la ejecución se realiza de forma secuencial y en el orden correcto. 
   <kbd>![workflow_helm_4](https://user-images.githubusercontent.com/4158659/95977591-56e8ca00-0e19-11eb-8093-3b90545261cf.png)</kbd>

2. Procederemos a subir los cambios al repositorio de código, ejecutando
   ```powershell
   git add .
   git commit -m "Helm Chart Added"
   git push 
   ```

3. Revisaremos que nuestra GitHub Action se ejecuta y finaliza correctamente.
   <kbd>![github_action_helm_finish](https://user-images.githubusercontent.com/4158659/95970144-e7baa800-0e0f-11eb-8fcd-2f50151d8743.png)</kbd>

4. Consultaremos el cluster de AKS para revisar que el despliegue se ha realizado correctamente, ejecutando el siguiente comando:
   ```powershell
   kubectl get pod,service,deployment -o wide
   ```
   Observaremos algo similar a lo mostrado en la siguiente imagen.
   <kbd>![kubectl_get](https://user-images.githubusercontent.com/4158659/95972997-6c5af580-0e13-11eb-9fab-67d52a3e42ce.png)</kbd>
   >***NOTA*** Especial atención a que ha quedado asociada como version del la imagen desplegada la ultima version, pero sin hacer uso del tag `latest` 

5. Y por ultimo, podemos volver a probar el correcto funcionamiento de la aplicación haciendo uso del navegador web. Es posible que al realizar el despliegue desde entornos distintos, y posiblemente con nombres de despliegue distintos, la IP Pública asignada haya cambiado.
   <kbd>![demoapp_browser_2](https://user-images.githubusercontent.com/4158659/95973346-ce1b5f80-0e13-11eb-9b91-6a129091bb7d.png)</kbd>



# Ejercicio 5: Limpieza

Por ultimo, eliminaremos los recursos, especialmente los de Azure, para evitar que generen costes innecesarios
En una ventana de powershell o linux bash conectada con Azure (az CLI), o desde la ventana de Bash Shell que ofrece el portal de Azure, ejecutaremos:

1. Para eliminar los recursos Azure, ejecutar:
   ```powershell 
   az group delete --name cloudnativelab --no-wait -y
   az group delete --name tstate --no-wait -y
   az ad app delete --id $(az ad app list --display-name cloudnativelab --query [0].appId)
   ```

2. Si también deseamos borrar el repositorio de GitHub que hemos utilizado para el laboratorio, debemos acceder al apartado `Settings`, en la parte superior derecha.
   <kbd>![GitHub_repo_Settings_1](https://user-images.githubusercontent.com/4158659/95974794-b218bd80-0e15-11eb-8959-383e28e8438d.png)</kbd>

   Y haciendo scroll hasta el final, visualizaremos una sección denominada `Danger Zone` donde está ubicado el botón `Delete this repository` 
   <kbd>![GitHub_repo_Settings_2](https://user-images.githubusercontent.com/4158659/95974795-b2b15400-0e15-11eb-87f0-a535f8242c9b.png)</kbd>



