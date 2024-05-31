# Discover Ansible TP3

This tutorial aims to guide you through the process of installing and deploying an application automatically using Ansible. We learnt to manage inventories, create and execute playbooks, utilize roles, and deploy your application using Docker containers.

### Files
 - inventories: contains the id_rsa file for the ansible connection and the setup file to connect to ansible.
 - roles: contains the roles for the application. which was loaded after running the code - **ansible-galaxy init roles/docker**

## Steps

### 1. Setting Up Ansible Inventory

Create a project-specific inventory in your project directory:

1. Create an `ansible` directory **my-project/ansible/inventories/setup.yml**:

~~~bash
all:
 vars:
   ansible_user: centos
   ansible_ssh_private_key_file: /path/to/private/key
 children:
   prod:
     hosts: hostname or IP
~~~

TO test the connection we used the following code:

~~~bash
ansible all -i inventories/setup.yml -m ping
~~~

## Advanced Playbook

~~~bash
# This playbook is intended to run on all hosts.
- hosts: all
  gather_facts: false   # Disables gathering of facts to speed up playbook execution.
  become: true          # Escalates privileges to perform tasks that require root access.

  # Tasks to install and configure Docker on CentOS.
  tasks:

  # Install the device-mapper-persistent-data package, which is a prerequisite for Docker.
  - name: Install device-mapper-persistent-data
    yum:
      name: device-mapper-persistent-data
      state: latest

  # Install the lvm2 package, another prerequisite for Docker.
  - name: Install lvm2
    yum:
      name: lvm2
      state: latest

  # Add the Docker repository to yum using the yum-config-manager command.
  - name: add repo docker
    command:
      cmd: sudo yum-config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo

  # Install Docker Community Edition (CE).
  - name: Install Docker
    yum:
      name: docker-ce
      state: present

  # Install Python 3, which is required to use pip for installing Python packages.
  - name: Install python3
    yum:
      name: python3
      state: present

  # Install the Docker Python module using pip3. This is useful for managing Docker containers via Python scripts.
  - name: Install docker with Python 3
    pip:
      name: docker
      executable: pip3
    vars:
      ansible_python_interpreter: /usr/bin/python3  # Specifies the Python interpreter to use.

  # Ensure the Docker service is running.
  - name: Make sure Docker is running
    service:
      name: docker
      state: started
    tags: docker


~~~


## Deploy your App

**create_network**

~~~yaml
# Creating the network
- name: Create Docker network
  docker_network:
    name: app-network
  vars:
    ansible_python_interpreter: /usr/bin/python3
~~~

- This task creates a Docker network named `app-network` using the `docker_network` module. The specified Python interpreter is used to ensure compatibility with Docker modules.

**install_docker**

~~~yaml
- name: Install Python 3 and pip3
  command: yum install -y python3 python3-pip
  args:
    creates: /usr/bin/python3
~~~

- This task installs Python 3 and pip3 using the `yum` package manager. The `creates` argument ensures that the command runs only if Python 3 is not already installed.

**launch_app**

~~~yaml
# Launching the backend

- name: Launch application container
  docker_container:
    name: multistage_app2
    image: deghostt/tp1-backend:latest
    networks:
      - name: app-network
    ports:
      - "8080:8080"
  vars:
    ansible_python_interpreter: /usr/bin/python3
~~~

- This task launches the backend application container named `multistage_app2` from the specified Docker image. It connects the container to the `app-network` and maps port 8080 on the host to port 8080 on the container.

**launch_database**

~~~yaml
# Launching the database container

- name: Launch database container
  docker_container:
    name: postgres
    image: deghostt/tp1-database:latest
    env:
      POSTGRES_DB: db
      POSTGRES_USER: usr
      POSTGRES_PASSWORD: pwd
    networks:
      - name: app-network
  vars:
    ansible_python_interpreter: /usr/bin/python3
~~~

- This task launches the PostgreSQL database container named `postgres` using the specified Docker image. Environment variables for the database configuration are provided, and the container is connected to the `app-network`.

**launch_proxy**

~~~yaml
# Launching the proxy (frontend)

- name: Launch frontend container
  docker_container:
    name: launch_proxy
    image: deghostt/tp1-httpd:latest
    state: started
    ports:
      - "80:80"
    networks:
      - name: app-network
    env:
      API_URL: "http://backend:8080/api"
      DB_HOST: "postgres"
      DB_USER: "usr"
      DB_PASSWORD: "pwd"
      DB_NAME: "db"
    user: root
    privileged: true
  vars:
    ansible_python_interpreter: /usr/bin/python3
~~~

- This task launches the frontend proxy container named `launch_proxy` from the specified Docker image. It starts the container, maps port 80 on the host to port 80 on the container, connects it to the `app-network`, and sets necessary environment variables. The container runs as the root user with privileged access.


## Continuous Deployment

For this part we have to add a part on the main.yml to deploy the application. 

~~~bash
deploy:
    needs: build-and-push-docker-image
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v2.5.0

      - name: Setup SSH and known hosts
        uses: webfactory/ssh-agent@v0.5.4
        with:
          ssh-private-key: ${{ secrets.SSH_PRIVATE_KEY }}

      - name: Install Ansible
        run: |
          sudo apt-get update
          sudo apt-get install -y ansible

      - name: Disable host key checking
        run: echo "ANSIBLE_HOST_KEY_CHECKING=False" >> $GITHUB_ENV

      - name: Deploy to production server
        run: |
          ansible-playbook -i my-project/ansible/inventories/setup.yml my-project/ansible/playbook_app.yml

~~~

1. **Checkout code**: This step checks out the code from the repository.

2. **Setup SSH and known hosts**: SSH authentication is set up using the provided private key stored in the repository secrets.

3. **Install Ansible**: Ansible is installed on an Ubuntu environment to facilitate the deployment process.

4. **Disable host key checking**: Host key checking is disabled to prevent interruptions during deployment.

5. **Deploy to production server**: An Ansible playbook (`playbook_app.yml`) is executed to deploy the project to the production server.

## Important Keys

- **needs**: Ensures the preceding task (`build-and-push-docker-image`) is completed before deployment.
- **secrets.SSH_PRIVATE_KEY**: Variable used to securely authenticate with the production server.

## Usage

To use this workflow, ensure that your project structure aligns with the provided Ansible playbook (`playbook_app.yml`) and inventory setup (`inventories/setup.yml`). Additionally, make sure to configure the necessary secrets in the repository settings, including `SSH_PRIVATE_KEY` for SSH authentication.