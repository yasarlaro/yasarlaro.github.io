---
layout: post
title: Building your home lab - 2
excerpt: "Provisioning a VM with Ansible and cloud-init"
comments: true
category: blog
author: Onur Yasarlar
---
In this article, I will show you how you can provision new VMs in your infrastructure with Ansible. My preferred method to provision a new VM would be to use Terraform but since we have only our hypervisor server installed yet, we need to first provision the environment. I will be using CentOS 7 cloud image here but you can also go with CentOS 8 if you like. 

So let's make our hands dirty and create a new role for this task. I named it kvm_cloud_init and it is almost the ugliest way of using Ansible. Ansible Galaxy rated also my module with "3.1/5" but I already know why. I needed to use "shell" module lots of time instead of other regular Ansible modules because there is no official Ansible KVM module that satisfies my needs. Who knows, this can be my next project to write a fully functional KVM module in Ansible.

Since I am using generic cloud images, I used cloud-init to customize my new virtual machine. Cloud-init is the de facto industry-standard tool for cross-platform cloud instance initialization. If you are not familiar with it, please spend some time to understand what it is and how you can use it.

So what do we need to have a VM to be provisioned? First of all, we require a running hypervisor and we already covered that part in the previous article. Then let's make the list:

- Hostname

- Network (IP Address / Subnet / GW)

- Compute resource

- Operating System Image

- User Data and Meta Data for Cloud-Init

As you can see, almost all of them can be kept under default values except IP address if you are not using DHCP. Using DHCP could be an easier solution but we like the hard way, right? That's why the role expects 'vm_ip' variable and fails if you forget to define it. 

Let's go to the tasks of the play and see what and how we are covering the provisioning.

As I mentioned, I like to use default variables as much as possible. I don't want my code to fail because of an unset variable. As you can see below, I am starting my main playbook with variable checking. If you do want to set your hostname, feel free to set vm_name variable while running the role

```yaml
- name: "Set VM name if it has not been defined"
  set_fact:
     vm_name: "test-{{ 1000 | random }}"
     run_once: "yes"
   when: vm_name is not defined

- name: "Check if VM IP has been set"
  fail:
    msg: "No IP has been set to VM. Please set it by (vm_ip) variable"
    when: vm_ip is not defined
```

Then the next thing is to download the cloud image if we have not done it already.

```yaml
- name: "Check if Image file exists"
  stat:
    path: "{{ image_path }}/{{ image_name }}"
  register: image_exist

- name: "Download KVM Image file"
  get_url:
    url: "{{ image_url }}"
    dest: "{{ image_path }}"
 when: not image_exist.stat.exists
```

Then we need to create the image directory and place our VM disk image. But wait a second, what if I provided a name which already exists. Don't worry I am also checking that.

```yaml
- name: "Check if VM directory exists"
  stat:
    path: "/var/lib/libvirt/images/{{ vm_name }}"
  register: image_dir

- fail:
    msg: "VM Image directory already exists for {{ vm_name }}"
  when: image_dir.stat.isdir is defined and image_dir.stat.isdir

- name: "Create VM directory"
  file:
    path: "/var/lib/libvirt/images/{{ vm_name }}"
    state: directory

- name: "Copy image to VM directory"
  copy:
    src: "{{ image_path }}/{{ image_name }}"
    dest: "/var/lib/libvirt/images/{{ vm_name }}/{{ vm_name }}.qcow2"
    remote_src: "yes"
```

Then it is time to resize our disk size. We don't want to live with a very minimal size.

```yaml
- name: "Resize VM boot disk"
  shell: "qemu-img resize {{ vm_name }}.qcow2 {{vm_root_disk_size}}"
  args:
    chdir: "/var/lib/libvirt/images/{{ vm_name }}/"
```

Now the most important part, we will need to provide user and meta data to our virtual machine with the help of cloud-init. We need to generate an ISO image with the data and mount it while installing our virtual machine. I strongly recommend you to take a look at user and meta data templates in the role as almost all customization happens there. By the way, if you do not take a look, you will not know which user and default password to use to login to your newly provisioned VM :)

```yaml
- name: "Copy user-data and meta-data cloud-init tempalates"
  template:
    src: "{{ item }}"
    dest: "/var/lib/libvirt/images/{{ vm_name }}/{{ item.split('.')[0] }}"
   loop:
     - meta-data.j2
     - user-data.j2

- name: "Install genisoimage to create iso"
  yum:
    name: genisoimage
    state: present

- name: Create cloud-init iso image
  shell: "mkisofs -o {{ vm_name}}-cidata.iso -V cidata -J -r user-data meta-data"
  args:
   chdir: "/var/lib/libvirt/images/{{ vm_name }}/"
```

So we are almost ready. It is time to run the virt-install command to complete our work

```yaml
- name: "Provision VM"
   shell: >
    virt-install --import --name "{{ vm_name}}"
    --memory "{{ vm_memory }} "
    --vcpus "{{ vm_cpu }}"
    --cpu host 
    --disk "{{ vm_name}}".qcow2,format=qcow2,bus=virtio 
    --disk "{{ vm_name}}"-cidata.iso,device=cdrom 
    --network type=direct,source="{{ ansible_default_ipv4['interface'] }}",source.mode=bridge 
    --os-type=linux 
    --os-variant=centos7.0 
    --graphics spice 
    --noautoconsole
   args:
     chdir: "/var/lib/libvirt/images/{{ vm_name }}/"
```

And we are done. You are having an up and running brand new VM to use. Let me also give you I ran it in my environment.

