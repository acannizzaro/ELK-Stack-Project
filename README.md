# Automated ELK Stack Deployment

This document contains the following details:
- Description of the Topology
- ELK Configuration
- Access Policies
- Beats in Use
- Machines Being Monitored
- How to Use the Ansible Build


### Description of the Topology
This repository includes code defining the infrastructure below. 

![Network Topology](/Images/ELK%20Stack.png)

The main purpose of this network is to expose a load-balanced and monitored instance of DVWA, the "D*mn Vulnerable Web Application"

Load balancing ensures that the application will be highly **available**, in addition to restricting **inbound access** to the network. The load balancer ensures that work to process incoming traffic will be shared by both vulnerable web servers. Access controls will ensure that only authorized users — namely, ourselves — will be able to connect in the first place.

Integrating an ELK server allows users to easily monitor the vulnerable VMs for changes to the **file systems of the VMs on the network**, as well as watch **system metrics**, such as CPU usage; attempted SSH logins; `sudo` escalation failures; etc.

The configuration details of each machine may be found below.

| Name     |   Function  | IP Address | Operating System |
|----------|-------------|------------|------------------|
| Jump Box | Gateway     | 10.1.0.4   | Linux            |
| Web 1    | Web Server  | 10.1.0.5   | Linux            |
| Web 2    | Web Server  | 10.1.0.6   | Linux            |
| Web 3    | Web Server  | 10.1.0.7   | Linux            |
| ELK      | Monitoring  | 10.0.0.4   | Linux            |

In addition to the above, Azure has provisioned a **load balancer** in front of all Webservers. The load balancer's targets are organized in the following availability zone:
- **Web Set**: Web 1 + Web 2 + Web 3


## ELK Server Configuration
The ELK VM exposes an Elastic Stack instance. **Docker** is used to download and manage an ELK container.

Rather than configure ELK manually, we opted to develop a reusable Ansible Playbook to accomplish the task.


To use this playbook, one must log into the Jump Box and ssh into the ansible container, and configure the hosts file for ansible to properly execute playbooks on the servers we want.
run `nano /etc/ansible/hosts` and add a section for elkservers

[elkservers]
[private server ip] ansible_python_interpreter=/usr/bin/python3

then issue: `ansible-playbook elk-playbook.yml` . 
This runs the `elk-playbook.yml` playbook on the `elkservers` hosts.

### Access Policies
The machines on the internal network are _not_ exposed to the public Internet. 

The **jump box** machine and **ELK Stack** machine can accept connections from the Internet. Access to these machines is only allowed from the IP address `73.66.116.101`
- **Note**: This will change, as it is based off of the current local machine IP.

Machines _within_ the network can only be accessed by **each other**. The DVWA 1, DVWA 2, and DVWA 3 VMs send traffic to the ELK server.

A summary of the access policies in place can be found in the table below.

| Name     | Publicly Accessible | Allowed IP Addresses 		    | Allowed Ports
|----------|---------------------|------------------------------|------------------
| Jump Box | Yes                 | 73.66.116.101       	    		| 22
| ELK      | Yes                 | 10.1.0.1-254  & 73.66.116.101| 5601
| DVWA 1   | No                  | 10.0.0.1-254         		    | 22 80
| DVWA 2   | No                  | 10.0.0.1-254         	    	| 22 80
| DVWA 2   | No                  | 10.0.0.1-254         	    	| 22 80
### Elk Configuration

Ansible was used to automate configuration of the ELK machine. No configuration was performed manually, which is advantageous because it allows us to configure as many machines as we want by simply reconfiguring the hosts file and running the playbook.

The playbook implements the following tasks:
- Install Docker.io
- Install Python3-pip
- Install docker with pip
- Increase memory allocation for ELK
- Pull and start the ELK docker image with the proper virtual ports
- Ensure Docker starts on system reboots

The following screenshot displays the result of running `docker ps` after successfully configuring the ELK instance.

![Docker PS Output](/Images/docker_ps.PNG)

The playbook is duplicated below.
![ELK Playbook](/playbooks/elk-playbook.yml) 

```yaml
---  
  - name: ELK Server Setup                                                            
    hosts: elkservers                                     
    become: true                                                                  
    tasks:                                                                          
      - name: Install docker.io                                                       
        apt:                                                                            
          name: docker.io                                                               
          update_cache: yes                                                             
          state: present                                                            
      - name: Install python3-pip                                                     
        apt:                                                                            
          name: python3-pip                                                             
          state: present                                                            
      - name: Install docker with pip                                                 
        pip:                                                                            
           name: docker                                                                  
           state: present                                                             
      - name: Use more memory                                                         
        sysctl:                                                                         
          name: vm.max_map_count                                                        
          value: "262144"                                                               
          state: present                                                                
          reload: yes                                           
      - name: Pull default Docker image                                               
        docker_container:                                                               
          name: elk                                                                     
          image: sebp/elk:761                                                           
          state: started                                                                
          restart_policy: always                                                        
          ports:                                                                          
            - 5601:5601                                                                   
            - 9200:9200                                                                   
            - 5044:5044                                                             
      - name: Enable docker service on restart                                        
        systemd:                                                                        
          name: docker                                                                  
          enabled: yes
```

