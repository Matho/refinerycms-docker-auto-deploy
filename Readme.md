## How to dockerize and self host Refinery CMS demo app with auto building via Github Actions and auto deploy via Portainer

**Read time:** 15 minutes  
**Difficulty:** intermediate  
**Precondition:** basic knowledge of Linux commands, server with Linux (preferably Ubuntu), installed Docker, Postgres and Portainer, Github account, Docker Hub account, Rails app (Refinery) to dockerize

I'm running on Docker on my 2x Raspberry Pi self-hosted at home. More info about installation of this platform can be found at [https://github.com/Matho/dockerize-pi-2](https://github.com/Matho/dockerize-pi-2)
In this article we will write how to dockerize Refinery CMS demo app and self-host it on Raspberry Pi.
We will use Github Actions to do auto building and push image to Docker Hub. From the Docker Hub, the webhook will be send to Portainer and Portainer will fetch and deploy latest image from Docker Hub.

Note: This is not step-by-step tutorial for absolute beginner. Some of the commands are skipped, or described only by words. I expect, you have some basic docker knowledge and I do not explain Docker basics here.
Also, the docker stack deploy script is related to my mini cluster, where Traefik is running. I'm not writing more about it, for complete reference how I setuped my architecture check the already mentioned tutorial at [https://github.com/Matho/dockerize-pi-2](https://github.com/Matho/dockerize-pi-2)

## Dockerization

