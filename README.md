## Steps

1. Used **Google Cloud Platform** as my infrustruture.
2. Going to use **gcloud**,**Terraform**,**Ansible**,**docker** to build up a nodejs web server.
There was a **ubuntu 16.04** virtual machine on my laptop. 
3. Used this vm worked as my managed node to control my instance on Google Cloud Platform
Generated SSH key pair.
4. Created a credential json file from GCP. 
5. Terraform needs credential file to build up instance,firewall, external ip.
6. Terraform copy SSH public key into instance when instance created.
6. Ansible used SSH private key login into instances on GCP, and installed package, installed docker, built up nodejs image, run nodejs container.


## Source Code 

Under devops/ directory.

```sh
|-- hosts        # inventory of ansible
|-- lucasko.pub  # ssh public key
|-- lucasko      # ssh private key
|-- lucasko-credential.json  #service account credential from GCP
|-- main.tf      # terraform file for creating instance,ip,firewall
|-- nodejs       # Dockerfile with node js source code
|   |-- Dockerfile
|   |-- package.json
|   `-- server.js
|-- playbook.yml    # playbook of ansible for installing package,runing docker
```


## My Environment

1. Google Cloud Platform
2. Ubuntu 16.04 (My Managed Node)
3. gcloud：Version is Google Cloud SDK 220.0.0
4. Terraform v0.11.8
5. Ansible：2.7.0

Notes：
You need to apply for are credential.json of google service account from GCP.
You can copy my code into root directory  **/devops**


## Set up Credential & SSH Key

1. Create User Credential of GCP：
	* GCP → Credentials → Service Account key
	* Service Account Name：**lucasko**
	* Select Role:
		* Computer Admin
		* Computer Network Admin
		* Service Acount User
 
 You will get a .json file ,which contains details of credential. I rename it to **lucasko-credential.json**

2. Generate SSH Key

key name should be the same as your google service account name.

```sh
ubuntu@ubuntu-vm:/devops$ ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/home/ubuntu/.ssh/id_rsa): lucasko
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in lucasko.
Your public key has been saved in lucasko.pub.
The key fingerprint is:
SHA256:wXZ1NheinMof8Z9FEV/IL+bfzOVIuKKHFC3AYqdwRNc ubuntu@ubuntu-vm
The key's randomart image is:
+---[RSA 2048]----+
|   oo...    .o+==|
|  . +.+.E ..oo+o+|
|   + + .+..=   .o|
|    .  .+oo o o..|
|        S= . = ..|
|        . . o + +|
|       . . . o Oo|
|        . o . . *|
|        .o .     |
+----[SHA256]-----+

ubuntu@ubuntu-vm:/devops$ ls -l
total 28
-rw-rw-r-- 1 ubuntu ubuntu  151 Oct 11 10:09 hosts
-rw-rw-r-- 1 ubuntu ubuntu 1066 Oct 11 10:37 main.tf
drwxrwxr-x 2 ubuntu ubuntu 4096 Oct 11 10:17 nodejs
-rw-rw-r-- 1 ubuntu ubuntu 1318 Oct 11 10:18 playbook.yml
-rw------- 1 ubuntu ubuntu 1679 Oct 11 11:36 lucasko
-rw-r--r-- 1 ubuntu ubuntu  398 Oct 11 11:36 lucasko.pub
-rw-rw-r-- 1 ubuntu ubuntu 2313 Oct 11 11:01 lucasko-credential.json
```

1. lucasko：private key 
2. lucasko.pub：public key

## Run Terraform

There is a file named  main.tf. Follwing details of **main.tf** file ：

1. Create instance on Google Cloud Platform
2. Setting metadata for putting public key into instance
3. Apply for a public IP
4. Set up public IP on your instance
5. Apply for a Firewall, open 80 port
6. Set up Firewall on your instance



Check credential before you running terraform

```sh
cd /devops
vim .env
```



**In .env file, Please use your own credential,project id**

```sh
export GOOGLE_APPLICATION_CREDENTIALS="/devops/lucasko-credential.json"

export GCLOUD_PROJECT="xxxx-218917"

export GCLOUD_REGION="asia-east1-b" 
GCLOUD PROJECT is your GCP project id.
```


Run Terraform

```sh
source .env

terraform init

terraform plan

