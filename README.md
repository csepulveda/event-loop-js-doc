#Paso 1: Creación del entorno AWS.

##IAM
###Creación de Politicas *https://console.aws.amazon.com/iam/home#/policies*

- Política que permitira desde instancia EC2 acceder a nuestros bucket S3:

**`CodeDeploy-EventLoopJS-EC2-Permissions`**
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "s3:Get*",
                "s3:List*"
            ],
            "Effect": "Allow",
            "Resource": "*"
        }
    ]
}
```
- Política que permitira a Travis alojar archivos en nuestro Bucket S3:

**`Travis-EventLoopJS-Deploy-To-S3`**
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "s3:PutObject"
            ],
            "Effect": "Allow",
            "Resource": "*"
        }
    ]
}
```
- Política que permitira a Travis interactuar con el servicio de CodeDeploy:

**`Travis-EventLoopJS-Code-Deploy-Policy`**
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "codedeploy:RegisterApplicationRevision",
                "codedeploy:GetApplicationRevision"
            ],
            "Resource": [
                "arn:aws:codedeploy:us-east-1:ID:application:EventLoopJS"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "codedeploy:CreateDeployment",
                "codedeploy:GetDeployment"
            ],
            "Resource": [
                "*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "codedeploy:GetDeploymentConfig"
            ],
            "Resource": [
                "arn:aws:codedeploy:us-east-1:ID:deploymentconfig:CodeDeployDefault.OneAtATime",
                "arn:aws:codedeploy:us-east-1:ID:deploymentconfig:CodeDeployDefault.HalfAtATime",
                "arn:aws:codedeploy:us-east-1:ID:deploymentconfig:CodeDeployDefault.AllAtOnce"
            ]
        }
    ]
}
```
###Creación de Usuarios *https://console.aws.amazon.com/iam/home#/users*

- Creamos el usuario `Travis-EventLoopJS`
- Agregamos los Policies: `Travis-EventLoopJS-Deploy-To-S3` y `Travis-EventLoopJS-Code-Deploy-Policy`

###Creación de Roles *https://console.aws.amazon.com/iam/home#/roles*

- Creamos el Rol que permitirá a nuestras instancias llegar a los bucket S3.
  - Creamos el Rol `CodeDeploy-EventLoopJS-EC2-DeployInstance`
  - Seleccionamos el tipo `Amazon EC2`
  - Seleccionamos la política `CodeDeploy-EventLoopJS-EC2-Permissions`
- Creamos el Rol que permitirá a CodeDeploy ejecutar instrucciones sobre nuestras Instancias.
  - Creamos el Rol `CodeDeployEventLoopJSServiceRole`
  - Seleccionamos el tipo `AWS CodeDeploy`
  - Seleccionamos la política `AWSCodeDeployRole`

##S3
###Creación del Bucket *https://console.aws.amazon.com/s3/*
- Creamos el bucket `event-loop-js` en la zona `us-east-1`
- Activamos el versionamiento en el bucket

##EC2
###Creación de AMI Base: *https://console.aws.amazon.com/ec2/v2/home?region=us-east-1#Instances*

- Lanzamos una nueva instancia con Ubuntu 16.04
- Creamos el gruopo de seguridad `EventLoopJS-AppInstances`
    - Permitimos solo `SSH` para nuestra ip publica.
- Descargamos la llave y la movemos a nuestra carpeta `~/.ssh/` con permisos `600`
- Nos conectamos a la Instancia con la llave + usuario `ubuntu` y comenzamos la instalación base.
```bash
###Instalamos Node 6.x + Herramientas base + Nginx
curl -sL https://deb.nodesource.com/setup_6.x | sudo -E bash -
sudo apt-get install -y nodejs nginx-extras python-pip ruby wget imagemagick graphicsmagick htop
wget https://aws-codedeploy-us-east-1.s3.amazonaws.com/latest/install
chmod a+x install
sudo ./install auto

##configuramos nginx:
cat << EOF > /etc/nginx/sites-enabled/default
upstream app {
  server 127.0.0.1:8081 max_fails=0 fail_timeout=10s;
  keepalive 512;
}

server {
  listen   *:80;
  access_log /var/log/nginx/access.log;
  error_log /var/log/nginx/error.log;
  keepalive_timeout 10;

  location / {
    proxy_set_header Host \$host;
    proxy_set_header X-Real-IP \$remote_addr;
    proxy_set_header   Connection "";
    proxy_http_version 1.1;
    proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
    proxy_pass http://app;
    client_max_body_size       16m;
    client_body_buffer_size    128k;
    proxy_buffering     on;
    proxy_connect_timeout      90;
    proxy_send_timeout         90;
    proxy_read_timeout         120;
    proxy_intercept_errors     on;
  }
}
EOF
```
- Desde la Consola de EC2, generamos Nuestra AMI Base, Luego de eso damos `terminate` a la instancia.

###Creación Base de datos MongoDB: *https://console.aws.amazon.com/ec2/v2/home?region=us-east-1#Instances*

- Lanzamos una nueva instancia de tipo Ubuntu 16.04
- Asignamos el siguiente `user-data`:

```
#cloud-config

runcmd:  
  - [ bash, -c, 'apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 0C49F3730359A14518585931BC711F9BA15703C6' ]
  - [ bash, -c, 'echo "deb http://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/testing multiverse" | tee /etc/apt/sources.list.d/mongodb-org-3.4.list' ]
  - [ bash, -c, 'apt-get update' ]
  - [ bash, -c, 'apt-get install -y mongodb-org' ]
  - [ bash, -c, "sed -i -- 's/127.0.0.1/0.0.0.0/g' /etc/mongod.conf" ]
  - [ bash, -c, 'service mongod restart' ]