At the beginning, we need to have dockerized sample app. My starting point was git clone of [https://github.com/refinery/refinerycms-example-app](https://github.com/refinery/refinerycms-example-app)
End of dockerization process is located at my fork at [https://github.com/Matho/refinerycms-example-app](https://github.com/Matho/refinerycms-example-app)

This tutorial you can reuse also for case, that you want to dockerize your Refinery CMS project and self host on your VPS.

Clone the demo app / or use your Refinery app you want to dockerize. Switch to the directory of the app

At first, we will prepare shell scripts (from the root on your refinery project):

`$ vim bin/run.docker.sh`

```
#!/bin/bash
set -x
set -e
set -o pipefail

./bin/run.symlinks.docker.sh

bundle exec rake db:migrate

service nginx start

bundle exec puma -C config/puma.rb
```

`$ chmod +x bin/run.docker.sh`

Then

`$ vim bin/run.symlinks.docker.sh`

```
#!/bin/bash

# Create symlinks (use absolute paths)
for folder in 'tmp/cache' 'log' 'public/uploads' 'public/system'; do
  rm -rf "/app/$folder"
  mkdir -p "/app/shared/$folder"
  ln -sf "/app/shared/$folder" "/app/$folder"
done

mkdir -p /app/shared/nginx/cache/dragonfly
```

`$ chmod +x bin/run.symlinks.docker.sh`

Add Nginx config. At the docker container, Nginx will be exposed and listening on port 80. Rails requests will be forwarded to Puma cluster running in each docker container.

`$ mkdir -p config/etc/nging/conf.d`

`$ vim config/etc/nging/conf.d/nginx.docker.conf`

```
upstream app {
  server unix:/app/puma.sock fail_timeout=0;
}

proxy_cache_path /app/shared/nginx/cache/dragonfly levels=2:2 keys_zone=dragonfly:100m inactive=30d max_size=1g;
server {
  listen 80 default_server;
  root /app/public;

  location ^~ /assets/ {
    gzip_static on;
    expires max;
    add_header Cache-Control public;
    add_header Vary Accept-Encoding;
  }

  try_files $uri/index.html $uri $uri.html @app;
  location @app {
    proxy_set_header Host $http_host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Host $server_name;
    proxy_pass_request_headers      on;

    proxy_redirect off;
    proxy_pass http://app;

    proxy_connect_timeout       1800;
    proxy_send_timeout          1800;
    proxy_read_timeout          1800;
    send_timeout                1800;

    proxy_buffer_size   128k;
    proxy_buffers   4 256k;
    proxy_busy_buffers_size   256k;

    gzip             on;
    gzip_min_length  1000;
    gzip_proxied     expired no-cache no-store private auth;
    gzip_types       text/plain text/css application/x-javascript text/xml application/xml application/xml+rss text/javascript;
    gzip_disable     "MSIE [1-6]\.";
  }

  error_page 500 502 503 504 /500.html;
  client_max_body_size 4G;

  client_body_timeout 12;
  client_header_timeout 12;
  keepalive_timeout 20;
  send_timeout 10;

  client_body_buffer_size 10K;
  client_header_buffer_size 1k;
  large_client_header_buffers 4 32k;

  server_tokens off;
}
```

Prepare database file

`$ vim config/database.yml`
```
default: &default
  adapter: postgresql
  encoding: unicode
  pool: <%= ENV['RAILS_MAX_THREADS'].to_i * 30 %>
  host: <%= ENV.fetch("POSTGRES_HOST") { 'postgres' } %>
  database: <%= ENV.fetch("POSTGRES_DB") { 'db' } %>
  username: <%= ENV.fetch("POSTGRES_USER") { 'postgres' } %>
  password: <%= ENV.fetch("POSTGRES_PASSWORD") { 'postgres' } %>
  port: <%= ENV.fetch("POSTGRES_PORT") { 5432 } %>

development:
  <<: *default

staging:
  <<: *default

test:
  <<: *default
  database: <%= ENV.fetch("POSTGRES_TEST_DB") { 'db_test' } %><%= ENV['TEST_ENV_NUMBER'] %>

production:
  <<: *default
```

Then prepare Puma config  
`$ vim config/puma.rb`

```
workers ENV.fetch("RAILS_WORKERS") { 2 }

threads_min = ENV.fetch("RAILS_MIN_THREADS") { 5 }
threads_max = ENV.fetch("RAILS_MAX_THREADS") { 5 }
threads threads_min, threads_max

# Specifies the `environment` that Puma will run in
environment ENV.fetch("RAILS_ENV") { "development" }

# Executed in Docker container
if File.exists?('/.dockerenv')
  app_dir = File.expand_path("../..", __FILE__)

  stdout_redirect "#{app_dir}/log/puma.stdout.log", "#{app_dir}/log/puma.stderr.log", true

  bind "unix://#{app_dir}/puma.sock"
else
  # Listen only on port
  port ENV.fetch("PORT") { 3000 }
end

# Allow puma to be restarted by `rails restart` command.
plugin :tmp_restart

```

Secrets file:
`$ vim config/secrets.yml`

```
development:
  secret_key_base: <%= ENV.fetch("SECRET_KEY_BASE") { '123456gw555ssd12166fc8b0b3a8462be050811114e351ae2f6634b284411a3acf6d923d17c778f355b45eb123456' } %>

test:
  secret_key_base: <%= ENV.fetch("SECRET_KEY_BASE") { '123456gw555ssd12166fc8b0b3a8462be050811114e351ae2f6634b284411a3acf6d923d17c778f355b45eb123456' } %>

production:
  secret_key_base: <%= ENV.fetch("SECRET_KEY_BASE") { '123456gw555ssd12166fc8b0b3a8462be050811114e351ae2f6634b284411a3acf6d923d17c778f355b45eb123456' } %>
```

Docker ignore files:

`$ vim .dockerignore`

```
Dockerfile
.byebug_history
.rspec
README.md
bin/build.docker.sh
bin/deploy.docker.sh
doc/*
log/*
tmp/*
.git/*
.idea/*
coverage/*
public/uploads/*
public/system/*
docker-compose.*
node_modules/*
!.env.production
!.env.example
```

Env example:
`$ .env.example`

```
SMTP_ADDRESS=
SMTP_DOMAIN=
SMTP_USERNAME=
SMTP_PASSWORD=

POSTGRES_HOST=localhost
POSTGRES_DB=refinerycms_demo
POSTGRES_USER=root
POSTGRES_PASSWORD=

# This is only needed to start the app for asset precompilation
SECRET_KEY_BASE=123456gw555ssd12166fc8b0b3a8462be050811114e351ae2f6634b284411a3acf6d923d17c778f355b45eb123456

RAILS_WORKERS=2
RAILS_MIN_THREADS=5
RAILS_MAX_THREADS=5
```

To your .gitignore file append:

```
.env.*
!.env.example
config/secrets.yml
```

The most important file is Dockerfile

`$ vim Dockerfile`

```
FROM mathosk/rpi-ruby-2.6.5-ubuntu-aarch64:latest

MAINTAINER Matho "martin.markech@matho.sk"

RUN apt-get update && apt-get install -y \
    curl \
    vim \
    git \
    build-essential \
    libgmp-dev \
    libpq-dev \
#    postgresql-client \
    locales \
    nginx \
    cron \
    bash \
    imagemagick \
    python \
    nodejs \
    npm

RUN npm install --global yarn

WORKDIR /app

ARG BUNDLE_CODE__MATHO__SK
ARG RAILS_ENV

ADD ./Gemfile ./Gemfile
ADD ./Gemfile.lock ./Gemfile.lock

RUN gem install bundler -v '> 2'
RUN bundle install --deployment --clean --path vendor/bundle --without development test --jobs 8

ADD . .

# Set Nginx config
ADD config/etc/nginx/conf.d/nginx.docker.conf /etc/nginx/conf.d/default.conf
RUN rm /etc/nginx/sites-enabled/default

ADD .env.example .env.production

RUN ASSET_PRECOMPILE_MODE=1 bundle exec rake assets:precompile RAILS_ENV=production --trace

RUN echo $(date) > BUILD_DATE.txt

ARG GITHUB_SHA
RUN echo $GITHUB_SHA > DEPLOYED_REVISION.txt

EXPOSE 80

CMD bin/run.docker.sh
```

Some changes in Gemfile:

`$ vim Gemfile`

At the end of file, append:
```
# use old version from Github, the used versions were yanked on RubyGems
gem 'mimemagic', github: 'mimemagicrb/mimemagic', ref: '01f92d86d15d85cfd0f20dabd025dcbd36a8a60f'

# For ENV values
gem 'dotenv-rails'
```

With this changes, you should be able to build the docker image.

## Auto building on Github and auto deploy

To be able build the Refinery CMS demo app, we need at first build the base docker image. My base docker repo is located at [https://github.com/Matho/rpi-ruby-2.6.5-ubuntu-aarch64](https://github.com/Matho/rpi-ruby-2.6.5-ubuntu-aarch64)
It is based on Ubuntu 20.04.

The Github actions are configured, to build this image for aarch64. It is instruction architecture of Raspberry Pi. If you need to host it on standard server, you would need the amd64 architecture

The recipe for build on Github Action is located in the folder `.github/workflows`
If you open the file `docker-image.yml` you will see this code:
```
name: Build

on:
  push:
    branches: main

jobs:
  build:
    runs-on: ubuntu-20.04
    steps:
      - name: checkout code
        uses: actions/checkout@v2
      - name: install buildx
        id: buildx
        uses: crazy-max/ghaction-docker-buildx@v1
        with:
          version: latest   
      - name: login to docker hub
        run: echo "${{ secrets.DOCKER_PASSWORD }}" | docker login -u "${{ secrets.DOCKER_USERNAME }}" --password-stdin          
      - name: build the image
        run: |
          docker buildx build --push \
            --tag mathosk/rpi-ruby-2.6.5-ubuntu-aarch64:latest \
            --platform linux/aarch64 .
```

The recipe was based on this video - [https://www.youtube.com/watch?v=5yeeMdhGgrE&ab_channel=Piwi%27sTechTalk](https://www.youtube.com/watch?v=5yeeMdhGgrE&ab_channel=Piwi%27sTechTalk)
It says, that it will build the image on each push to master branch. After the build is done, it will push the builded image to Docker Hub. It will tag always with the tag `latest`, which
will override the latest version. The `--platform linux/aarch64` says that it will build the image only for the aarch64 platform. Feel free to add more platforms, or build only on the one you need.
Because it is building on non native aarch64 device, the building takes a long time, ~ 1 hour .

I expect you have already existing account on Docker Hub. If not, create one. Create Docker Hub repository for the base image and the second one for Refinery CMS example app.
Public repositories are for free. Private are paid - but you can have one free private repository.

Our docker images will not be build on Docker Hub, but via Github Actions. Therefore, you will need to connect Github Actions with Docker Hub. We don't want to use the Docker Hub password,
we will use access token for that.

In the right upper corner, navigate to your account name, from dropdown list select Account Settings. Then from the left navigation select Security. Then click on New Access Token.
Copy the token which is shown. It is shown only once, so copy it somewhere.

Then navigate back to your Github repository. If you see the `docker-image.yml` file, you can see there `DOCKER_PASSWORD` and `DOCKER_USERNAME`. Now we will set this ENV in Github.
Open tab Settings. Then from the left navigation select Secrets. Fill in the DOCKER_PASSWORD and DOCKER_USERNAME credentials.

When the base docker image will be builded and pushed to Docker Hub, download it from Docker Hub to your rpi server.

Then we will need to prepare Docker stack script:

`$ vim refinerycms-projects-compose.yml`

```
version: "3.6"

services:
  refinerycms:
    image: mathosk/refinerycms-example-app:latest 
    networks:
      spilo_db-nw:
        ipv4_address: 10.0.2.4
    ports:
      - "7087:80"
    volumes:
      - "/storage-pool/data/refinerycms_example_app:/app/shared"
    environment:
      - POSTGRES_HOST=10.0.2.3
      - POSTGRES_DB=refinerycms_demo
      - POSTGRES_USER=refinerycms_demo
      - POSTGRES_PASSWORD=
      - POSTGRES_PORT=8432
      - RAILS_ENV=production
      - RAILS_WORKERS=2
      - RAILS_MIN_THREADS=5
      - RAILS_MAX_THREADS=5
      - SECRET_KEY_BASE=your-prod-value-here
    deploy:
      labels:
        - traefik.http.routers.refinerycms.rule=Host(`refinerycms.matho.sk`)
        - traefik.http.services.refinerycms-service.loadbalancer.server.port=80
        - traefik.docker.network=spilo_db-nw
      placement:
        constraints:
          - "node.role==worker"
      replicas: 2 
networks:
  spilo_db-nw:
    external: true
```
As you can see, there is restriction to run only on node.role==worker. Also there are Traefik labels to run the project on domain `refinerycms.matho.sk`

Then you need to create database user `refinerycms_demo`. You can use Pgadmin for it.

Deploy docker stack script via:  
`$ sudo docker stack deploy --compose-file refinerycms-projects-compose.yml refinerycms-projects`

Then you should see in Portainer this stack service.

In the Refinery CMS sample Github project, configure Github Action to build the project automatically. Do not forget to setup secret ENV variables also for this project.

`$ mkdir .github/workflows`

```
name: Build

on:
  push:
    branches: master

jobs:
  build:
    runs-on: ubuntu-20.04
    steps:
      - name: checkout code
        uses: actions/checkout@v2
      - name: install buildx
        id: buildx
        uses: crazy-max/ghaction-docker-buildx@v1
        with:
          version: latest   
      - name: login to docker hub
        run: echo "${{ secrets.DOCKER_PASSWORD }}" | docker login -u "${{ secrets.DOCKER_USERNAME }}" --password-stdin          
      - name: build the image
        run: |
          docker buildx build --build-arg GITHUB_SHA=$GITHUB_SHA --push \
            --tag mathosk/refinerycms-example-app:latest \
            --platform linux/aarch64 .

```

When the connection with Docker Hub works, you will have latest image under the tag latest on Docker Hub. But how to trigger deploy, when the image is pushed to Docker Hub?
Docker Hub has so called `Webhooks` and integration with Portainer is easy. Check this tutorial [https://documentation.portainer.io/v2.0/webhooks/create/](https://documentation.portainer.io/v2.0/webhooks/create/)

Open Portainer. Go to services from left menu and then select the refinerycms service. Then activate the Service webhook switch and copy the link. Then go back to Docker Hub to Webhook section and
copy paste this link from Portainer. Now once the image will be pushed, Portainer will receive this webhook and will pull the latest image from Docker Hub and restart services to start with the latest image. Just keep the link secret -
anybody who has this link, can force deploy of latest image to your infra.

If you want to see status of current or historic Github builds, you can navigate to menu Actions and there submenu Builds.

## Recreate db periodically

Sometimes (like demo app hosting), you need to recreate db periodically.

The problem is, that you can't simply drop db, if there is active connection to it. So the solution is to delete only public schema, instead of the whole database.

Create default db dump

`$ pg_dump --dbname=postgresql://user:password@127.0.0.1:5432/refinerycms_demo --schema=public > 2021_07_23_initial_backup.sql`

`$ vim reload_refinerycms.sh `

```
#!/bin/bash

psql --dbname=postgresql://postgres:password@127.0.0.1:5432/refinerycms_demo -c "DROP SCHEMA public CASCADE"

psql --dbname=postgresql://postgres:password@127.0.0.1:5432/refinerycms_demo -c "CREATE SCHEMA public"

psql --dbname=postgresql://postgres:password@127.0.0.1:5432/refinerycms_demo  < /home/ubuntu/pg_reloading/2021_07_23_initial_backup.sql

psql --dbname=postgresql://postgres:password@127.0.0.1:5432/refinerycms_demo -c "GRANT ALL ON SCHEMA public TO public;"

echo "Done"

```

`$ crontab -e`
```
0 12 * * * /bin/bash /home/ubuntu/pg_reloading/reload_refinerycms.sh >/dev/null 2>&1
```

This will restore db each day at 12 AM.

Thats all for now!
