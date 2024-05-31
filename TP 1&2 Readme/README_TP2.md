# Discover Github Action TP2

This repository demonstrates a complete CI/CD pipeline setup using GitHub Actions. The goal is to automate the build, test, and deployment processes for a Java application using Maven. Additionally, the pipeline includes steps for code quality checks with SonarCloud and Docker image management.

## Prerequisites

Before starting this TP it is necessary to have the following:

 - Pushed the TP1 into the Github repository.
 - Docker Hub account
 - SonarCloud account
 - Java application with a Maven build setup


### Setup GitHub Actions
#### Step 1: Create Workflow Directory
Create a directory named .github/workflows in the root of your repository. This is where GitHub Actions looks for workflow files.

#### Step 2: Create the main.yml Workflow File
Create a file named **main.yml** inside **.github/workflows directory**. This file will define your CI/CD pipeline.

##### 2-1 What are testcontainers?
 - Testcontainers is a Java library that provides lightweight, throwaway instances of common databases, Selenium web browsers, or anything else that can run in a Docker container, for use in automated tests.

**main.yml**
~~~bash
# Define the name of the CI/CD pipeline
name: CI devops 2024

# Define the events that trigger the pipeline
on:
  # Trigger the pipeline on pushes to the 'main' branch
  push:
    branches: [main]
  # Trigger the pipeline on pull requests to the 'main' branch
  pull_request:
    branches: [main]

# Define the jobs that make up the pipeline
jobs:
  # Define the 'test-backend' job
  test-backend:
    # Define the virtual machine that the job runs on
    runs-on: ubuntu-22.04

    # Define the steps that make up the job
    steps:
      # Define the 'Checkout code' step
      - name: Checkout code
        # Use the 'actions/checkout' action to clone the repository and check out the 'main' branch
        uses: actions/checkout@v2.5.0

      # Define the 'Set up JDK 17' step
      - name: Set up JDK 17
        # Use the 'actions/setup-java' action to install and configure the Java Development Kit (JDK) version 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'

      # Define the 'Build and test with Maven' step
      - name: Build and test with Maven
        # Use the 'mvn clean verify' command to build the application and run both unit tests and integration tests
        run: mvn clean verify

~~~
This code structures the functions to be carried out wehn the code 
Important command: **mvn clean verify**

- This command will actually clear your previous builds inside your cache (otherwise your can have unexpected behavior because maven did not build again each part of your application), then it will freshly build each module inside your application, and finally it will run both Unit Tests and Integration Tests 

**Trigers**
 - push: The pipeline will run whenever there is a push to the main branch of the repository. This ensures that any changes to the codebase are automatically built, tested, and deployed.
 - pull_request: The pipeline will also run whenever there is a pull request to the main branch of the repository. This allows you to test and validate the changes in the pull request before merging them into the main branch.

Then we have the **pom.xml** in the directory of the **simple-api-student-main**.

This Maven dependency helps configuration sets up the testcontainers, jdbc, and postgresql dependencies for a Java application, with a scope of test for running unit tests and integration tests.

WIth the above **main.yml** file we can see the first branch **test-backen** test branch  in the **Actions** section in the github when the code is being pushed to the repo.

After this we will be improving our **main.yml** to be able to **build-and-push-docker-image** once the test is completed. this will be our seconf branch on the github action centre.

Section with the **build-and-push-docker-image**
~~~bash
build-and-push-docker-image:
    needs: test-backend
    runs-on: ubuntu-22.04
    steps:
      - name: Login to DockerHub
        run: echo ${{ secrets.DOCKER_PASSWORD }} | docker login -u ${{ secrets.DOCKER_USERNAME }} --password-stdin
      
      - name: Checkout code
        uses: actions/checkout@v2.5.0
      
      - name: Build image and push backend
        uses: docker/build-push-action@v3
        with:
          context: ./Backend/simple-api-student-main
          tags: ${{ secrets.DOCKER_USERNAME }}/tp1-backend:latest
          push: ${{ github.ref == 'refs/heads/main' }}

      - name: Build image and push database
        uses: docker/build-push-action@v3
        with:
          context: ./postgres
          tags: ${{ secrets.DOCKER_USERNAME }}/tp1-database:latest
          push: ${{ github.ref == 'refs/heads/main' }}

      - name: Build image and push httpd
        uses: docker/build-push-action@v3
        with:
          context: ./httpd
          tags: ${{ secrets.DOCKER_USERNAME }}/tp1-httpd:latest
          push: ${{ github.ref == 'refs/heads/main' }}

~~~

**Github secrets:** It is essential to set github secrets on the repo with docker hub pasword: **DOCKER_PASSWORD** and username:**DOCKER_USERNAME**. This enables these keys to be secure without having them plain on the files which is not good for the security.

 - This workflow configuration file defines a job called **build-and-push-docker-image** that runs after the **test-backend** job has completed successfully. The job runs on an Ubuntu 22.04 virtual machine and includes four steps. The first step logs in to Docker Hub using the **DOCKER_USERNAME** and **DOCKER_PASSWORD** secrets. The second step checks out the code using the actions/checkout action. The third and fourth steps use the docker/build-push-action action to build and push Docker images for the backend, database, and httpd services to Docker Hub. The context parameter specifies the directory containing the Dockerfile, and the tags parameter specifies the Docker image name and tag. The push parameter is set to true only if the workflow is triggered by a push to the main branch, which ensures that Docker images are only pushed to Docker Hub when the code is ready to be deployed.

