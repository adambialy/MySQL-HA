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

```
wget wget https://github.com/openark/orchestrator/releases/download/v3.2.6/orchestrator-3.2.6-linux-amd64.tar.gz
tar zxvf orchestrator-3.2.6-linux-amd64.tar.gz -C /
ln -s /usr/local/orchestrator/resources/bin/orchestrator-client /usr/bin/
systemctl daemon-reload
```

create orchestrator config file ```/etc/orchestrator.conf.json```

```
{
  "Debug": true,
  "EnableSyslog": false,
  "ListenAddress": ":3000",
  "MySQLTopologyUser": "orchestrator",
  "MySQLTopologyPassword": "orch123",
  "MySQLTopologyCredentialsConfigFile": "",
  "MySQLTopologySSLPrivateKeyFile": "",
  "MySQLTopologySSLCertFile": "",
  "MySQLTopologySSLCAFile": "",
  "MySQLTopologySSLSkipVerify": true,
  "MySQLTopologyUseMutualTLS": false,
  "BackendDB": "sqlite",
  "SQLite3DataFile": "/usr/local/orchestrator/orchestrator.sqlite3",
  "MySQLConnectTimeoutSeconds": 1,
  "DefaultInstancePort": 3307,
  "DiscoverByShowSlaveHosts": true,
  "InstancePollSeconds": 5,
  "RecoveryPollSeconds": 10,
  "ReplicationCredentialsQuery": "select repl_user, repl_pass from mysql.orch_meta;", 
  "DiscoveryIgnoreReplicaHostnameFilters": [
    "a_host_i_want_to_ignore[.]example[.]com",
    ".*[.]ignore_all_hosts_from_this_domain[.]example[.]com",
    "a_host_with_extra_port_i_want_to_ignore[.]example[.]com:3307"
  ],
  "RaftEnabled": true,
  "RaftDataDir": "/var/lib/orchestrator",
  "RaftBind": "192.168.1.128",
  "DefaultRaftPort": 10008,
  "RaftNodes": [
    "192.168.1.128",
    "192.168.1.212",
    "192.168.1.213"
  ],
  "WebRefreshSeconds": 10,
  "UnseenInstanceForgetHours": 240,
  "SnapshotTopologiesIntervalHours": 0,
  "InstanceBulkOperationsWaitTimeoutSeconds": 10,
  "HostnameResolveMethod": "none",
  "MySQLHostnameResolveMethod": "@@report_host",
  "SkipBinlogServerUnresolveCheck": true,
  "ExpiryHostnameResolvesMinutes": 60,
  "RejectHostnameResolvePattern": "",
  "ReasonableReplicationLagSeconds": 10,
  "ProblemIgnoreHostnameFilters": [],
  "VerifyReplicationFilters": false,
  "ReasonableMaintenanceReplicationLagSeconds": 20,
  "CandidateInstanceExpireMinutes": 60,
  "AuditLogFile": "",
  "AuditToSyslog": false,
  "RemoveTextFromHostnameDisplay": ".mydomain.com:3306",
  "ReadOnly": false,
  "AuthenticationMethod": "",
  "HTTPAuthUser": "",
  "HTTPAuthPassword": "",
  "AuthUserHeader": "",
  "PowerAuthUsers": [
    "*"
  ],
    "ClusterNameToAlias": {
    "127.0.0.1": "test suite"
  },
    "Hooks": {
    "OnMasterFailoverScript": "/usr/local/bin/update-haproxy-master.sh {{.NewMasterHostname}}"
  },
  "ReplicationLagQuery": "",
  "DetectClusterAliasQuery": "SELECT ifnull(max(cluster_name), '''') as cluster_alias from meta.cluster where anchor=1;",
  "DetectClusterDomainQuery": "SELECT ifnull(max(cluster_name), '''') as cluster_alias from meta.cluster where anchor=1;",
  "DetectInstanceAliasQuery": "",
  "DetectPromotionRuleQuery": "",
  "DataCenterPattern": "[.]([^.]+)[.][^.]+[.]mydomain[.]com",
  "PhysicalEnvironmentPattern": "[.]([^.]+[.][^.]+)[.]mydomain[.]com",
  "PromotionIgnoreHostnameFilters": [],
  "DetectSemiSyncEnforcedQuery": "",
  "ServeAgentsHttp": false,
  "AgentsServerPort": ":3001",
  "AgentsUseSSL": false,
  "AgentsUseMutualTLS": false,
  "AgentSSLSkipVerify": false,
  "AgentSSLPrivateKeyFile": "",
  "AgentSSLCertFile": "",
  "AgentSSLCAFile": "",
  "AgentSSLValidOUs": [],
  "UseSSL": false,
  "UseMutualTLS": false,
  "SSLSkipVerify": false,
  "SSLPrivateKeyFile": "",
  "SSLCertFile": "",
  "SSLCAFile": "",
  "SSLValidOUs": [],
  "URLPrefix": "",
  "StatusEndpoint": "/api/status",
  "StatusSimpleHealth": true,
  "StatusOUVerify": false,
  "AgentPollMinutes": 60,
  "UnseenAgentForgetHours": 6,
  "StaleSeedFailMinutes": 60,
  "SeedAcceptableBytesDiff": 8192,
  "PseudoGTIDPattern": "",
  "PseudoGTIDPatternIsFixedSubstring": false,
  "PseudoGTIDMonotonicHint": "asc:",
  "DetectPseudoGTIDQuery": "",
  "BinlogEventsChunkSize": 10000,
  "SkipBinlogEventsContaining": [],
  "ReduceReplicationAnalysisCount": true,
  "FailureDetectionPeriodBlockMinutes": 60,
  "FailMasterPromotionOnLagMinutes": 0,
  "AutoRecoverDeadSlaves": true,
  "RecoveryPeriodBlockSeconds": 3600,
  "AutoRecoverMaster": true,
  "AutoRecoverIntermediateMaster": true,
  "RecoveryIgnoreHostnameFilters": [],
  "RecoverMasterClusterFilters": ["*" ],
  "RecoverIntermediateMasterClusterFilters": ["*"],
  "AutoRecoverMasterCluster": true,
  "AutoRecoverIntermediateMasterCluster": true,
  "FailMasterWithSlaves": true,

  "OnFailureDetectionProcesses": [
    "echo 'Detected {failureType} on {failureCluster}. Affected replicas: {countSlaves}' >> /tmp/recovery.log"
  ],
  "PreGracefulTakeoverProcesses": [
    "echo 'Planned takeover about to take place on {failureCluster}. Master will switch to read_only' >> /tmp/recovery.log"
  ],
  "PreFailoverProcesses": [
    "echo 'Will recover from {failureType} on {failureCluster}' >> /tmp/recovery.log"
  ],
  "PostFailoverProcesses": [
    "echo '(for all types) Recovered from {failureType} on {failureCluster}. Failed: {failedHost}:{failedPort}; Successor: {successorHost}:{successorPort}' >> /tmp/recovery.log"
  ],
  "PostUnsuccessfulFailoverProcesses": [],
  "PostMasterFailoverProcesses": [
    "echo 'Recovered from {failureType} on {failureCluster}. Failed: {failedHost}:{failedPort}; Promoted: {successorHost}:{successorPort}' >> /tmp/recovery.log"
  ],
  "PostIntermediateMasterFailoverProcesses": [
    "echo 'Recovered from {failureType} on {failureCluster}. Failed: {failedHost}:{failedPort}; Successor: {successorHost}:{successorPort}' >> /tmp/recovery.log"
  ],
  "PostGracefulTakeoverProcesses": [
    "echo 'Planned takeover complete' >> /tmp/recovery.log"
  ],
  "CoMasterRecoveryMustPromoteOtherCoMaster": true,
  "DetachLostSlavesAfterMasterFailover": true,
  "ApplyMySQLPromotionAfterMasterFailover": true,
  "PreventCrossDataCenterMasterFailover": false,
  "PreventCrossRegionMasterFailover": false,
  "MasterFailoverDetachReplicaMasterHost": false,
  "MasterFailoverLostInstancesDowntimeMinutes": 0,
  "PostponeReplicaRecoveryOnLagMinutes": 0,
  "OSCIgnoreHostnameFilters": [],
  "GraphiteAddr": "",
  "GraphitePath": "",
  "GraphiteConvertHostnameDotsToUnderscores": true,
  "ConsulAddress": "",
  "ConsulAclToken": ""
}

```

start and enable orchestrator

```systemctl enable --now orchestrator```



