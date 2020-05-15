---
layout: post
title: Building your home lab - 1
excerpt: "Installing KVM with Ansible"
comments: true
category: blog
author: Onur Yasarlar
---

I really like testing new things in my home lab but I also break things from time to time while testing. I am really bored with installing my home lab over and over again so I decided to create a series of posts to show how I automated my home lab provisioning.

Let me start with my home lab setting. I use Intel **[NUC8i7BEH](https://www.intel.com/content/www/us/en/products/boards-kits/nuc/kits/nuc8i7beh.html)** as my physical server. That small box has almost everything I need. It comes with a 4-core CPU and Hyper-Threading support. I added 32GB memory and 1TB SSD on top of it to have a solid lab environment. As a Linux guy, my choice of the operating system on my physical server is CentOS 8.

Enough with my setup, so let's make our hands dirty. I personally like KVM more than Oracle Virtualbox so in this article I will mention how to install KVM on top of CentOS 7 and CentOS 8. All you need to start with your installation is to have a workstation with Ansible so you can run the playbook. First thing first, you need to clone my **[home-lab](https://github.com/yasarlaro/home-lab)** repository and go into the directory.

```bash
git clone https://github.com/yasarlaro/home-lab.git
```

There are a couple of things you need to do before you run the ansible playbook. First, please be sure that you have passwordless SSH connectivity to your hypervisor server with a user with root privileges. Then, you need to edit the **[inventory](https://github.com/yasarlaro/home-lab/blob/master/inventory)** file and add your hypervisor IP address or FQDN under "[hypervisor]" section like below:

```yaml
[hypervisor]
192.168.0.112

[lab_jenkins]

[lab_k8s_master]

[lab_k8s_nodes]

[lab_k8s:children]
```

Since there are multiple roles under my repository, I needed to add a logical switch to allow me to run only the roles I want. That's why I created a variable file called "vars/infrastrucure.yml" and defined boolean switches for each role. You will need to edit "install_kvm" variable and set it to "yes". If you take a look at the variable file, you will immediately understand what I mean and you will also discover that I give you a choice of the user you want to run the roles on the target server. No surprise that the default admin user is "onur". Feel free to change it for your environment. Then all you need to run is:

```bash
ansible-playbook main.yml
```

Now it is time to get a coffee because it will update your server and then install all required packages for KVM and will start libvirtd service. If you are lucky enough, you will see an output like below:

```yaml
PLAY [Installing Home Lab Play] *****************************************************************************************************************************************************************************************************************************

TASK [Gathering Facts] ***********************************************************************************************************************************************************************************************************************
ok: [192.168.0.112]

TASK [/home/onur/gitrepos/home-lab/roles/yasarlaro.kvm : Check OS version] *******************************************************************************************************************************************************************
skipping: [192.168.0.112]

TASK [/home/onur/gitrepos/home-lab/roles/yasarlaro.kvm : Get CPU Virtualization Support] *****************************************************************************************************************************************************
changed: [192.168.0.112]

TASK [/home/onur/gitrepos/home-lab/roles/yasarlaro.kvm : Check CPU Virtualization Support] ***************************************************************************************************************************************************
skipping: [192.168.0.112]

TASK [/home/onur/gitrepos/home-lab/roles/yasarlaro.kvm : Install yum-utils] ******************************************************************************************************************************************************************
ok: [192.168.0.112]

TASK [/home/onur/gitrepos/home-lab/roles/yasarlaro.kvm : Upgrade system] *********************************************************************************************************************************************************************
ok: [192.168.0.112]

TASK [/home/onur/gitrepos/home-lab/roles/yasarlaro.kvm : Check if reboot is required] ********************************************************************************************************************************************************
changed: [192.168.0.112]

TASK [/home/onur/gitrepos/home-lab/roles/yasarlaro.kvm : Reboot server if needed] ************************************************************************************************************************************************************
skipping: [192.168.0.112]

TASK [/home/onur/gitrepos/home-lab/roles/yasarlaro.kvm : Load libvirt packages from variable file] *******************************************************************************************************************************************
ok: [192.168.0.112]

TASK [/home/onur/gitrepos/home-lab/roles/yasarlaro.kvm : Install Libvirt Packages] ***********************************************************************************************************************************************************
changed: [192.168.0.112]

TASK [/home/onur/gitrepos/home-lab/roles/yasarlaro.kvm : Enable and start libvirt service] ***************************************************************************************************************************************************
changed: [192.168.0.112]

PLAY RECAP ***********************************************************************************************************************************************************************************************************************************
192.168.0.112              : ok=8    changed=4    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0
```

Congrats, you have a working KVM on your server with default settings. I will not go into details of how you can set up a custom configuration on your KVM because "default" settings will be sufficient in most cases. In the next series of the article, I will show you how to automate VM provisioning on your newly built environment.