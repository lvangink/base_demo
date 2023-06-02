# Model-Driven Devops Repo

This is the base repo for Model-Driven devops.  It has the needed python requirements (requirements.txt) and
Ansible Collection (requirements.yml) to run a Model-Driven Devops pipeline.  It requires an ansible inventory and
certain envrionment variables to be set. It also assumes you are using Cisco Modeling Labs, GitLab, and NSO. Details on running
NSO and CML can be found by using the following links.


# Installing Tooling
## NSO
Installing in Kubernetes  
Installing in CML  
Installing in Docker  
Installing Locally  

## CML
Installing in AWS  
Installing in VMware  

## GitLab

# Installation
## Clone Repo
```
git clone git@github.com:model-driven-devops/mdd-base.git
```
Move to the mdd-base directory.
```
cd mdd-base
```

## Installing Dependancies

### Environmental Variables
The MDD tooling requires several environment variables. The first one required for base execution is:
```
export ANSIBLE_PYTHON_INTERPRETER=${VIRTUAL_ENV}/bin/python
export ANSIBLE_COLLECTIONS_PATH=./
```

### Python 
Next, it is highly recommended that you create a virtual environment to make it easier to install the dependencies without conflict:
```
python3 -m venv venv-mdd
. ./venv-mdd/bin/activate
```
Next, install the Python requirements via pip:
```
pip3 install -r requirements.txt
```

### Reactivate Virtual Environment
Reactivate virtual environment to ensure your shell is using the newly installed ansible.
```
deactivate
```
```
. ./venv-mdd/bin/activate
```

### Ansible Collections
```
ansible-galaxy collection install -r collections/requirements.yml
```

### Environment Variables
Set the environment variables listed below:

#### NSO
The TEST_REMOTE and PROD_REMOTE passwords are used for your device authentication.
```
export NSO_USERNAME=
export NSO_PASSWORD=
export TEST_REMOTE_PASSWORD=
export PROD_REMOTE_PASSWORD=
```

#### CML
```
export CML_HOST=cml.domain.com
export CML_USERNAME=user
export CML_PASSWORD=password
export CML_LAB=mdd
export CML_VERIFY_CERT=false
```

# Setup Your Inventory
The base repository contains two ansible inventory directories. The directory labled inventory_prod will contain an nso.yml file that points to the NSO instance managing your production devices. Below is an example of how your nso.yml file should look.
```
 nso_rest_url: http://xx.xx.xx.xx:8080
    nso_username: admin
    nso_auth_groups: default
      default:
        remote_name: admin
        remote_password: "{{ lookup('env', 'PROD_REMOTE_PASSWORD') | default('admin', true) }}"        
  children:
    nso:
      vars:
        ansible_user: "{{ lookup('env', 'NSO_USERNAME') | default('ubuntu', true) }}"
        ansible_password: "{{ lookup('env', 'NSO_PASSWORD') | default('admin', true) }}"
        ansible_python_interpreter: /usr/bin/python3
      hosts:
        nso1: 
          ansible_host: xx.xx.xx.xx:8080
```

Follow the same instructions for the nso.yml file in your inventory_test directory with the information for your test instance of NSO.

