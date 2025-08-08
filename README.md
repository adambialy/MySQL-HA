# MySQL-HA
mysql ha setup with haproxy/orchestrator/mariadb on debian12

## Prereq

Setup 6 nodes with Debian12 - 3X mysql and 3X haproxy.

And update them:

```apt update; apt upgrade -y; reboot```


Before start for simplicity add keys so you can login to each node without password.

ssh-keygen -t ed25599

### Configure /etc/hosts on all nodes

```/etc/hosts```

```192.168.1.201 haproxy01
192.168.1.202 haproxy02
192.168.1.203 haproxy03
192.168.1.211 mysql01
192.168.1.212 mysql02
192.168.1.213 mysql03```

## MySQL setup

mariadb

### setup slaves



## haproxy

follow link to install haproxy

### xinetd on servers

```apt install xinetd -y```

copy scripts:



## orchestrator install

download orchestrator:


