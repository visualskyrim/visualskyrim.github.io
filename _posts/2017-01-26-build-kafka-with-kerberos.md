---
layout: post
title: "Build Kafka Cluster with Kerberos"
description: "Step by step to show you how to make Kafka cluster work with Kerberos."
modified: 2017-01-26
tags: [stream process,kafka,kerberos]
category: FrameworkSetting
image:
  feature: abstract-14.jpg
  credit: Chris Kong
  creditlink: http://visualskyrim.github.io/
comments: true
share: true
---

***This is a for-beginner tutorial for those who already understand how Kafka works and the basic functionality of Kerberos.***

# Purpose

- Setup a simple pipeline for stream processing within the VMs.
- Integrate Kafka with Kerberos' SASL authentication.
- Console pipeline to check if our setup works.

The versions of components in this post will be:

- Ubuntu: 16.04.1
- Kafka: 0.10.1.0 with scala 2.11
- Kerberos: 5

# Prepare the Servers

We are going to have 3 Ubuntu servers in the VirtualBox:

***server-kerberos*** A server for Kerberos.
***server-kafka*** A server for both Zookeeper and Kafka Broker.
***server-kafka-client*** A server for Kafka clients(producer and consumer).

You can also make ***server-kafka-client*** into two different servers, but the setups for these two servers will be exactly the same.
To make things simple and clean, *this post will only use one server to host both producer and consumer*.

For the same reason, ***server-kafka*** can also be splited into two servers.

## Installation of Kerberos and Kafka

Use VirtualBox to create all these 3 servers with Ubuntu.
Go to your VirtualBox manager, and make sure all your boxes' network adapters are set to NAT.

For ***server-kerberos***:

To install kerberos, enter:

```
apt-get install krb5-admin-server krb5-kdc
```

During the installation, you will be asked for several settings. Enter the settings like below:

> *Default Kerberos version 5 realm?* [VISUALSKYRIM]
>
> *Kerberos servers for your realm?* [kerberos.com]
>
> *Administrative server for your realm?* [kerberos.com]


For ***server-kafka***, ***server-kafka-client***:

Install krb5-user for SASL authentication:

```
sudo apt-get install krb5-user
```

During this installation, you will be asked the same questions. Just answer them with the same answer:

> *Default Kerberos version 5 realm?* [VISUALSKYRIM]
>
> *Kerberos servers for your realm?* [kerberos.com]
>
> *Administrative server for your realm?* [kerberos.com]


These question will generate a Kerberos config file under you */etc/krb5.config*.


Install kafka:

```
wget http://ftp.meisei-u.ac.jp/mirror/apache/dist/kafka/0.10.1.0/kafka_2.10-0.10.1.0.tgz
tar -xzf kafka_2.11-0.10.1.0.tgz
cd kafka_2.11-0.10.1.0
```

Later in this post, you will need to transfer authentication files(keytabs) between servers.
For that purpose, this post will use *scp*, and *openssh-server* will be installed.

If you are going to use other methods to transfer files from ***server-kerberos***, feel free to skip this installation.

```
apt-get install openssh-server
```

## Setting up servers

Before starting to set up servers, we need to change our VMs' network adapters to **Host-Only** and reboot to get an individual IP address for each VM.

After that, go to each server to get their IP address by

```
ifconfig
```

Then input those IP address and the hostnames into */etc/hosts*. Something like this:

```
192.168.56.104  kerberos.com
192.168.56.106  kafka.com
192.168.56.107  kafka-client.com
```

Then make sure all these 3 servers can ping each other by using the hostname.

After that, we start to set up our servers one by one:

### Kerberos Server

Create the new realm.

```
sudo krb5_newrealm
```

> This might stuck at the place where command-line prompts *Loading random data*.
>
> If this happens, run the following code first: *cat /dev/sda > /dev/urandom*

Then edit */etc/krb5.conf*.

The updated content should be like:

```
[libdefaults]
    default_realm = VISUALSKYRIM

...

[realms]
    VISUALSKYRIM = {
        kdc = kerberos.com
        admin_server = kerberos.com
    }

...

[domain_realm]
    kerberos.com = VISUALSKYRIM
```

Next, add principals for each of your roles:

```
- zookeeper
- kafka
- kafka-client
```

Enter:

```
$ sudo kadmin.local

> addprinc zookeeper
> ktadd -k /tmp/zookeeper.keytab zookeeper
> addprinc kafka
> ktadd -k /tmp/kafka.keytab kafka
> addprinc kafka-client
> ktadd -k /tmp/kafka-client.keytab kafka-client
```

Move */tmp/zookeeper.keytab* and */tmp/kafka.keytab* to your ***server-kafka***, and move */tmp/kafka-client.keytab* to your ***server-kafka-client***.

### Kafka Server

