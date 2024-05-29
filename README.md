# DIscover Docker

## Database

#### Creating network
~~~bash
  docker network create app-network
~~~

#### Creating the database

~~~bash
  docker run -d --name postgres -v ${PWD}/data:/var/lib/postgresql/data --net=app-network -p 5432:5432 my-postgres:1.0
~~~

#### Connecting adminer to the database

~~~bash
docker run -p "8090:8080" --net=app-network --name=adminer -d adminer
~~~

## Backend API
#### Creating the backend API dockerfile
~~~bash


~~~

## Multistage build

#### Dockerfile
~~~bash
//Use an appropriate Java JRE base image

FROM eclipse-temurin:17-jre-alpine

// Set the working directory inside the container

WORKDIR /app

// Copy the compiled Java class into the container

COPY /src/main/java/Main.class .

// Command to run the Java program

CMD ["java", "Main"]
~~~



## Backend API

~~~bash
spring:
  jpa:
    properties:
      hibernate:
        jdbc:
          lob:
            non_contextual_creation: true
    generate-ddl: false
    open-in-view: true
  datasource:
    url: jdbc:postgresql://localhost:5432/db
    username: usr
    password: pwd
    driver-class-name: org.postgresql.Driver
management:
  server:
    add-application-context-header: false
  endpoints:
    web:
      exposure:
        include: health,info,env,metrics,beans,configprops


docker build -t apache-reverse-proxy .
docker run  --name Apache-reverse-proxy --network app-network -p 80:80 apache-reverse-proxy
~~~



## Link application
### Docker-compose

#### docker-compose.yml

~~~bash
 services:
    backend:
        container_name: backend
        build: .\Backend\simple-api-student-main\
        environment:
            DB_host: postgres
            DB_port: 5432
            DB_name: db
            DB_user: usr
            DB_mdp: pwd
        ports:
            
          - "8080:8080"
        networks:
          - app-network
        depends_on:
          - database

    database:
        container_name: database
        build: .\postgres\
        environment:
            POSTGRES_DB: db
            POSTGRES_USER: usr
            POSTGRES_PASSWORD: pwd
        volumes:
          - postgresdata:/var/lib/postgresql/data
        networks:
          - app-network

    httpd:
        container_name: httpd
        build: .\httpd\
        environment:
            BACKEND_host: backend
        ports:    
          - "80:80"
        networks:
          - app-network
        depends_on:
          - backend

networks:
    app-network:

volumes:
    postgresdata:

~~~