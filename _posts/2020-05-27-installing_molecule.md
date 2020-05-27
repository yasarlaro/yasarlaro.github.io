---
layout: post
title: Installing Ansible Molecule for role testing
excerpt: "Installing Ansible Molecule for role testing"
comments: true
category: blog
author: Onur Yasarlar
---
If you are a believer of testing, then you already know [Ansible Molecule](https://molecule.readthedocs.io/en/latest/index.html). In this post, I would like to show you how you can install / setup Ansible Molecule on a CentOS 7 or CentOS 8 server. If you would like to more about `Ansible Molecule`, you can take a look at below documentations:

  - [Getting Started to Ansible Molecule](https://molecule.readthedocs.io/en/latest/getting-started.html)
  - [Ansible Module with Docker tutorial](https://www.youtube.com/watch?v=DAnMyBZ8-Qs)

It is becoming a habit for my Ansible roles to start with including OS specific variables. I am not breaking that habit and will start again. Let me show what it looks like for CentOS 7:

```yaml
required_packages:
  - ansible
  - python3
  - yum-utils
  - gcc
  - python-pip
  - python-devel
  - openssl-devel
  - libselinux-python

molecule_pip:
  - molecule
  - docker
```

I have also other OS agnostic variables defined under `vars/main.yml`. Feel free to change `remote_user` with your user.

```yaml
remote_user: vagrant
epel_repo_path: "/etc/yum.repos.d/epel.repo"
dockerce_repo_path: "/etc/yum.repos.d/docker-ce.repo"
dockerce_repo_url: "https://download.docker.com/linux/centos/docker-ce.repo"
containderdio_version: "1.2.6-3.3"
containerdio_url: "https://download.docker.com/linux/centos/7/x86_64/stable/Packages/containerd.io-{{ containderdio_version }}.el7.x86_64.rpm"
```

With the current version of Docker, if you follow the official documentation, you will hit an issue so until a fix comes, I am manually installing  `containerd.io` package. I will go back and update this playbook once the fix is there.

You have to have `Docker` up and running along with `Ansible`. So, let's start with installing them. Since I don't want to run my `Docker` as the root user, I am using become for installation tasks under a `block` statement.

```yaml
- name: Install Molecule
  block:
    - name: Check if EPEL repo is already configured
      stat:
        path: "{{ epel_repo_path }}"
      register: epel_repo_exists

    - name: Enable epel-repository
      yum:
        name: epel-release
        state: present
      when: not epel_repo_exists.stat.exists

    - name: Install required packages
      yum:
        name: "{{ required_packages }}"
        state: present

    - name: Check if docker-ce repo is already configured
      stat:
        path: "{{ dockerce_repo_path }}"
      register: dockerce_repo_exists

    - name: Enable Docker Repo
      get_url:
        dest: "{{ dockerce_repo_path }}"
        url: "{{ dockerce_repo_url }}"
        mode: "0644"
        owner: root
        group: root
      when: not dockerce_repo_exists.stat.exists

    - name: Gather the rpm package facts
      package_facts:
        manager: auto

    # Currently it gives an error if you install containerd.io
    # along with docker
    - name: Install containerd.io manually
      yum:
        name: "{{ containerdio_url }}"
        state: present
      when: "'containerd.io' in ansible_facts.packages"

    - name: Install Docker
      yum:
        name:
          - docker-ce
          - docker-ce-cli

    - name: Add remote user to docker group
      user:
        name: "{{ remote_user }}"
        groups: docker
        append: true

    - name: Start and enable docker
      service:
        name: docker
        state: started
        enabled: true
  become: true
```

All the above steps are taken from the official [docker](https://docs.docker.com/engine/install/centos/) website except the manual installation of `containerd.io` package as I described at the beginning of the post.

Then now it is time to make your OS ready for `molecule` installation. I will again use a `block` statement to run the tasks as a privileged user. I could have included the tasks in the first block statement but I am planning to change the methodology and use Python virtualenv later. That's why I made a new block for the next two tasks to remind myself that I need to change it in the future.

```yaml
- name: Pip requirements in CentOS7
  block:
    - name: Upgrade pip in CentOS7
      command: pip install --upgrade pip
      args:
        warn: no
      become: true

    - name: Upgrade setuptools
      command: pip install --upgrade --user setuptools
      args:
        warn: no
  when: ansible_facts['distribution_major_version'] == "7"
```

All prerequisites are completed so we can install molecule with `pip`.

```yaml
- name: Install molecule
  pip:
    name: "{{ molecule_pip }}"
    extra_args: --user
```

That's it. I am planning to write another blog post on how to user `docker` and `vagrant` as your molecule drivers but two websites I provided at the beginning of the post will just do fine if you cannot wait for my new post.

I will not miss the time I wasted to run multiple tests for my Ansible roles and will let molecule handle the dirty job for me anymore. I wish I had been introduced to molecule earlier.