### Target Machines & Beats
This ELK server is configured to monitor the Web 1, Web 2, and Web 3 VMs, at `10.1.0.5`, `10.1.0.6`, and `10.1.0.7` respectively.

We have installed the following Beats on these machines:
- Filebeat
- Metricbeat

These Beats allow us to collect the following information from each machine:
- **Filebeat**: Filebeat detects changes to the filesystem. Specifically, we use it to collect Apache logs.
- **Metricbeat**: Metricbeat detects changes in system metrics, such as CPU usage. We use it to detect SSH login attempts, failed `sudo` escalations, and CPU/RAM statistics.


The playbook below installs Metricbeat on the target hosts. The playbook for installing Filebeat is not included, but looks essentially identical — simply replace `metricbeat` with `filebeat`, and it will work as expected.

![Metricbeat Playbook](/playbooks/metricbeat-playbook.yml)

```yaml
---
  - name: Installing and Launch Metricbeat
    hosts: webservers
    become: yes
    tasks:
## Check for version updates before running
    - name: Download metricbeat .deb file
      command: curl -L -O https://artifacts.elastic.co/downloads/beats/metricbeat/metricbeat-7.6.1-amd64.deb
    - name: Install metricbeat .deb
      command: dpkg -i metricbeat-7.6.1-amd64.deb
    - name: Drop in metricbeat.yml
      copy:
        src: /etc/ansible/files/metricbeat-config.yml
        dest: /etc/metricbeat/metricbeat.yml
    - name: Enable and Configure Docker Module
      command: metricbeat modules enable docker
    - name: Setup metricbeat
      command: metricbeat setup
    - name: Start metricbeat service
      command: service metricbeat start
```

### Using the Playbooks
In order to use the playbooks, you will need to have an Ansible control node already configured. We use the **jump box** for this purpose.

To get an Ansible Container running on your **jump box** use the following commands:
```bash
$ sudo apt install docker.io
$ sudo systemctl start docker
$ sudo docker pull cyberxsecurity/ansible:latest
$ sudo docker run -ti cyberxsecurity/ansible:latest bash
$ exit
```


To use the playbooks, we must perform the following steps:
- Copy the playbooks to the Ansible Control Node
- Run each playbook on the appropriate targets

The easiest way to copy the playbooks is to use Git:

```bash
$ cd /etc/ansible
$ mkdir files
$ mkdir configs
# Clone Repository + IaC Files
$ git clone https://github.com/acannizzaro/ELK-Stack-Project.git
# Move Playbooks and hosts file Into `/etc/ansible`
$ cp ELK-Stack-Project/playbooks/* ./files
$ cp ELK-Stack-Project/files/* ./configs
```

This copies the playbook files to the correct place.

Next, you must edit the `hosts` file to specify which VMs to run each playbook on. Run the commands below:

```bash
$ cd /etc/ansible
$ nano hosts 
[webservers]
10.1.0.5 ansible_python_interpreter=/usr/bin/python3
10.1.0.6 ansible_python_interpreter=/usr/bin/python3
10.1.0.7 ansible_python_interpreter=/usr/bin/python3

[elkservers]
10.0.0.4 ansible_python_interpreter=/usr/bin/python3

```

After this, the commands below run the playbook:

 ```bash
 $ cd /etc/ansible
 $ ansible-playbook elk_playbook.yml
 $ ansible-playbook dvwa_playbook.yml
 $ ansible-playbook filebeat_playbook.yml 
 $ ansible-playbook metricbeat_playbook.yml 
 ```

To verify success, wait five minutes to give ELK time to start up. 

Then in your browser, navigate to: `http://[ELK public IP]:5601/app/kibana`. This is the address of Kibana. If the installation succeeded, this should bring you to the kibana home page.
To verify Filebeat is running, click "Add Log Data", scroll down to "System Logs", then scroll to the bottom and click "Check Data". If everything was done correctly Filebeat should be sending data from the Webservers to your ELK server.
Metricbeat can also be verified by clicking "Add Metric Data", scolling to "Docker", then scrolling again to the bottom to "Check Data".