Just like real world, every individual(program) in the distributed system must tell the authority(Kerberos) two things to identify itself:

The first thing is the accepted way for this role to be identified.
It's like in America, people usually use drive license, and in China people use ID card, while in Japan, people use so called MyNumber card.

The second thing is the file or document that identifies you according to your accepted identify method.

The file to identify the role in our SASL context, is the keytab file we generated via the *kadmin* just now.

Suggest you put *zookeeper.keytab* and *kafka.keytab* under */etc/kafka/* of you ***server-kafka***.

We need a way to tell our program where to find this file and how to hand it over to the authority(Kerberos). And that will be the JAAS file.

We create the JAAS files for Zookeeper and Kafka and put it to */etc/kafka/zookeeper_jaas.conf* and */etc/kafka/kafka_jaas.conf*.

```
Server {
  com.sun.security.auth.module.Krb5LoginModule required debug=true
  useKeyTab=true
  keyTab="/etc/kafka/zookeeper.keytab"
  storeKey=true
  useTicketCache=false
  principal="zookeeper@VISUALSKYRIM.COM";
};
```


```
KafkaServer {
  com.sun.security.auth.module.Krb5LoginModule required debug=true
  useKeyTab=true
  storeKey=true
  keyTab="/etc/kafka/kafka.keytab"
  principal="kafka@VISUALSKYRIM.COM";
};

// For Zookeeper Client
Client {
  com.sun.security.auth.module.Krb5LoginModule required debug=true
  useKeyTab=true
  storeKey=true
  keyTab="/etc/kafka.keytab"
  principal="kafka@VISUALSKYRIM.COM";
};
```

To specify the locations of these JAAS file, we need to put the locations into JVM options like:

```
-Djava.security.krb5.conf=/etc/krb5.conf
-Djava.security.auth.login.config=/etc/kafka/zookeeper_jaas.conf
-Dsun.security.krb5.debug=true
```

and

```
-Djava.security.krb5.conf=/etc/krb5.conf
-Djava.security.auth.login.config=/etc/kafka/kafka_jaas.conf
-Dsun.security.krb5.debug=true
```

To make this post easy and simple, I choose to modify the the *bin/kafka-run-class.sh*, *bin/kafka-server-start.sh* and *bin/zookeeper-server-start.sh* to insert those JVM options into the launch command.

To enable SASL authentication in Zookeeper and Kafka broker, simply uncomment and edit the config files *config/zookeeper.properties* and *config/server.properties*.

For *config/zookeeper.properties*:

```
authProvider.1=org.apache.zookeeper.server.auth.SASLAuthenticationProvider
jaasLoginRenew=3600000
kerberos.removeHostFromPrincipal=true
kerberos.removeRealmFromPrincipal=true
```

For *config/server.properties*:

```
listeners=SASL_PLAINTEXT://kafka.com:9092
security.inter.broker.protocol=SASL_PLAINTEXT
sasl.mechanism.inter.broker.protocol=GSSAPI
sasl.enabled.mechanism=GSSAPI
sasl.kerberos.service.name=kafka
```

Then start the Zookeeper and Kafka by:

```
$ bin/zookeeper-server-start.sh config/server.properties
$ bin/kafka-server-start.sh config/zookeeper.properties
```

### Kafka Client Server

The setting up for you ***server-kafka-client*** is quite similar to what you've just done for ***server-kafka***.

For JAAS file, because we are going to use the same principal and keytab for both producer and consumer in this case, we only need to create one single JAAS file */etc/kafka/kafka_client_jaas.conf*:

```
KafkaClient {
  com.sun.security.auth.module.Krb5LoginModule required debug=true
  useKeyTab=true
  storeKey=true
  keyTab="/etc/kafka-client.keytab"
  principal="kafka-client@VISUALSKYRIM.COM";
};
```

We also need to put JVM options to the *bin/kafka-run-class.sh*:

```
-Djava.security.krb5.conf=/etc/krb5.conf
-Djava.security.auth.login.config=/etc/kafka/kafka_client_jaas.conf
-Dsun.security.krb5.debug=true
```

## Give it a try

Now we can check if our setup actually works.

First we start a console-producer:

```
bin/kafka-console-producer.sh --broker-list kafka.com:9092 --topic test \
--producer-property security.protocol=SASL_PLAINTEXT \
--producer-property sasl.mechanism=GSSAPI \
--producer-property sasl.kerberos.service.name=kafka
```


And start a console-comsumer:

```
bin/kafka-console-consumer.sh --bootstrap-server ssh.com:9092 --topic test \
--consumer-property security.protocol=SASL_PLAINTEXT \
--consumer-property sasl.mechanism=GSSAPI \
--consumer-property sasl.kerberos.service.name=kafka
```

Then input some message into the console-producer to see if the same message prompted in console-consumer after a few seconds.
