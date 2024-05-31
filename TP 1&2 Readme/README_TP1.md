# Discover Docker TP1

## Database
FOr the beginnning of the tp we started by creating the database on postgres with the credentials, database_name = db, user = usr and password = pwd. We will be keeping these credientials the same throughout the tp.
~~~bash
FROM postgres:14.1-alpine

ENV POSTGRES_DB=db \
   POSTGRES_USER=usr \
   POSTGRES_PASSWORD=pwd
~~~

#### Creating network

Code:
~~~bash
  docker network create app-network
~~~
This is the network where the 3 tiers will be connected later on to be able to communicate. The name of my network is "app-network".
#### Creating the database
Code:
~~~bash
  docker run -d --name postgres -v ${PWD}/data:/var/lib/postgresql/data --net=app-network -p 5432:5432 my-postgres:1.0
~~~
My database name is 'postgres'

#### Connecting adminer(database management tool) to the database

~~~bash
docker run -p "8090:8080" --net=app-network --name=adminer -d adminer
~~~
#### Persist data

Adding a volume to persist data on the host disk.
~~~
volumes:
   - postgresdata:/var/lib/postgresql/data
~~~
###### 1-1 Why do we need a volume to be attached to our postgres container?


Attaching a volume to a container ensures data persistence beyond the container's lifetime. It allows data to be stored in a persistent storage area managed by Docker, rather than in the container's file system. This facilitates data management and migration, as the volume can be easily copied or moved.


## Backend API

## Multistage build

#### Dockerfile
this is javabased image I used: 
~~~bash
FROM eclipse-temurin:17-jre-alpine

Set the working directory inside the container

WORKDIR /app

// Copy the compiled Java class into the container

COPY /src/main/java/Main.class .

// Command to run the Java program

CMD ["java", "Main"]
~~~


## Backend API
~~~bash
# Spring configuration for JPA and data source
spring:
  jpa:
    # Hibernate-specific configuration for LOB (large object) data types
    properties:
      hibernate:
        jdbc:
          lob:
            non_contextual_creation: true
    # Disable automatic schema generation
    generate-ddl: false
    # Enable Open-in-View to allow JPA entities to be accessed in views
    open-in-view: true
  datasource:
    # URL for the PostgreSQL database
    url: jdbc:postgresql://localhost:5432/db
    # Username for the database
    username: usr
    # Password for the database
    password: pwd
    # Driver class name for the database
    driver-class-name: org.postgresql.Driver

# Management configuration for Spring Boot Actuator
management:
  server:
    # Disable the application context header header in management endpoints
    add-application-context-header: false
  endpoints:
    web:
      exposure:
        # Enable the following management endpoints to be accessed over HTTP
        include: health,info,env,metrics,beans,configprops
~~~



## Reverse proxy

~~~bash
# Docker commands for building and running the Apache reverse proxy container
# Build the Docker image for the Apache reverse proxy with the tag "apache-reverse-proxy"
docker build -t apache-reverse-proxy .
# Run the Apache reverse proxy container with the name "Apache-reverse-proxy",
# connected to the "app-network" network, and with port 80 on the host mapped to port 80 in the container
docker run  --name Apache-reverse-proxy --network app-network -p 80:80 apache-reverse-proxy
~~~
##### Why do we need a reverse proxy?
 - A reverse proxy is a server that sits in front of one or more application servers.
 - It forwards incoming client requests to the appropriate backend server.
 - Reverse proxies can distribute incoming requests evenly across multiple backend servers.
 - They can provide an additional layer of security by hiding the IP addresses of the backend servers.
 - Reverse proxies can handle SSL/TLS encryption and decryption.
 - They can cache frequently-accessed content to reduce the load on the backend servers.
 - Reverse proxies can rewrite URLs to simplify the URL structure or to route requests to different backend servers.

## Link application
### Docker-compose

#### docker-compose.yml

~~~bash
 # Define three services: backend, database, and httpd
services:

  # Backend service
  backend:
    # Set the container name
    container_name: backend
    # Build the service from the specified directory
    build: .\Backend\simple-api-student-main\
    # Set environment variables for the database connection
    environment:
      DB_host: postgres
      DB_port: 5432
      DB_name: db
      DB_user: usr
      DB_mdp: pwd
    # Map the container's port 8080 to the host's port 8080
    ports:
      - "8080:8080"
    # Connect the service to the app-network network
    networks:
      - app-network
    # Start the service after the database service is started
    depends_on:
      - database

  # Database service
  database:
    # Set the container name
    container_name: database
    # Build the service from the specified directory
    build: .\postgres\
    # Set environment variables for the PostgreSQL database
    environment:
      POSTGRES_DB: db
      POSTGRES_USER: usr
      POSTGRES_PASSWORD: pwd
    # Mount a Docker volume named postgresdata to the PostgreSQL data directory
    volumes:
      - postgresdata:/var/lib/postgresql/data
    # Connect the service to the app-network network
    networks:
      - app-network

  # HTTPD service
  httpd:
    # Set the container name
    container_name: httpd
    # Build the service from the specified directory
    build: .\httpd\
    # Set an environment variable for the backend service hostname
    environment:
      BACKEND_host: backend
    # Map the container's port 80 to the host's port 80
    ports:
      - "80:80"
    # Connect the service to the app-network network
    networks:
      - app-network
    # Start the service after the backend service is started
    depends_on:
      - backend

# Define a Docker network named app-network
networks:
  app-network:

# Define a Docker volume named postgresdata
volumes:
  postgresdata:

~~~

This is how I created the docker compose to link the application which will take care of the building of the image and allocating the relavent ports for out images.

This code takes the 3 stages of our tiers, backend, database and the http connection and basically have them organised in order to build and run the containers.
And as you cna see I have connected all the 3 architectures to one network(app-network) to be able to communicate.

## Publish
 
My dockerhub username: deghostt

Before tagging and pushing it is important to login to the docker using "docker login".
Tagging my images on docker desktop to be pushed to dockerhub.

This code will tag the exsisting images named httpd, backend and database and will rename them with the the username and the extension I give. This will make the image be able to be shared.
~~~bash
docker tag httpd deghostt/tpl-httpd:1.0
docker tag backend deghostt/tpl-backend:1.0
docker tag database deghostt/tp1-badabase:1.0
~~~


Pushing to dockerhub
~~~bash
docker push deghostt/tpl-httpd:1.0
docker push deghostt/tpl-backend:1.0
docker push deghostt/tp1-database:1.0
~~~

After pushing the images with the above method, it was possible to see the images online on the github account. It shows 3 different images repos with names: tpl-httpd, tpl-backend and tpl-database


##### Why do we put our images into an online repo?

Putting Docker images into an online repository provides collaboration, accessibility, versioning, and security benefits. This will also help to have an additional layer of safety and backup for your images, compared to having them only on your local machine.