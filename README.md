# Ansible Integration in Terraform

## Project Overview
This project demonstrates seamless integration of **Ansible** and **Terraform** for provisioning and configuring cloud infrastructure on AWS. Using Terraform to create resources, such as an EC2 instance, and automating configuration with Ansible, we ensure a streamlined DevOps process.

### Key Features:
- Automatically execute Ansible playbooks after Terraform provisions resources.
- Install and configure Docker and Docker Compose on an EC2 instance.
- Create and manage Docker containers using a predefined `docker-compose.yaml` file.
- Secure server setup by creating a new Linux user with appropriate permissions.

## Technologies Used
- **Terraform**: Infrastructure as Code (IaC) for provisioning AWS resources.
- **Ansible**: Configuration management and automation.
- **AWS**: Cloud platform for hosting infrastructure.
- **Docker**: Containerization platform.
- **Linux**: Operating system for EC2 instances.


## Ansible Playbook: `deploy-docker-new-user.yaml`
This playbook is divided into several plays, each responsible for a specific task in server configuration:

### 1. **Wait for SSH Connection**
Ensures the EC2 instance is ready for SSH connections before proceeding.
```yaml
- name: Wait for SSH connection
  hosts: all
  gather_facts: False
  tasks:
    - name: Ensure SSH port open
      wait_for:
        port: 22
        delay: 10
        timeout: 100
        search_regex: OpenSSH
        host: "{{ (ansible_ssh_host|default(ansible_host))|default(inventory_hostname) }}"
      vars:
        ansible_connection: local
        ansible_python_interpreter: /usr/bin/python3
```

### 2. **Install Docker**
Installs and starts the Docker daemon.
```yaml
- name: Install Docker
  hosts: all
  become: yes
  tasks:
    - name: Install Docker
      yum:
        name: docker
        update_cache: yes
        state: present
    - name: Start docker daemon
      systemd:
        name: docker
        state: started
```

### 3. **Create a New Linux User**
Creates a new user with appropriate group memberships.
```yaml
- name: Create new Linux user
  hosts: all
  become: yes
  tasks:
    - name: Create new Linux user
      user:
        name: irschad
        groups: adm,docker
```

### 4. **Install Docker Compose**
Downloads and installs Docker Compose for the appropriate architecture.
```yaml
- name: Install Docker Compose
  hosts: all
  become: yes
  become_user: irschad
  tasks:
    - name: Create docker-compose directory
      file:
        path: ~/.docker/cli-plugins
        state: directory
    - name: Get architecture of remote machine
      shell: uname -m
      register: remote_arch
    - name: Install docker-compose
      get_url:
        url: "https://github.com/docker/compose/releases/latest/download/docker-compose-linux-{{ remote_arch.stdout }}"
        dest: ~/.docker/cli-plugins/docker-compose
        mode: +x
```

### 5. **Start Docker Containers**
Copies the Docker Compose configuration file, logs into Docker Hub, and starts containers.
```yaml
- name: Start Docker Containers
  hosts: all
  become: yes
  become_user: irschad
  vars_files:
    - project-vars
  tasks:
    - name: Copy Docker Compose file
      copy:
        src: /Users/irschad/java-mysql-project/docker-compose-full.yaml
        dest: /home/irschad/docker-compose.yaml
    - name: Docker login
      docker_login:
        username: irschad
        password: "{{ docker_password }}"
    - name: Start containers from compose
      community.docker.docker_compose_v2:
        project_src: /home/irschad
```

## Terraform Configuration

### Key Components in `main.tf`
1. **AWS VPC and Subnet Creation**:
   ```hcl
   resource "aws_vpc" "myapp-vpc" {
     cidr_block           = var.vpc_cidr_block
     enable_dns_hostnames = true
     tags = {
       Name : "${var.env_prefix}-vpc"
     }
   }
   ```

2. **EC2 Instance Provisioning**:
   ```hcl
  resource "aws_instance" "app-server" {
    ami           = data.aws_ami.latest-amazon-linux-image.id
    instance_type = var.instance_type
  
    subnet_id              = aws_subnet.myapp-subnet-1.id
    vpc_security_group_ids = [aws_default_security_group.default-sg.id]
    availability_zone      = var.avail_zone
  
    associate_public_ip_address = true
    key_name                    = aws_key_pair.ssh-key.key_name
  
    tags = {
      Name : "${var.env_prefix}-server"
    }
  }
  
  resource "null_resource" "configure_server" {
    provisioner "local-exec" {
      working_dir = "/Users/irschad/ansible"
      command     = "ansible-playbook --inventory ${aws_instance.app-server.public_ip}, --private-key ${var.ssh_private_key} --user ec2-user deploy-docker-new-user.yaml"
    }
  }
   ```

### Timing Issue Fix
To handle the potential timing issue, the `wait_for` module in the Ansible playbook ensures the EC2 instance is ready before proceeding.

## How to Run
1. Clone the repository and navigate to the Terraform directory:
   ```bash
   cd terraform
   ```

2. Initialize Terraform:
   ```bash
   terraform init
   ```

3. Apply the Terraform configuration:
   ```bash
   terraform apply --auto-approve
   ```

4. Terraform will provision the infrastructure, and Ansible will automatically configure the server.

## Outputs
- Public IP address of the EC2 instance.
- AMI ID used for the instance.

## Conclusion
This project automates infrastructure provisioning and configuration, showcasing a powerful combination of Terraform and Ansible to streamline deployment processes in a DevOps pipeline.
