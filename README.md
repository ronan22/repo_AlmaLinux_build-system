# repo_AlmaLinux_build-system

## Init build-system

```bash
mkdir -p build-system
cd build-system
BUILD_SYS_ROOT=$(pwd)
repo init  git@github.com:ronan22/repo_AlmaLinux_build-system.git
repo sync
```

## configure build-system

###  albs-web-server

#### Generate github client ID

You must registering your app:

https://github.com/settings/applications/new

Important:

Your "Authorization callback URL" must be set to: http://0.0.0.0:8080/auth/login/github

Now copy/past id and secret:

```bash
your_github_client_id=XXXXX
your_github_secret=XXXXX
```

#### Configure the DB

```bash
your_db_password=password
```

#### Configure rabbitmq

```bash
random_generated_string=XXXXX
rabbitmq_user=test-system
random_password=XXXXXX
random_secret="$(openssl rand -hex 32)"
```

sed -i "s/mqtt_queue_username:.*/mqtt_queue_username: ${rabbitmq_user}/g"  ./alws/scripts/albs-gitea-listener/albs-gitea-listener-config.yaml
sed -i "s/mqtt_queue_password:.*/mqtt_queue_password: ${random_password}/g"  ./alws/scripts/albs-gitea-listener/albs-gitea-listener-config.yaml
sed -i "s/albs_jwt_token:.*/albs_jwt_token: ${your_generated_token}/g"  ./alws/scripts/albs-gitea-listener/albs-gitea-listener-config.yaml

#### Configure alts

```bash
your_generated_token=XXXX
```

```bash
cd "${BUILD_SYS_ROOT}"
cd alts
rm -fr configs/alts_config.yaml
cp configs/example_config.yaml configs/alts_config.yaml



sed -i "s/use_ssl:.*/use_ssl: false/g"  "${BUILD_SYS_ROOT}/alts/configs/alts_config.yaml"

sed -i "s/result_backend:.*/result_backend: \'file:\/\/\/srv\/celery_results\'/g"  "${BUILD_SYS_ROOT}/alts/configs/alts_config.yaml"
sed -i "s/rabbitmq_user:.*/rabbitmq_user: \'${rabbitmq_user}\'/g"  "${BUILD_SYS_ROOT}/alts/configs/alts_config.yaml"
sed -i "s/rabbitmq_password:.*/rabbitmq_password: \'${random_password}\'/g"  "${BUILD_SYS_ROOT}/alts/configs/alts_config.yaml"

sed -i "s/pulp_host:.*/pulp_host: \'http:\/\/pulp\'/g"  "${BUILD_SYS_ROOT}/alts/configs/alts_config.yaml"
sed -i "s/pulp_user:.*/pulp_user: \'admin\'/g"  "${BUILD_SYS_ROOT}/alts/configs/alts_config.yaml"
sed -i "s/pulp_password:.*/pulp_password: \'admin\'/g"  "${BUILD_SYS_ROOT}/alts/configs/alts_config.yaml"

sed -i "s/jwt_secret:.*/jwt_secret: \'${random_secret}\'/g"  "${BUILD_SYS_ROOT}/alts/configs/alts_config.yaml"

sed -i "s/bs_host:.*/bs_host: \'http:\/\/web_server:8000\'/g"  "${BUILD_SYS_ROOT}/alts/configs/alts_config.yaml"
sed -i "s/bs_token:.*/bs_token: \'XXXXXXXX\'/g"  "${BUILD_SYS_ROOT}/alts/configs/alts_config.yaml"

sed -i "s/testing:.*/testing: \'true\'/g"  "${BUILD_SYS_ROOT}/alts/configs/alts_config.yaml"
sed -i "s/working_directory:.*/working_directory: \'\/srv\/alts\/scheduler\'/g"  "${BUILD_SYS_ROOT}/alts/configs/alts_config.yaml"
```

You need to feed the alts_config.yaml with fake value to avoid check error.

```bash
for f in s3_access_key_id \
            s3_secret_access_key \
            s3_bucket \
            s3_region \
            azureblockblob_container_name \
            azureblockblob_base_path \
            azure_connection_string \
            azure_logs_container \
            opennebula_rpc_endpoint \
            opennebula_username \
            opennebula_password \
            opennebula_vm_group; do
    sed -i "s/${f}:.*/${f}: \'FAKE\'/g"  "${BUILD_SYS_ROOT}/alts/configs/alts_config.yaml"
done
```

#### Configure albs-web-server

```bash
cd "${BUILD_SYS_ROOT}"
cd albs-web-server

cat << EOF > vars.env
POSTGRES_USER="postgres"
POSTGRES_PASSWORD="$your_db_password"
POSTGRES_DB="almalinux-bs"

DATABASE_URL="postgresql+asyncpg://postgres:$your_db_password@db/almalinux-bs"

GITHUB_CLIENT="$your_github_client_id"
GITHUB_CLIENT_SECRET="$your_github_secret"

ALTS_HOST="http://alts-scheduler:8000"
ALTS_TOKEN="$your_generated_token"
JWT_SECRET="${random_secret}"

RABBITMQ_ERLANG_COOKIE="${random_generated_string}"
RABBITMQ_DEFAULT_USER=${rabbitmq_user}
RABBITMQ_DEFAULT_PASS="${random_password}"
RABBITMQ_DEFAULT_VHOST="test_system"
EOF
```

If your github account is not part of the AlmaLinux organization, you must remove python code from:

alws/crud.py

"""bash
-        if not any(item for item in github_info['organizations']
-                   if item['login'] == 'AlmaLinux'):
-            return
"""

#### Start albs-web-server

```bash
cat << EOF > run.sh
#!/bin/bash

set -e pipefail

mkdir -p volumes/pulp/settings

echo "CONTENT_ORIGIN='http://pulp'
ANSIBLE_API_HOSTNAME='http://pulp'
ANSIBLE_CONTENT_HOSTNAME='http://pulp/pulp/content'
TOKEN_AUTH_DISABLED=True" >> volumes/pulp/settings/settings.py

docker-compose up -d --build --force-recreate --remove-orphans

#Fix mosquitto
docker exec -it albs-web-server_mosquitto_1 sh -c "echo 'allow_anonymous true' >> /mosquitto/config/mosquitto.conf"
docker exec -it albs-web-server_mosquitto_1 sh -c "echo 'listener 1883 0.0.0.0' >> /mosquitto/config/mosquitto.conf"
docker-compose restart mosquitto


sleep 25
docker exec -it albs-web-server_pulp_1 bash -c 'pulpcore-manager reset-admin-password --password="admin"'
EOF

chmod a+x ./run.sh

./run.sh
```

#### Check albs-web-server

Here somes useful command to debug the servers.

```bash
docker-compose ps

docker-compose logs -f web_server

docker-compose restart web_server

docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' xxxxx

```

#### Bootstrap a project

```bash
cd "${BUILD_SYS_ROOT}"
cd albs-web-server

pip3 install -r  requirements.txt


echo 127.0.0.1 db | sudo tee -a /etc/hosts
source vars.env
ALTS_TOKEN="$ALTS_TOKEN"  GITHUB_CLIENT="$GITHUB_CLIENT" GITHUB_CLIENT_SECRET="$GITHUB_CLIENT_SECRET" JWT_SECRET="$JWT_SECRET" PULP_HOST=http://0.0.0.0:8081 PULP_USER=admin PULP_PASSWORD=admin  python scripts/bootstrap_repositories.py -c reference_data/platforms.yaml -v
```
