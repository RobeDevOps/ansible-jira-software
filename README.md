## Install jira-software using Ansible
Jira installation required several components in order to get this working.

Ansible Playbook
----------------

All the JIRA requirements and configurations were possible using this [Ansible](https://www.ansible.com/)

This orchestration were develop based on Atlassian documentation and forum questions.

The ansible-jira-software playbook contains 3 roles:

- **user**: create the dedicated system user for jira.
- **postgresql**: install and configure all the database requirements
- **unattended_mode**: install Jira as unattended mode reusing the the response.varfile configuration.

Before, all the requirements

EC2 Details
-------------------------
- **Type**: t2.xlarge
- **AMI**:  ami-011b3ccf1bd6db744
- **Arch**: x86_64
- **OS**:  RHEL-7.6
- **Virt**: HVM

System Requirements
--------------------
1. Install python 2.7 or 3.x in the orchestration server and target hosts due python is required on every node
2. Install Ansible	2.7 only on orchestration server.

Jira Database requirements
--------------------------------
The following requirements were developed using the **postgresql** role.

*Install postgresql server and packages dependencies*

**Description**: Install postgreql-<version>. In this case the version used is 9.6 but any previous version (supported by Jira) should be well installed using the playbook tasks

**ansible task**: ```postgres/tasks/install_postgres.yml```
_________________________________
*Create and Start postgresql-<version>.service service*

**Description:** Create the service as systemd service and allow easily to start/stop/restart the service and enabled for failover use cases.

**ansible task**: ```postgresql/tasks/create_service.yml```

_________________________________
*Configure postgresql.conf and pg_hba.conf file*

**Description**: There are two main files:
    1. postgresql.conf: enable to listen any connections. For production solutions should be ip address
    2. pg_hba.conf: required to allow connection for ip address also.

**ansible task**: ```postgresql/tasks/create_files_configuration.yml```
_________________________________
*Create JIRA database is and database user*

**Description**: This task create the Jira database and user. This values are required by the dbconfig.xml configuration file and it is required to skip the steps of database configuration in the web browser wizard. The user is granted with the permission of SELECT, INSERT, UPDATE, DELETE.
*psycopg2 is a package required by Ansible in order to use the postgresql_db module*

**ansible task**: ```postgresql/tasks/create_and_configure_db.yml```
_________________________________
*Create the dbconfig.xml file*

**Description**: In order to skip the database setup properties from the wizard the dbconfig.xml files contains all the required information for the database. (this can be done by unattended_mode role)

**ansible task**: ```postgresql/tasks/create_dbconfig_xml.yml```
_________________________________
Postgres Role
----------------------------
All these tasks are defined in **main.yml** file 

```yaml
---
- import_tasks: install_postgres.yml
- import_tasks: create_service.yml
- import_tasks: create_files_configuration.yml
- import_tasks: create_and_configure_db.yml
- import_tasks: create_dbconfig_xml.yml
```

And the role is used in the **main.yml** playbook as

```yaml
- hosts: postgres
  become: Yes
  gather_facts: Yes
  roles:
    - user
    - { role: postgresql, tags: ['psql'] }
```
_________________________________

Jira Server requirements and Implementation
--------------------------------------

The whole playbooks was developed using the Atlassian documentation for unattended and linux installation described in the links:

1. [Installing Jira applications on Linux](https://confluence.atlassian.com/adminjiraserver/installing-jira-applications-on-linux-938846841.html)
2. [Unattended installation](https://confluence.atlassian.com/adminjiraserver/unattended-installation-938846846.html)


Based on those links the **unattended_mode** role was developed:

*Create system jira user*

**Description**: This task creates a dedicated user **jira user** that allows to run the service and own the JIRA directories. The user is created by the ansible role: **user** but the specification is defined under the ```<jira_install_dir>/bin/user.sh``` file with the **unattended_mode** role. 
*The user role is executed in conjunction posgtres also due the ```<jira_home_directory>``` is required when ```dbconfig.xml``` is created. 

**ansible_tasks**:

| Role            | Task                                          |
|-----------------|-----------------------------------------------|
| user            | ```user/tasks/main.yml```                     |
| unattended_mode | ```unattended_mode/tasks/set_jira_user.yml``` |
_________________________________

*Create JIRA directories*

**Description**: JIRA requires two main directory: 
1. ```<jira_install_directory>```: where all the binaries and default configuration are hosted. 
2. ```<jira_home_directory>```: where data, logs, cache, indexes, db and custom configuration is hosted like ```jira-application.properties```, ```jira-config.properties``` and ```dbconfig.xml``. This directory needs to be owned by JIRA user created by **user** role.

**ansible task**: ```unattended_mode/tasks/create_jira_directories.yml```
_________________________________
*Install JIRA as unattended_mode*

What am I avoiding here:

1. Install from **Binary**: You can specify all the configuration from the command prompt. It is easy but still manual process and it requires human inputs.
2. Install from **tar.gz file**: You have to know where to configure all the files manually. This is an advance configuration and really need to know where the changes are required. ( can be automated )
3. Install using as unattended mode (used in this solution) : You need to install at least once JIRA and then the configuration was generated, then the configuration can be reused using the ```.install4j/response.varfile``` file. *This is the solution used in this playbook in order to automate mostly and avoid the web ui browser wizard steps*.

See an example of the ```response.varfile``` content:

```bash
launch.application$Boolean=false
rmiPort$Long=jira_server_port
app.jiraHome=jira_home_dir
app.install.service$Boolean=false
existingInstallationDir=jira_install_dir
sys.confirmedUpdateInstallationString=false
sys.languageId=en
sys.installationDir=jira_install_dir
executeLauncherAction$Boolean=true
httpPort$Long=jira_connector_port
portChoice=custom
```
**ansible tasks**: ```unattended_mode/tasks/unattended_installation.yml```

_________________________________
*Ensure all the directories are owner by JIRA system user*:

**Description**: It is mandatory to ensure and grant the right permissions to the ```<jira_install_directory>``` and ```<jira_home_directory>``` directories to avoid any error when service starts.	
**ansible task**: ```unattended_mode/tasks/ensure_jira_owner.yml```

_________________________________
*Create jira service*

**Description**: Create a systemd service for JIRA to allow an easily way to start/stop/restart the service. Also enabled feature allows to start automatically in case of node failover.

**ansible task**: ```unattended_mode/tasks/create_jira_service.yml```

_________________________________
*Configure Application properties*	

**Description**: In order to skip the application properties from the web browser wizard, it task defines all these properties in the jira-config.properties file. This information should be loaded once Jira is restarted (**so far is not working and still working on this step**).
The file is hosted at ```<jira_home_directory>/jira-config.properties```.

See example below of this property file
```bash
jira.home=jira_home_dir
jira.title=Jira Ansible
jira.baseurl=http://jira.ansible.com
jira.mode=private
```

**ansible task**: ```unattended_mode/tasks/create_jira_config_properties.yml```

_________________________________
unattended_mode Role
----------------------------

All these tasks are defined in **main.yml** file   

```yaml
---
- import_tasks: create_jira_directories.yml
- import_tasks: unattended_installation.yml
- import_tasks: set_jira_user.yml
- import_tasks: ensure_jira_owner.yml
- import_tasks: create_jira_service.yml
- import_tasks: create_jira_config_properties.yml
```

And the role is under the main.yml file as:

```yaml
- hosts: jira
  become: Yes
  gather_facts: Yes
  roles:
    - user
    - unattended_mode
```

_________________________________
Putting altogether
---------------------------

The final playbooks looks like:

```yaml
---
- hosts: postgres
  become: Yes
  gather_facts: Yes
  roles:
    - user
    - { role: postgresql, tags: ['psql'] }
    
- hosts: jira
  become: Yes
  gather_facts: Yes
  roles:
    - user
    - unattended_mode
```

And it is possible to run this one using the command:

```ansible-playbook main.yml -i inventory/hosts.ini``` 

More verbose output

```bash
ansible-playbook main.yml -i inventory/hosts.ini -v
ansible-playbook main.yml -i inventory/hosts.ini -vv
ansible-playbook main.yml -i inventory/hosts.ini -vvv
``` 
_________________________________
**There are more configuration files**: There other configuration files that provides more flexibility and automation but those are not implemented yet in this solution:

**```jira-application.properties```**: allows to defined jira_jome.
**```server.xml```**: configuration related with ssl and other kind of advance features.
**```setenv.sh```**: JAVA args and memory setup and more.

All these can be added under **unattended_mode** role or a more general role that can be reused on any installed JIRA solution. 