```shell
onur@lab:~ $ ansible-playbook main.yaml -e "vm_ip=192.168.0.37"

PLAY [Provision a VM play] *************************************************

TASK [Gathering Facts] *****************************************************
Monday 11 May 2020  22:23:08 +0400 (0:00:00.023)       0:00:00.023 ************ 
[DEPRECATION WARNING]: Distribution fedora 31 on host 192.168.0.50 should use /usr/bin/python3, but is using /usr/bin/python for backward compatibility with prior Ansible releases. A future Ansible release will default to using the 
discovered platform python for this host. See https://docs.ansible.com/ansible/2.9/reference_appendices/interpreter_discovery.html for more information. This feature will be removed in version 2.12. Deprecation warnings can be disabled 
by setting deprecation_warnings=False in ansible.cfg.
ok: [192.168.0.50]

TASK [ansible-role-kvm-cloud-init : Set VM name if it has not been defined] 
Monday 11 May 2020  22:23:10 +0400 (0:00:01.876)       0:00:01.900 ************ 
ok: [192.168.0.50]

TASK [ansible-role-kvm-cloud-init : Check if VM IP has been set] ***********
Monday 11 May 2020  22:23:10 +0400 (0:00:00.054)       0:00:01.954 ************ 
skipping: [192.168.0.50]

TASK [ansible-role-kvm-cloud-init : Print VM Information] ******************
Monday 11 May 2020  22:23:10 +0400 (0:00:00.048)       0:00:02.003 ************ 
ok: [192.168.0.50] => {
    "msg": "VM Information: test-403 - 192.168.0.37"
}

TASK [ansible-role-kvm-cloud-init : Check if Image file exists] ************
Monday 11 May 2020  22:23:10 +0400 (0:00:00.054)       0:00:02.057 ************ 
ok: [192.168.0.50]

TASK [ansible-role-kvm-cloud-init : Download KVM Image file] ***************
Monday 11 May 2020  22:23:12 +0400 (0:00:01.716)       0:00:03.773 ************ 
skipping: [192.168.0.50]

TASK [ansible-role-kvm-cloud-init : Check if VM directory exists] **********
Monday 11 May 2020  22:23:12 +0400 (0:00:00.049)       0:00:03.823 ************ 
ok: [192.168.0.50]

TASK [ansible-role-kvm-cloud-init : fail] **********************************
Monday 11 May 2020  22:23:13 +0400 (0:00:00.756)       0:00:04.579 ************ 
skipping: [192.168.0.50]

TASK [ansible-role-kvm-cloud-init : Create VM directory] *******************
Monday 11 May 2020  22:23:13 +0400 (0:00:00.049)       0:00:04.629 ************ 
changed: [192.168.0.50]

TASK [ansible-role-kvm-cloud-init : Copy image to VM directory] ************
Monday 11 May 2020  22:23:14 +0400 (0:00:00.949)       0:00:05.578 ************ 
changed: [192.168.0.50]

TASK [ansible-role-kvm-cloud-init : Resize VM boot disk] *******************
Monday 11 May 2020  22:23:19 +0400 (0:00:05.258)       0:00:10.837 ************ 
changed: [192.168.0.50]

TASK [ansible-role-kvm-cloud-init : Copy user-data and meta-data cloud-init tempalates] ******************************************
Monday 11 May 2020  22:23:20 +0400 (0:00:00.978)       0:00:11.816 ************ 
changed: [192.168.0.50] => (item=meta-data.j2)
changed: [192.168.0.50] => (item=user-data.j2)

TASK [ansible-role-kvm-cloud-init : Install genisoimage to create iso] *****
Monday 11 May 2020  22:23:23 +0400 (0:00:02.895)       0:00:14.712 ************ 
ok: [192.168.0.50]

TASK [ansible-role-kvm-cloud-init : Create cloud-init iso image] ***********
Monday 11 May 2020  22:23:27 +0400 (0:00:04.591)       0:00:19.303 ************ 
changed: [192.168.0.50]

TASK [ansible-role-kvm-cloud-init : Provision VM] **************************
Monday 11 May 2020  22:23:28 +0400 (0:00:00.638)       0:00:19.941 ************ 
changed: [192.168.0.50]

PLAY RECAP ***********
192.168.0.50               : ok=12   changed=6    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0   

Monday 11 May 2020  22:23:30 +0400 (0:00:01.627)       0:00:21.569 ************ 
=============================================================================== 
ansible-role-kvm-cloud-init : Copy image to VM directory -------------------------------------------------------- 5.26s
ansible-role-kvm-cloud-init : Install genisoimage to create iso ------------------------------------------------- 4.59s
ansible-role-kvm-cloud-init : Copy user-data and meta-data cloud-init tempalates -------------------------------- 2.90s
Gathering Facts ------------------------------------------------------------------------------------------------- 1.88s
ansible-role-kvm-cloud-init : Check if Image file exists -------------------------------------------------------- 1.72s
ansible-role-kvm-cloud-init : Provision VM ---------------------------------------------------------------------- 1.63s
ansible-role-kvm-cloud-init : Resize VM boot disk --------------------------------------------------------------- 0.98s
ansible-role-kvm-cloud-init : Create VM directory --------------------------------------------------------------- 0.95s
ansible-role-kvm-cloud-init : Check if VM directory exists ------------------------------------------------------ 0.76s
ansible-role-kvm-cloud-init : Create cloud-init iso image ------------------------------------------------------- 0.64s
ansible-role-kvm-cloud-init : Print VM Information -------------------------------------------------------------- 0.05s
ansible-role-kvm-cloud-init : Set VM name if it has not been defined -------------------------------------------- 0.05s
ansible-role-kvm-cloud-init : fail ------------------------------------------------------------------------------ 0.05s
ansible-role-kvm-cloud-init : Download KVM Image file ----------------------------------------------------------- 0.05s
ansible-role-kvm-cloud-init : Check if VM IP has been set ------------------------------------------------------- 0.05s
```
