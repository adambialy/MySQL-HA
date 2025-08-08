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

```
192.168.1.201 haproxy01
192.168.1.202 haproxy02
192.168.1.203 haproxy03
192.168.1.211 mysql01
192.168.1.212 mysql02
192.168.1.213 mysql03
```

## MySQL setup

mariadb

### setup slaves



## haproxy

follow link to install haproxy

### xinetd on mysql servers

```apt install xinetd -y```

configure:

```/etc/xinetd.d/readonlycheck```

```
service readonlycheck

{
    flags = REUSE
    disable         = no
    socket_type     = stream
    protocol        = tcp
    wait            = no
    user            = mysql
    server          = /usr/local/bin/readonlycheck.sh
    port            = 9201
    type            = UNLISTED
    only_from       = 127.0.0.1 192.168.1.0/24
    log_on_failure  += USERID
}
```

script ```/usr/local/bin/readonlycheck.sh```

```
#!/bin/bash

function RO_OFF(){
/bin/echo -e "HTTP/1.1 200 OK\r\n"
/bin/echo -e "Content-Type: Content-Type: text/plain\r\n"
/bin/echo -e "\r\n"
/bin/echo -e "MySQL is running.\r\n"
/bin/echo -e "\r\n"
}

function RO_ON(){
/bin/echo -e "HTTP/1.1 503 Service Unavailable\r\n"
/bin/echo -e "Content-Type: Content-Type: text/plain\r\n"
/bin/echo -e "\r\n"
/bin/echo -e "MySQL is running.\r\n"
/bin/echo -e "\r\n"
}

READ_ONLY=`mysql -N -se "SHOW GLOBAL VARIABLES LIKE 'read_only';" | awk '{print $2}'`

if [ "${READ_ONLY}" != "OFF" ]; then
	RO_ON
else
       	RO_OFF
fi

```
restart and enable xinetd:

```systemctl restart --now xinetd```


## orchestrator install on haproxy servers

download orchestrator:

```wget https://github.com/openark/orchestrator/releases/download/v3.2.6/orchestrator-cli_3.2.6_linux_amd64.tar.gz```



