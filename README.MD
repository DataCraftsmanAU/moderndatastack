# Open Source Data Analytics Platform - Docker Modern Data stack

This Repo is a guide to setting up a modern data stack all in Ubuntu on docker with just a few commands!

## Out of Scope - TLS/HTTPS
Setting up using TLS and certificates adds a massive complexity to the install however it is possible. Generally you need to get some certificates made up for your server, put them someone safe in your home drive, then mount them to various locations on each application using -v /directory:/app/cert/dir and usually an -e TLSOPTION=true type environment variable.


## Installation

Most of the content below are direct commands to run in the linux terminal. Just check each one before you run them in case there are passwords to enter or something.

sudo sudo su

### Installing Docker
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do sudo apt-get remove $pkg; done

#### Add Docker's Repo
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" |
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update



#### Install and Test Docker
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

sudo docker run hello-world


### KeepassXC
You'll need somewhere to store your keys for all your apps. I recommend using Keepass, however you can skip this step if you are just using the same shit password for every app. To each their own.

docker run -d --name=keepassxc --security-opt seccomp=unconfined -e PUID=1000 -e PGID=1000 -e TZ=Etc/UTC -p 3000:3000 -p 3001:3001 -v keepass_data:/config --restart unless-stopped lscr.io/linuxserver/keepassxc:latest

localhost:3000

Create a database, follow the prompts.
Create a new group under Root and then you can click 'Add a New Entry' to generate and store passwords for your team to use as admin accounts for all the applications we are going to deploy.



### Install Portainer
docker run -d -p 38000:8000 -p 9443:9443 --name portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce:sts

Go to hostname:9443 in your browser and create a portainer admin account.

Go to environments and update the local details to your servers hostname in the Public IP field. This allows you to shell into your containers from the containers menu.



#### Install Docker Registry
This allows you to load your own docker images onto the server.

mkdir auth
docker run --entrypoint htpasswd httpd:2 -Bbn <USERNAME> <PASSWORD> > auth/htpasswd

docker run -d -p 5000:5000 --restart always -v registry_data:/var/lib/registry --name registry -v "$(pwd)"/auth:/auth -e "REGISTRY_AUTH=htpasswd" -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd registry:2.8.3

Test your account by doing
docker login localhost:5000 then enter the username and password you made.



#### Install Minio (S3 Buckets)

docker run -d -p 9000:9000 -p 9001:9001 --user $(id -u):$(id -g) --name minio -e "MINIO_ROOT_USER=admin" -e "MINIO_ROOT_PASSWORD=password" -v minio_data:/data quay.io/minio/minio:latest server /data --console-address ":9001"

##### Old CPU
If you have an old cpu, find an image that has cpuv1 in the tag such as:
quay.io/minio/minio:RELEASE.2024-06-04T19-20-08Z-cpuv1

Use port 9001 for Web UI
Create a bucket in Object Browser
Create an Access Key
Use port 9000 with no TLS for S3 browser.


### Install Airflow
mkdir airflow
cd airflow
curl -LfO 'https://airflow.apache.org/docs/apache-airflow/2.9.1/docker-compose.yaml'
mkdir -p ./dags ./logs ./plugins ./config 

#### Create .env file and add environment variables
vi .env
AIRFLOW_UID=50000
_AIRFLOW_WWW_USER_USERNAME=<USERNAME>
_AIRFLOW_WWW_USER_PASSWORD=<PASSWORD>

Press i to type in the VI editor. This is called Insert Mode.
Press Escape then :wq to save an exit VI. :)

Can vi into the docker-compose.yaml too to change AIRFLOW__CORE__LOAD_EXAMPLES: 'true' to 'false' if you don't need them. Chuck all your dags into the dags folder you created and they will automatically load into airflow. Also recommend changing the webserver port from 8080 to something else since everything uses that.

#### Additional PIP Requirements
_PIP_ADDITIONAL_REQUIREMENTS=

If you want to add any extra package on startup, add to the following env variable. This is useful for different providers like Jira, Airbyte, apache atlas, flink, kafka, spark, azure, google, github, etc. https://airflow.apache.org/docs/apache-airflow/stable/extra-packages-ref.html 

#### Permanent changes to packages.
Use this guide https://airflow.apache.org/docs/docker-stack/build.html to do a permanent change to the image that has those packages installed.
Basically you make a Dockerfile and a requirements.txt.

Build it with:
docker build localhost:5000/custom/airflow:2.9.1

Push it to your Docker registry with:
docker push localhost:5000/custom/airflow:2.9.1

Then add this to your .env but change for your image:
AIRFLOW_IMAGE_NAME=localhost:5000/custom/airflow:2.9.1

#### Build and start Airflow
docker compose build
docker compose up airflow-init
docker compose up -d

Default account is airflow / airflow if you didn't change in .env

#### To upgrade Airflow
docker-compose run airflow-worker airflow db upgrade
docker compose up -d

#### Remove Airflow, get rid of -rmi all to keep the images
docker compose down --volumes --rmi all



### Install Postgres
docker run -d --name postgres-warehouse -e POSTGRES_USER=<username> -e POSTGRES_PASSWORD=<password> -e PGDATA=/var/lib/postgresql/data/pgdata -v postgres_data:/var/lib/postgresql/data -p 31250:5432 postgres:16.3

#### Install pgadmin
docker run -d --name pgadmin -e 'PGADMIN_DEFAULT_EMAIL=<emailaddress>' -e 'PGADMIN_DEFAULT_PASSWORD=<password>' -e 'PGADMIN_CONFIG_ENHANCED_COOKIE_PROTECTION=True' -e 'PGADMIN_CONFIG_LOGIN_BANNER="Authorised users only!"' -e 'PGADMIN_CONFIG_CONSOLE_LOG_LEVEL=10' -v pgadmin_data:/var/lib/pgadmin -p 31280:80 dpage/pgadmin4:8.7

Login at localhost:31280, usually takes a minute to boot. 
Register your new postgres server to the pgadmin server list.



### Install Airbyte
Go back to your linux home directory and run this:
git clone https://github.com/airbytehq/airbyte.git
cd airbyte
./run-ab-platform.sh

#### Once you are happy it works, run Airbyte in background
nohup ./run-ab-platform.sh >> nohup.out &

localhost:8000
Default account = airbyte / password
Create an account with Email and Company


### Jupyterspark



### Business Intelligence / Dashboard Tools
I personally recommend paying for Tableau since your users will be interfacing with this part of the stack and it's just objectively good. Superset and Lightdash are both great Open Source tools if you are just using it internally or for yourself. I have played around with several other tools but none are as good as these. If you want something just on your desktop, another option is installing PowerBI Desktop through the Windows Store and just connect to your postgres database. I do this for my jobscraper tool. If you are really unsure you can setup a Semantic Layer using cube.js and just decide later, but I've never done this so I can't recommend it.



### Apache Superset
docker run -d -p 31251:8088 -e "SUPERSET_SECRET_KEY=your_secret_key_here" --name superset apache/superset

#### Create Accounts
docker exec -it superset superset fab create-admin
    --username admin
    --firstname Superset
    --lastname Admin
    --email admin@superset.com
    --password admin
docker exec -it superset superset db upgrade
docker exec -it superset superset load_examples
docker exec -it superset superset init



### Lightdash
git clone https://github.com/lightdash/lightdash
cd lightdash
export LIGHTDASH_SECRET="your-secret"
export PGPASSWORD="your-db-password"
docker-compose -f docker-compose.yml --env-file .env up --detach --remove-orphans


### Logging and Monitoring
git clone https://github.com/vegasbrianc/prometheus/
cd prometheus
docker compose up




## Retrieval Augmented Generation (RAG)
### PGVector
### Ollama
### LangChain/


## Lakehouse
### Hudi
### Spark
### Trino


## Streaming
### Pulsar
### Apache Druid
### Apache Flink


## MLOPS
### ClearML
### DVC


## HTAP
### TiDB