terraform apply
```

## Run Ansible

There is a file named  playbook-yml ; Follwing details of playbook-yml  file.

1. Install Docker
2. Install easy_install
3. Install pip：It is used for module of docker of ansible 
4. Copy node js source into instance in GCP
5. Docker pull image
6. Docker build new image for nodejs web server
7. Run Docker of nodejs web server

Check Ansible Hosts File

```sh 
vim hosts
```

content of hosts

```sh 
[webservers]
35.221.207.117 ansible_ssh_port=22 ansible_ssh_user=lucasko ansible_ssh_private_key_file=/devops/lucasko
```

1. change IP to your public instance IP.

2. /devops/lucasko is private key for using ssh into your instance


Run Ansible

```sh
ansible-playbook -i hosts playbook.yml
```

Check Result

```sh
ubuntu@ubuntu-vm:/devops$ curl 35.221.207.117
Hello world
```

After I login into instance, I also show up the containers of docker 

```sh
ubuntu@ubuntu-vm:/devops$ ssh -i lucasko lucasko@35.221.207.117
Welcome to Ubuntu 16.04.5 LTS (GNU/Linux 4.15.0-1021-gcp x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  Get cloud support with Ubuntu Advantage Cloud Guest:
    http://www.ubuntu.com/business/services/cloud

11 packages can be updated.
3 updates are security updates.

New release '18.04.1 LTS' available.
Run 'do-release-upgrade' to upgrade to it.


Last login: Thu Oct 11 11:55:40 2018 from 109.246.71.122
lucasko@devops-tech-test:~$ sudo docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS                  NAMES
e0a70c4cbe9d        nodejs:v1           "npm start"         About an hour ago   Up About an hour    0.0.0.0:80->8080/tcp   helloworld
```

## Details of Terraform

### Create Instance

```sh
resource "google_compute_instance" "test" {

  name         = "devops-tech-test"
  zone         = "asia-east1-b"
  machine_type = "n1-standard-1"

  tags = [ "my-tag"]

  boot_disk {
        initialize_params {
            image = "ubuntu-1604-lts"
            type  = "pd-ssd"
            size = 30
        }
    }

  network_interface {
    network = "default"

    access_config {

      nat_ip = "${google_compute_address.test-ip.address}"

                  }
  }
}
```
1. instance name is devops-tech-test.
2. nat_ip is used for setting external ip.
3. machine image is ubuntu 16.04
4. tags is used for mapping firewall setting.



### SSH Key
GCP offers metadata to let you putting your public key into instance.

```sh
  metadata {
    sshKeys = "${var.gce_ssh_user}:${file(var.gce_ssh_pub_key_file)}"
  }
```

user is lucasko, it should be the same as google service account.
gce_ssh_pub_key_file is location of public key on you managed node.

```sh
variable "gce_ssh_user" {
 default = "lucasko"
}
variable "gce_ssh_pub_key_file" {
 default = "/devops/lucasko.pub"
}
```


External IP
Apply for a public ip
resource "google_compute_address" "test-ip" {
  name = "my-ip"
  region = "asia-east1"
}


### Create Firewall

```sh
resource "google_compute_firewall" "test-firewall" {
  name    = "my-firewall"
  network = "default"

  allow {
    protocol = "icmp"
  }

  allow {
    protocol = "tcp"
    ports    = ["80","22"]
  }
  source_ranges = ["0.0.0.0/0"]
  source_tags = ["my-tag"]
}

```

1. open 80,22 ports.
2. source_range is all ip addresses.
3. source_tage is used for mapping firewall setting into instances

## Details of Ansible File

### Install Docker

```sh
- hosts: webservers
  become: true
  tasks:
  - name: Add Docker GPG key
    apt_key: url=https://download.docker.com/linux/ubuntu/gpg

  - name: Add Docker APT repository
    apt_repository:
      repo: deb [arch=amd64] https://download.docker.com/linux/ubuntu {{ansible_distribution_release}} stable

  - name: Install list of packages
    apt:
      name: "{{ item }}"
      state: present
      update_cache: yes
    with_items:
      - apt-transport-https
      - ca-certificates
      - curl
      - software-properties-common
      - docker-ce
```

### Install Packages for module of Ansible
Ansible offers a module called docker_container,docker_image to control you containers

You need to install following package before you use this module.

```sh
  - name: Easy Install
    apt:
      name: python-setuptools
      state: present

  - name: Install pip
    easy_install: 
     name: pip
     state: latest

  - name: pip install docker-py
    pip:
      name: docker-py
```
### Install npm for Node JS Webserver
copy nodejs source code into instance on GCP.
npm install based on package.json file.

```sh
  - name: pip install npm
    apt:
      name: npm

  - name:
    copy:
      src: ./nodejs
      dest: /

  - name: npm install based on package.json
    npm:
      path: /nodejs
```

### Run Docker
1. docker pull node:8
2. build up new image of node js web server
3. Run Docker of node js , 80 port

```sh
  - name: Docker Pull Image
    docker_image: 
      name: node
      tag: 8

  - name: Build Image for Node JS
    docker_image:
      path: /nodejs
      name: nodejs
      tag: v1

  - name: Run Docker for Node JS
    docker_container:
      name: helloworld
      image: nodejs:v1
      state: started
      restart: yes
      ports:
        - "80:8080"
``` 
