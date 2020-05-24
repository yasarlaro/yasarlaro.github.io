---
layout: post
title: Installing VirtualBox on Centos 7 / Centos 8
excerpt: "Installing VirtualBox on Centos 7 / Centos 8"
comments: true
category: blog
author: Onur Yasarlar
---

I was asked by a few people "why do you use libvirt as a virtualization later?" after one of my posts. Well, the answer is simple, I really like KVM and prefer it over VirtualBox but it does not stop me to write a small how to article on "How to install Virtualbox on Centos".

I will again show you how I created an Ansible role for this. You can also find my role both in [Ansible Galaxy](https://galaxy.ansible.com/yasarlaro/virtualbox) and in my personal [Github repository](https://github.com/yasarlaro/ansible-role-virtualbox)

So let's get started. I divided my tasks into 2:
- [main.yml](https://github.com/yasarlaro/ansible-role-virtualbox/blob/master/tasks/main.yml)
- [prelim.yml](https://github.com/yasarlaro/ansible-role-virtualbox/blob/master/tasks/prelim.yml)

As the name suggests, we first need to perform some preliminary checks. Let's take a look at `prelim.yml` task.

First of all, let's check what OS versions this role supports. I always find it useful to perform this against the roles I write. It also prevent users to make some certain mistakes.

```yaml
- name: "Check OS version"
  fail:
    msg: "OS Version( {{ ansible_distribution }}{{ ansible_distribution_major_version }} ) is not certified for the role"
  when:
    - ansible_facts['distribution'] != "CentOS"
    - ansible_facts['distribution_major_version'] != "7" or ansible_facts['distribution_major_version'] != "8"
```

Secondly, we need to check if we have a compatible hardware. We need to check CPU flags and see if hardware virtualization is supported.

```yaml
- name: "Get CPU Virtualization Support"
  shell:
    cmd: egrep -o '(vmx|svm)' /proc/cpuinfo | sort | uniq | wc -l
  register: cpuinfo_output

- name: Check CPU Virtualization Support
  fail:
    msg: "CPU does not support virtualization"
  when:
    - cpuinfo_output.stdout != "1"
```

Then I prefer to update the OS to the latest version and reboot it if needed. There is a very useful command, `need-restarting`, comes with `yum-utils` package. This check will prevent unnecessary reboots if any package is being updated in the server.

**Warning**: This will probably reboot your target server so please avoid running the role on `localhost`.

```yaml
- name: "Install yum-utils"
  package:
    name: yum-utils
    state: present

- name: "Upgrade system"
  yum:
    name: '*'
    state: latest

- name: "Check if reboot is required"
  shell: /usr/bin/needs-restarting -r
  ignore_errors: "yes"
  register: needs_restarting

- name: "Reboot server if needed"
  reboot:
    msg: "Reboot initiated by Ansible"
    connect_timeout: 5
    reboot_timeout: 600
    pre_reboot_delay: 0
    post_reboot_delay: 30
    test_command: whoami
  when: needs_restarting.rc == 1
```

So all checks are done and we can get started. We will install `VirtualBox` but how can I decided which version to install. The answer is hidden in the question. Version is a variable that we can change so you can find it under [vars/main.yml](https://github.com/yasarlaro/ansible-role-virtualbox/blob/master/vars/main.yml) as `virtualbox_version`. Currently I used `6.1` version but in the future, you may want to change it.

Now all setup and ready, so let's install the binaries. The remaining of the document will refer to [main.yml](https://github.com/yasarlaro/ansible-role-virtualbox/blob/master/tasks/main.yml).

As it is the main task of the Ansible role, we first need to include the `prelim.yml`.

```yaml
- name: "Perform prelim tasks"
  include_tasks: "prelim.yml"
```

Then the next task is something I am not proud of the way how I implemented it. There is a Ansible module called `yum_repository` but you cannot enable a yum repository without without providing a `baseurl`. At CentOS 8, `PowerTools` repository is by default there but it is disabled. I could for sure use `lineinefile` module to change `enabled=0` to `enabled=1` but what if the repository is not there with some reason. That's why I decided to keep the repository as a file and deploy it to the target server. If you have a better solution, please feel free to comment under this post.

`PowerTools` repository should be enabled only for `CentOS 8` and then we need to install prerequisite packages.

```yaml
- name: "Enable PowerTools repository"
  copy:
    src: CentOS-PowerTools.repo
    dest: "/etc/yum.repos.d/CentOS-PowerTools.repo"
    owner: "root"
    group: "root"
    mode: "0644"
  when: ansible_facts['distribution_major_version'] == "8"

- name: "Install prerequisite packages"
  package:
    name:
      - "gcc"
      - "make"
      - "perl"
      - "kernel-devel"
      - "kernel-headers"
      - "elfutils-libelf-devel"
      - "xorg-x11-xauth"
      - "xorg-x11-apps"
    state: present
```

Now it is time to install `VirtualBox` binaries and then we are done.

```yaml
- name: "Add VirtualBox repository"
  yum_repository:
    name: "virtualbox"
    description: "OEL / RHEL / CentOS-$releasever / $basearch - VirtualBox"
    baseurl: "http://download.virtualbox.org/virtualbox/rpm/el/$releasever/$basearch"
    gpgkey: "https://www.virtualbox.org/download/oracle_vbox.asc"
    gpgcheck: "yes"
    repo_gpgcheck: "yes"

- name: Import VirtualBox repo GPG key
  rpm_key:
    state: present
    key: https://www.virtualbox.org/download/oracle_vbox.asc

- name: "Install VirtualBox-{{ virtualbox_version }}"
  yum:
    name: "VirtualBox-{{ virtualbox_version }}"
    state: present
```

Congrats, you have an up and running VirtualBox now.