```

- Creamos el gruopo de seguridad `EventLoopJS-MongoDB`
    - Permitimos solo `SSH` para nuestra ip publica.
    - Permitimos el puerto `27017` desde el grupo de seguridad `EventLoopJS-AppInstances`
- Lanzamos la instancia y guardamos su IP Interna.

###Creación Load Balancer: *https://console.aws.amazon.com/ec2/v2/home?region=us-east-1#LoadBalancers:*
- Creamos un nuevo `Classic Load Balancer` con el nombre `EventLoopJS`
- Creamos un nuevo SecurityGroup con el nombre `EventLoopJS-ElasticLoadBalancer` el cual permita el puerto 80 para el mundo.
- Configuramos el Health Check a `/status`
- No Agregamos instancias.
- Agregamos el tag `Name=EventLoopJS`

###Creación Launch Configuration: *https://console.aws.amazon.com/ec2/autoscaling/home?region=us-east-1#LaunchConfigurations:id=*
- Creamos un nuevo Launch Configuration con instancia del tipo `Ubuntu Server 16.04`
- Nombre `EventLoopJS-AppInstance`
- IAM role: `CodeDeploy-EventLoopJS-EC2-DeployInstance`
- Habilitamos `Enable CloudWatch detailed monitoring`
- Seleccionamos el SecurityGroup `EventLoopJS-AppInstances`

###Creación Auto Scaling Group: *https://console.aws.amazon.com/ec2/autoscaling/home?region=us-east-1#AutoScalingGroups:id=*
- Creamos un nuevo Auto Scaling Group utilizando la configuración `EventLoopJS-AppInstance`
- Configuramos el nombre como `EventLoopJS-AutoScalingGroup` y seleccionamos todas las Subredes disponibles.
- En Advanced Details agregamos el balanceador previamente creado.
- Creamos una regla de autoescalamiento en base al uso de CPU mayor al 70% por mas de dos minutos que añada dos instancias.
- Creamos una regla de autoescalamiento en base al uso de CPU menor al 30% por mas de dos minutos que baje 1 instancia.
- Creamos el Tag `Name=EventLoopJS-AppInstance` y que se aplique a las nuevas instancias.

###Ultimo detalle: *https://console.aws.amazon.com/ec2/v2/home?region=us-east-1#SecurityGroups:*
- Editamos el Grupo de Seguridad `EventLoopJS-AppInstances`
    - Agregamos el grupo de seguridad `EventLoopJS-ElasticLoadBalancer` permitiendo trafico `HTTP`

##CodeDeploy
###Crear Aplicación *https://console.aws.amazon.com/codedeploy*
- Creamos nueva aplicación con el nombre `EventLoopJS`
- Configuramos el grupo `MASTER`
- En `Search by Tags` seleccionamos `Tag Type=Auto Scaling Group` y `Key=EventLoopJS-AutoScalingGroup`
- En `Service Role ARN` seleccionamos `arn:aws:iam::ID:role/CodeDeployEventLoopJSServiceRole`

#Paso 2: Creación y configuración de la  aplicación

##Example: https://github.com/csepulveda/event-loop-js

##PM2 *https://github.com/Unitech/pm2*

```bash
npm install -g pm2
```

Crear archivo JSON de configuración:

```js
{
  "apps" : [{
    "name"      : "server",
    "script"    : "./index.js",
    "watch"     : false,
    "instances" : "4",
    "exec_mode" : "cluster",
    "wait_ready": true,
    "env": {
      "SOME_ENV_VARIABLE": "some value"
    }
  }]
}
```

```bash
pm2 start
```

##Pruebas

Crear pruebas de unidad y configurar el script test en el package.json:

```js
"scripts": {
  ...
  "test": "mocha test/**/*.js"
}
```

Configurar herramienta de code coverage para métricas mínimas de prueba:

```js
"scripts": {
  ...
  "coverage": "istanbul cover _mocha -- test/**/*.js && istanbul check-coverage"
}
```

##Configuramos archivo .travis.yml
```
cache:
  bundler: true
  directories:
    - node_modules
language: node_js
node_js:
- '6'
script:
- mkdir build && zip -y -q -1 -r build/build.zip *
deploy:
  - provider: s3
    access_key_id: KEY
    bucket: event-loop-js
    skip_cleanup: true
    upload-dir: master
    local_dir: build
    region: us-east-1
    on:
      branch: master
  - provider: codedeploy
    access_key_id: KEY
    secret_access_key:
    bucket: event-loop-js
    key: master/build.zip
    application: EventLoopJS
    deployment_group: MASTER
    region: us-east-1
    on:
      branch: master
```
- Generamos `secret_access_key` segura: `travis encrypt`
- Agregamos la llave a nuestros deploy como

```
secret_access_key:
  secure:
```
##Configuramos archivo appspec.yml

```
version: 0.0
os: linux
files:
  - source: /
    destination: /var/www/event-loop-js/app/
hooks:
  BeforeInstall:
    - location: bash/delete-files.sh
      timeout: 120
  ApplicationStart:
    - location: bash/restart-app.sh
      timeout: 13
 ```

#Paso 3: Configuración Travis-CI
- Login + link de cuenta
- Seleccionamos el Repo
- Listo!
