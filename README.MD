# The Docker Modern Data Stack

This Repo is a guide to setting up a modern data stack all in Ubuntu on docker with just a few commands!

## Installation

Most of the content below are direct commands to run in the linux terminal. Just check each one before you run them in case there are passwords to enter or something.

### Installing Docker

for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do sudo apt-get remove $pkg; done

sudo apt-get update sudo apt-get install ca-certificates curl gnupg

sudo install -m 0755 -d /etc/apt/keyrings curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update

sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

sudo docker ps


### Install Python and Pip

sudo apt-get install python3.11

wget https://bootstrap.pypa.io/get-pip.py

python3 get-pip.py


### Install Portainer

docker volume create portainer_data

docker run -d -p 8000:8000 -p 9443:9443 --name portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce:latest

http://IPADDRESS:9443


### Install Minio (S3 Buckets)

mkdir -p ${HOME}/minio/data

docker run -p 9000:9000 -p 9090:9090 --user $(id -u):$(id -g) --name minio1 -e "MINIO_ROOT_USER=admin" -e "MINIO_ROOT_PASSWORD=password" -v ${HOME}/minio/data:/data quay.io/minio/minio server /data --console-address ":9090"

Use port 9090 for Web UI

Create a bucket in Object Browser

Create an Access Key

Use port 9000 with no TLS for S3 browser.


### Install Airflow

curl -LfO 'https://airflow.apache.org/docs/apache-airflow/2.7.0/docker-compose.yaml'

mkdir -p ./dags ./logs ./plugins ./config echo -e "AIRFLOW_UID=$(id -u)" > .env

Copy docker files from here to server

docker compose build

docker compose up airflow-init

docker compose up

docker-compose run airflow-worker airflow db upgrade

docker compose up

Default account = airflow / airflow

Remove Airflow

docker compose down --volumes --rmi all


### Install Postgres

docker run -d --name postgres-warehouse -e POSTGRES_PASSWORD=password -e PGDATA=/var/lib/postgresql/data/pgdata -v /var/lib/docker/volumes/postgres:/var/lib/postgresql/data -p 31250:5432 postgres


### Install Airbyte

git clone https://github.com/airbytehq/airbyte.git
cd airbyte
./run-ab-platform.sh
`Try below for no foreground process.`
`nohup ./run-ab-platform.sh >> nohup.out &`

Default account = airbyte / password


### Install Metabase

docker run -d -p 3000:3000 metabase/metabase


### Install Superset

docker run -d -p 31251:8088 -e "SUPERSET_SECRET_KEY=your_secret_key_here" --name superset apache/superset

docker exec -it superset superset fab create-admin
    --username admin
    --firstname Superset
    --lastname Admin
    --email admin@superset.com
    --password admin
docker exec -it superset superset db upgrade
docker exec -it superset superset load_examples
docker exec -it superset superset init
