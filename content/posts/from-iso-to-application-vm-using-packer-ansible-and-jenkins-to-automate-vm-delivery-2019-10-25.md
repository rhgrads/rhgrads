---
title: >-
  From ISO to Application VM - Using Packer, Ansible and Jenkins to automate VM
  delivery
author: Tom Page
jobtitle: Associate Consultant - Red Hat
date: '2019-10-25'
---
**Delivering virtual machines at scale is a common issue for many ops teams and having the ability to take an ISO and form a ‘Golden Image’ which can subsequently be configured in an automated way is often seen as a goal state. By using Packer and Ansible, with orchestration using Jenkins, it is possible to reach this level of automation.**

Over the past few months I have been working with a Telco company who have this fairly common use case of requiring template VMs to be created and then be able to specialise each machine depending on its application. This blog will follow the journey of deploying a service of VMs in vSphere using Packer, Ansible and Jenkins and detail some of the obstacles we encountered on the way.

As a disclaimer, this is not the only way to do this, and is probably not the best way either!

## What are the tools?
### Packer
Packer is a tool for automating the build of virtual machines created by HashiCorp. It sells itself as being quick and easy to use and supports the creation of VMs on a number of platforms. For our use case we have targeted vSphere as this was where the vast majority of our customer’s infrastructure sat. A Packer build is defined by a template which has three key sections: 
- **Builders** - which define the target platform, any credentials for said platform, the base ISO and any additional files and configuration needed for the base VM.
- **Provisioners** - which do some configuration against the machine created. Typical idea may be patching kernels, installing packages or creating users. This is optional.
- **Post-processors** - which are entirely optional and are typically used to upload artifacts or turn the VM into a template.

