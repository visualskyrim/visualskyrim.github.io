---
layout: post
title: "Ansible: Deploy public key to remote hosts"
description: "No magic."
modified: 2017-11-09
tags: [ansible]
category: Snippets
image:
  feature: abstract-07.jpg
  credit: Chris Kong
  creditlink: http://visualskyrim.github.io/
comments: true
share: true
---

Since ansible uses ssh to access to each of the remote hosts, before we execute a playbook, we need to put the public key to the `~/.ssh/authorized_keys` so that you don't need to input the password for ssh every time you execute the playbook.

Usually, people just manually copy the public key to the remote hosts' `~/.ssh/authorized_keys` files. But things could get quite exhausting when you have an inventory with more than hundreds of nodes.
That's why people are asking how to use ansible playbook to deploy the public key to the remote hosts.

This post will show how to do that in a more complex (and more realistic) scenario, where users are not allowed to ssh to the remote as the *super-user*, while they still need to deploy the public key under the remote *super-user*:

![A more complicated scenario]({{ site.url }}/images/cant-ssh-using-super-user.jpg)

> The above situation usually comes with the security policy of the company you are working for.
> Allowing people to ssh to remote as *super-user* **will expose the password of the super-user**, which could be a potential security risk.

First, you will need to get the public key of the *super-user* you want to deploy.

Since you can only ssh to the remote hosts via the *normal-user*, you need to run the deploy playbook under *normal-user* on your ansible server.
And because you are deploying the public key to `~/.ssh/authorized_keys` under *super-user* on the remote hosts, you will need to use [become](http://docs.ansible.com/ansible/latest/become.html) to tell ansible: Once ansible gets on to the remote hosts as the *normal-user*, change to *super-user* to execute the tasks:

{% highlight yml %}
---
- hosts: all
  become: true
  become_user: "super-user"
  tasks:
  - name: make direcotry
    file:
      path: "/home/<super-user>/.ssh"
      state: directory
  - name: create empty file
    file:
      path: "/home/<super-user>/.ssh/authorized_keys"
      state: touch
  - name: put pubkey
    lineinfile:
      path: "/home/<super-user>/.ssh/authorized_keys"
      line: "{{ pubkey }}"
{% endhighlight %}

Here, you will have two problems if you just run this playbook:

- On the remote hosts, there is no public key of your *normal-user* either in the `authorized_keys`. So we will be asked for the ssh password when we execute the playbook.
- Since we are going to change to *super-user* on remote hosts, we will be asked sudo password.

To specify those two passwords when we execute the playbook, we will be using [--ask-pass](http://docs.ansible.com/ansible/latest/intro_getting_started.html#remote-connection-information) to let ansible ask ssh password when needed, and [--ask-sudo-pass](http://docs.ansible.com/ansible/latest/intro_getting_started.html#remote-connection-information) to let ansible ask sudo password if needed:

{% highlight bash %}
ansible-playbook -i <inventory-file> deploy_authorized_keys.yml --ask-become-pass --ask-pass --extra-vars='pubkey="<pubkey>"'
{% endhighlight %}

With `--ask-become-pass` and `--ask-pass` being specified, ansible will ask you for your ssh password and sudo password when you kick the playbook.

There is one more problem.
If you access to a host via ssh for the first time, you will be asked about whether to add RSA key fingerprint of this host.
However, with `--ask-pass` being specified, ansible will directly run into an error if this is the first time you access to that host.

To walk through this, you will need to disable SSH authenticity checking by adding an `ansible.cfg` to the place where you want to execute the playbook:

{% endhighlight %}
[defaults]
host_key_checking = False
{% endhighlight %}

> You can also put this file in the home directory as `~/.ansible.cfg`.

By doing so, you will be able to deploy your public key to the remote hosts in several seconds. :)
