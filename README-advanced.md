# iRacelog server setup (advanced)

**Disclaimer:** 
This document targets people who are familar with setting up a linux server which is connected to the internet. You have to deal with the securing the server and the application on your own. 

## Requirements

You need the following software installed on your server
- docker
- docker-compose

## iRacelog server

The iRacelog server runs via docker-compose. 

The following snippet serves as an example. There are two services that need to be accesible from the internet. 

- iracelog 
- crossbar

In this sample their ports are just forwarded. Keep in mind that this setup would use unsecured connections. So you may want to add a proxy which handles TLS (nginx,traefik or the like).

The configuration values in `.env` 
```console
CROSSBAR_DATAPROVIDER_TICKET=EnterPasswordHere
CROSSBAR_BACKEND_TICKET=EnterPasswordHere
CROSSBAR_ADMIN_TICKET=EnterPasswordHere
DB_USER_PASSWORD=EnterPasswordHere

DB_USER_NAME=docker
DB_SCHEMA=iracelog

RACELOG_URL=ws://crossbar:8080/ws
RACELOG_REALM=racelog
RACELOG_USER=backend
```

The crossbar configuration is located at [here](roles/apps/iracelog/files/crossbar/cb-racelog.json) in this repository and should be copied to `./config/crossbar/cb-racelog.json` relative to the docker-compose.yml

The frontend configuration should be created at `./config/iracelog/frontend.json` with the following

```json
{
    "crossbar": {
        "url": "ws://yourserver/ws",
        "realm": "racelog"
    }

}

```
Sample docker-compose.yml

```yml
version: "3.3"

services:

  # the frontend (needs to be accesible from the internet)
  iracelog:
    image: ghcr.io/mpapenbr/iracelog-web:v0.13.0
    restart: always
    logging:
      driver: "json-file"
      options:
        max-size: "2m"
        max-file: "5"
    ports:
      - "80:80"
    networks:
      - somenet

    environment:
      TZ: "Europe/Berlin"
    volumes:
      - ./config/iracelog/frontend.json:/usr/share/nginx/html/config.json

  # the entrypoint for the server (needs to be accesible from the internet)
  crossbar:
    image: crossbario/crossbar:cpy-amd64-21.2.1
    restart: always
    networks:
      - somenet      
    ports:
      - "8080:8080"
    logging:
      driver: "json-file"
      options:
        max-size: "2m"
        max-file: "5"
    environment:
      TZ: "Europe/Berlin"
      CROSSBAR_DATAPROVIDER_TICKET: ${CROSSBAR_DATAPROVIDER_TICKET}
      CROSSBAR_BACKEND_TICKET: ${CROSSBAR_BACKEND_TICKET}
      CROSSBAR_ADMIN_TICKET: ${CROSSBAR_ADMIN_TICKET}
    volumes:
      - ./config/crossbar/cb-racelog.json:/node/.crossbar/config.json

  # the following services are used internally only.

  # iracelog-service-manager
  ism-manager:
    image: ghcr.io/mpapenbr/iracelog-service-manager:v0.2.3
    restart: always
    logging:
      driver: "json-file"
      options:
        max-size: "2m"
        max-file: "5"
    networks:
      - somenet

    environment:
      TZ: "Europe/Berlin"
      RACELOG_URL: ${RACELOG_URL}
      RACELOG_REALM: ${RACELOG_REALM}
      RACELOG_USER: ${RACELOG_USER}
      RACELOG_PASSWORD: ${CROSSBAR_BACKEND_TICKET}
      DB_URL: postgresql://${DB_USER_NAME}:${DB_USER_PASSWORD}@db:5432/iracelog

    command: "ism --check http://crossbar:8080/info manager"
    depends_on:
      - db
      - crossbar

  ism-archiver:
    image: ghcr.io/mpapenbr/iracelog-service-manager:v0.2.3
    restart: always
    logging:
      driver: "json-file"
      options:
        max-size: "2m"
        max-file: "5"
    networks:
      - somenet

    environment:
      TZ: "Europe/Berlin"
      RACELOG_URL: ${RACELOG_URL}
      RACELOG_REALM: ${RACELOG_REALM}
      RACELOG_USER: ${RACELOG_USER}
      RACELOG_PASSWORD: ${CROSSBAR_BACKEND_TICKET}
      DB_URL: postgresql://${DB_USER_NAME}:${DB_USER_PASSWORD}@db:5432/iracelog

    command: "ism --check http://crossbar:8080/info archiver"
    depends_on:
      - db
      - crossbar

  # iracelog-analysis-service
  ias:
    image: ghcr.io/mpapenbr/iracelog-analysis-service:v0.1.1
    restart: always
    logging:
      driver: "json-file"
      options:
        max-size: "2m"
        max-file: "5"
    networks:
      - somenet

    environment:
      TZ: "Europe/Berlin"
      CROSSBAR_URL: ${RACELOG_URL}
      CROSSBAR_REALM: ${RACELOG_REALM}
      CROSSBAR_USER: ${RACELOG_USER}
      CROSSBAR_CREDENTIALS: ${CROSSBAR_BACKEND_TICKET}
      DB_URL: postgresql://${DB_USER_NAME}:${DB_USER_PASSWORD}@db:5432/iracelog

    depends_on:
      - db
      - crossbar

  

  db-migrate:
    image: ghcr.io/mpapenbr/iracelog-service-manager:v0.2.2
    networks:
      - somenet
    environment:
      DB_URL: postgresql://${DB_USER_NAME}:${DB_USER_PASSWORD}@db:5432/iracelog

    command: "./wait-for-it.sh -h db -p 5432 -s -- alembic -c src/iracelog_service_manager/db/alembic.ini  upgrade head"
    depends_on:
      - db

  db:
    image: postgres:14
    restart: always
    
    volumes:
      - db-data:/var/lib/postgresql/data

    logging:
      driver: "json-file"
      options:
        max-size: "2m"
        max-file: "5"

    environment:
      TZ: "Europe/Berlin"
      POSTGRES_DB: ${DB_SCHEMA}
      POSTGRES_USER: ${DB_USER_NAME}
      POSTGRES_PASSWORD: ${DB_USER_PASSWORD}

    networks:
      - somenet


  main-services:
    image: busybox
    command: sh -c "Starting services"
    depends_on:
      - db
      - crossbar
      - ism-manager
      - ism-archiver
      - ias
      - iracelog


volumes:
  db-data:

networks:
  somenet:

```    