### Ansible
As a Red Hatter there was really only one choice for the automation engine to use in order to complete these tasks. First and foremost, Ansible is simple to understand, easy to use and, for the most part, is well documented. We deployed Ansible Tower in the environment (using our end-to-end method for deploying VMs, of course, as [dogfooding](https://en.wikipedia.org/wiki/Eating_your_own_dog_food) is important) in order to further create an easy to use, ‘press play’ solution for automating the configuration of machines. Ansible Tower can be used as an organisation’s ‘hub’ for automation and adds to the traceability and compliance of machines in the ecosystem.

### Jenkins
Jenkins is a build automation server used by many organisations. In our system it is used to orchestrate the creation of golden images. It takes what exists within our golden image git repository and runs a predefined set of instructions (set out in a Jenkinsfile) in order to reach the stage of having a template in vSphere. We also used Jenkins in order to test any roles we created as part of the project using molecule for linting, running, checking idempotency and verifying completeness. 

## Creating a Golden Image for vSphere
The first step of the process to getting a service within vmware is creating a packer template. There are a number of key factors which need to be set in order to achieve this. The repo structure can be seen in the image below and can also be found [here](https://github.com/Tompage1994/packer-example).

The key part of the builders section is the type which in our case is vsphere-iso. This is not a built-in Packer builder, but is instead a plugin created by the good people at Jetbrains. The plugin can be downloaded [here](https://github.com/jetbrains-infra/packer-builder-vsphere). Although Packer does have a built in builder to interact with vSphere; it requires ssh access to the ESX host which is frowned upon by most security teams in organisations. The documentation for the vsphere-iso is pretty good and gives lots of configuration options for creating a vm from scratch. In the image below you can see all of the variables we set in order to provision a VM. Most of these are pretty self explanatory and range from connection to the vSphere API, the location of  where the VM will be stored, the specification of the VM which is being built, the credentials for accessing the box (which would be defined in a kickstart), the path of the ISO on the datastore, the location of the kickstart file which will be mounted as a floppy and the initial boot command.

<img src="/images/screenshot-2019-10-25-at-14.58.11.png" width="100%"></img>

There is also a provisioners section which is used to run some additional configuration against the VM which will be baked in before it is turned into a template. In order to keep things simple we have one Ansible playbook which can run any roles we need. This could be things like installing organisation-wide required packages, adding security hardening, or just something as simple as we used for demo purposes: setting the [message of the day](https://en.wikipedia.org/wiki/Motd_(Unix)).

In the template you will see we have a number of variables filling some options. The notation of ``option: {{user `var_1`}}`` is used to take a user defined variable. This is passed in at run time by adding `--var “var_1=abc”` or by attaching a variables file using `--var-file=./centos7_vars`. In order to run the template defined in the image we use `packer build --var "BUILD_NUMBER=${BUILD_NUMBER}" --var "BRANCH_NAME=${BRANCH_NAME}" -var-file=centos7_vars.json centos7.json`. Another good aspect about packer is the ability to run `packer validate` against a template to gain fast feedback as to whether the template is syntactically correct.

The steps for running the pipeline are all defined within the Jenkinsfile. Very simply the code is pulled down and validated, before installing the ansible galaxy requirements set out in requirements.yml. Jenkins will finally run the build in order to provision your template VM. This can take a bit of time, but this is anticipated as you are doing a fresh install of an operating system. 

<img src="/images/screenshot-2019-10-25-at-14.29.43.png" width="100%"></img>

## Deploying a VM
Now we have our golden image stored as a template it's time to provision our first VM. The key module we will use here is the `vmware_guest` module in Ansible. Using this we can provide a load of parameters, along with the template location, in order to spin up our VM. The deployment of a machine is a three step process: firstly spinning up a machine using `vmware_guest`, then configuring the VM with base config such as enrolling into a domain, and finally deploying applications onto the machine.

### Step 1: Deploying a VM
The `vmware_guest` module has extensive documentation which can seem a little scary at first, but for a simple, base configuration the following are the options we used:

```yaml
vmware_guest:
  hostname: "{{ vcenter_hostname }}"
  username: "{{ vcenter_username }}"
  password: "{{ vcenter_password }}"
  datacenter: "{{ vcenter_datacenter }}"
  cluster: "{{ vcenter_cluster }}"
  template: "{{ vcenter_vmtemplate }}"
  folder: "{{ vcenter_vcsfolder }}"
  datastore: "{{ vcenter_datastore }}"
  name: "{{ instance_name }}"
  guest_id: "{{ instance_guest_id }}"
  hardware:
    memory_mb: "{{ instance_size_values.memory | default(1024) }}"
    num_cpus: "{{ instance_size_values.cores | default(1) }}"
    scsi: "{{ vcenter_scsitype }}"
  disk: "{{ instance_disks }}"
  networks:
    - name: "{{ network_interfaces.network_name }}"
      type: static
      ip: "{{ network_interfaces.ipv4.address }}"
      gateway: "{{ network_interfaces.ipv4.gateway }}"
      netmask: "{{ network_interfaces.ipv4.netmask }}"
      device_type: "vmxnet3"
  customization:
    dns_servers: "{{ network_interfaces.dns_servers }}"
  wait_for_ip_address: true
  state: poweredon
``` 
This will go ahead and create our VM and place it in the desired location, give it an IP address specified in the variable `network_interfaces.ipv4.address` which we will then use as the target for the next playbook.

### Step 2: Configuring VM
Now we have a shiny new VM we need to actually be able to use it. Most organisations will have some form of domain in which they will enrol all of their machines in order to be able to target a machine by its hostname and be able to log into it. For a Linux machine this requires a number of steps to be undertaken. Firstly the hostname must be updated to what we set the hostname as in vmware (this does not happen automatically in Linux), we then need to configure ntp by setting the ntp server to the domain controller. Finally we can join to AD by setting up Kerberos, sssd and then using the adcli package to `adcli join` against the domain. The role used to join to AD for linux can be found [here](https://github.com/Tompage1994/ansible-role-linux-ad-manage). 

> Windows AD join is actually more simple as you can use the `win_domain_membership` module and then simply reboot the machine. Makes me wonder why I’m not a Windows developer...

### Step 3: Deploying Applications
If we have got to this stage, the final section should be nice and easy. If you know how to write Ansible, and I assume you do if you got this far, then you should be able to deploy an application on the box. Simply target against the box as you did for step 2 and away you go!

## Deploying VMs as part of a service
Often we will want to do more than just deploy a single VM, and instead will want to deploy a service. This will mean we need to stitch together some of the playbooks we already created in order to deploy multiple VMs. We may also not want to use hard coded IP addresses and instead use some form of IP address management (IPAM) in order to provision the machine a space in the network. There are roles out there on Galaxy in order to complete this task, or write your own to suit your own environment.

At this point we will need to write a playbook which iterates over a list of instances and sticks it altogether. But we will be missing a step. We need to somehow pull out the ip address of the new machines to be able to configure each machine separately. This is where dynamic inventories come in. We use a slightly altered version of the vmware_inventory.py (original found [here](https://github.com/ansible/ansible/blob/devel/contrib/inventory/vmware_inventory.py), used version [here](https://raw.githubusercontent.com/ansible/ansible/807181806bdda6519e43ca9d0044a5b3ebda5019/contrib/inventory/vmware_inventory.py)) due to the fact we need to iterate over tags (more on this later) and this is not supported out of the box.

The pattern we followed in order to generate instances for a service was to create a group_vars file for a new group we will call `builder`. Here we define the instances in the following way:
```yaml
---
instances:
  - name: app1
    type: application
  - name: app2
    type: application
  - name: db1
    type: database
  - name: logger1
    type: logger
```

We then set out a vars file for each instance type. E.g. `vars/logger.yml` which contains all of the information which is needed to be supplied to vmware_guest and AD join and any further application based tasks. This pattern will work as we can assume each type will have the same variables for each of the instances with the exception of the name where we will use `{{ instance.name }}`. For a situation where there may be some variance then additional var files can be created and new key-value pairs can be added to the instance. For example, if we have two availability zones we may want to use the following for each instance and then create a vars file at `vars/aza/logger.yml`:

```yaml
  - name: logger1
    type: logger
    availability_zone: aza
```

The final piece of the puzzle is how we use the dynamic inventory to pull out only the guests which we have provisioned before we configure them. The most simple way to do this is to apply tags to the VMs in vmware. As you may have guessed there is a module which will do this in a neat and simple way. The example below will set whichever tags required by setting `instance_tags` in the form `category_name:tag_name`. Here we will probably want to do something such as set a ‘service’ and a ‘component’ to be ‘demoservice’ and ‘logger’ respectively. This way we can set our target hosts for the configure playbook to be `vmware_tag_service_demoservice` (this is dynamically created by `vmware_inventory.py`) in order to target each host within the new service and `vmware_tag_component_logger` for each application configuration playbook.

```yaml
vmware_tag_manager:
    hostname: '{{ vcenter_hostname }}'
    username: '{{ vcenter_username }}'
    password: '{{ vcenter_password }}'
    tag_names: "{{ instance_tags }}"
    object_name: "{{ instance_name }}"
    object_type: VirtualMachine
    state: add
```

So how do we piece this all together? Well we could run multiple steps where we manually run each playbook as it completes (we cant put it all in one playbook as we need the dynamic inventory to update after the machines have been provisioned), but this sounds like a lot of work which could surely be automated. Fortunately this is where Ansible Tower comes in; we can string together a list of plays and inventory updates in order to get an end-to-end deployment of our service. This workflow definition we have created can then be visualised in Tower in this end to end view.

<img src="/images/screenshot-2019-10-25-at-15.05.11.png" width="100%"></img>


## Conclusion
There is a lot of information here and a lot of steps, and as I said at the start, this is simply one way of achieving an end to end deployment of virtual machines by utilising a golden image pipeline, generating a template, and then using that template to deploy and configure your machine. There are certainly other tools out there that will help you achieve some of these steps such as Satellite which can take a template and spin up your VMs for you whilst maintaining an inventory you can query in Ansible. By following these steps, however, you will have a good platform from which to iterate and alter depending on your specific use cases. This can also be taken a lot further and be made more generic such that everything which is provisioned can be defined within various inventories which are then applied to provision each service.
