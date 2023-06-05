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
### Environmental Variables
The MDD tooling requires several environment variables. The first one required for base execution is:
```
export ANSIBLE_PYTHON_INTERPRETER=${VIRTUAL_ENV}/bin/python
export ANSIBLE_COLLECTIONS_PATH=./
```
### Ansible Collections
```
ansible-galaxy collection install -r requirements.yml
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

### inventory_prod/nso.yml
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

### inventory_prod/network.yml
The network.yml file is the key element for your NetDevOps pipeline. It defines your infrastructures heirarchy and design. It also allows you to provide tags to specific devices, giving operators the ability to execute specefic tests or push modular pieces of configuration to a subset of your infrastructure. The base repository provies a sample network.yml inventory file that can be used as a template to build your network. This template is taken from the core MDD repository and maps to the excercises that an be executed against our reference topology.

When implementing MDD in a brownfield environment, we want to harvest the configurations from our poduction network, so it is recommended to start with the network.yml file in your inventory_prod folder.

If you are using this base repository with CML and you do not have a physical environment, you do not need to provide any additional information for each device. The ansible collection takes advantage of the CML API to automatically grab the IP addresses for each device.
```
  children:
    network:
      children:
        switches:
          hosts:
            hq-sw1:
            hq-sw2:
            site1-sw1:
            site2-sw1:
```

If you are harvesting your data from a physical environment, you will need to provide IP addresses for each device as depicted in the example below:

```
  children:
    network:
      children:
        switches:
          hosts:
            hq-sw1:
              ansible_host: 10.0.0.1
            hq-sw2:
              ansible_host: 10.0.0.2
            site1-sw1:
              ansible_host: 10.0.0.3
            site2-sw1:
              ansible_host: 10.0.0.4
  ```
## Harvesting Data
Now that our inventories have been created, we need to update our production instance of NSO with our inventory. Start out by adding the default auth group to NSO so it can connect to your devices.

```
ansible-playbook ciscops.mdd.nso_init -i=inventory_prod
```
Now that our auth group has been created, we can have NSO connect to the devices by running the following playbook:

```
ansible-playbook ciscops.mdd.nso_update_devices -i=inventory_prod
```

Once the devices from the network.yml file in your inventory_prod directory have been added, we can execute a sync action for NSO to ingest the configurations into its database.

```
ansible-playbook ciscops.mdd.nso_sync_from -i=inventory_prod
```
Next, we will harvest the configurations as structured data in the OpenConfig format. If you would like to harvest the native data, refer to the additional commands section of this guide.

```
ansible-playbook ciscops.mdd.harvest -i=inventory_prod
```

You should start to see the configurations being populated under the mdd-data directory matching the heirarchy defined in your network.yml file.

![Screenshot 2023-06-05 at 4 37 14 PM](https://github.com/lvangink/mdd_base/assets/65776483/e01fbc26-cf02-49c3-8460-39b03f7f28da)

You will see files parsed as both oc-feature.yml and config-remaining.yml. We are regularly adding supported OpenConfig features to the NSO service. While we continue to add support, you will have your data represented as both OpenConfig and Native.

## Building Your Digital Twin
If you do not have an existing simulation environment for pipeline testing, our ansible collection includes a playbook that will use CDP and LLDP to map your network and build a simulation that can be used to import into CML. This playbook also builds the interfce mapping table necessary to ensure your simulation matches your production interface naming convention.

Before you run the simulation creation playbook, navigate to your group_vars directory under your inventory_prod directory and you will see a cml.yml file. This file enables the use of the virtual catalyst 9000 switch for features specefic to layer 3 switches. If you require L3 switching, uncomment the following:
```
  l3switch:
    node_definition: Cat9000v
    image_definition: Cat9000v-24p
    ram: 18432
    cpus: 4
    tags:
      - l3switch
    type: l3switch
```
If you do not require L3 switching, you can leave the Cat9000v commented out and make sure the following is uncommented:
```
    l3switch:
      node_definition: iosvl2
      ram: 768
      tags:
        - l3switch
      type: l3switch
```
Now we can execute the playbook with the proper Cat9kv flag set:

```
ansible-playbook ciscops.mdd.cml_update_lab -i inventory_prod -e start_from=1 -e inventory_dir=inventory_test -e use_cat9kv=yes
```
If you are not using the Cat9kv, you can set use_cat9kv=no
