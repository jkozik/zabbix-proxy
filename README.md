# zabbix-proxy
Zabbix-proxy on k8s

For my home LAN, I have a VM running zabbix-proxy in docker containers. I want to retire this VM and deploy zabbix-proxy to my home kubernetes cluster.  

The official [zabbix repository](https://github.com/zabbix/zabbix-docker) has [kubernetes.yaml](https://github.com/zabbix/zabbix-docker/blob/5.4/kubernetes.yaml) file that deploys the full zabbix stack: server, agent, and proxy.  I only want the proxy part.
So I cut and pasted a deployment, service, and persistant volume yaml files.  First here's the service and deployment files:
```
[jkozik@dell2 zabbix-proxy]$ cat zabbix-proxy-dep.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: zabbix-proxy-sqlite3
  labels:
    app: zabbix
    tier: proxy
  namespace: zabbix
spec:
  selector:
    matchLabels:
      app: zabbix-proxy-sqlite3
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: zabbix-proxy-sqlite3
    spec:
      containers:
        - name: zabbix-proxy-sqlite3
          image: zabbix/zabbix-proxy-sqlite3:alpine-5.2-latest
          imagePullPolicy: Always
          ports:
          - containerPort: 10051
            protocol: TCP
            name: zabbix-trapper
          volumeMounts:
          - mountPath: /var/lib/zabbix/enc
            name: zabbixpsk
          env:
          - name: ZBX_HOSTNAME
            value: "k8szabbixproxy"
          - name: ZBX_SERVER_HOST
            value: "linode2.kozik.net"
          - name: ZBX_CONFIGFREQUENCY
            value: "60"
          - name: ZBX_TLSPSKIDENTITY
            value: "PSK 001"
          - name: ZBX_TLSPSKFILE
            value: zabbix_agentd.psk
          - name: ZBX_TLSCONNECT
            value: "psk"
      volumes:
      - name: zabbixpsk
        persistentVolumeClaim:
          claimName: zabbixpsk

[jkozik@dell2 zabbix-proxy]$ cat zabbix-proxy-svc.yaml
---
apiVersion: v1
kind: Service
metadata:
  name: zabbix-proxy-sqlite3
  labels:
    app: zabbix
  namespace: zabbix
spec:
  ports:
  - port: 10051
    targetPort: 10051
    name: zabbix-trapper
    protocol: TCP
  - port: 162
    targetPort: 1162
    protocol: UDP
    name: snmp-trap
  selector:
    name: zabbix-proxy-sqlite3
  #type: LoadBalancer
[jkozik@dell2 zabbix-proxy]$
```
Some notes on the above:
- The image is based on alpine.  The github says there are images also based on CentOS, rhel, ol, ubuntu
- The env defaults to a zabbix proxy running in Active mode.  The proxy does not get polled; it does the polling
- The zabbix proxy talks to the server using a pre-shared key. I chose to setup a Persistant Volume to hold the psk file
## Persistant Volume
On my host computer, I do NFS sharing.  I setup an NFS share that holds my psk file.  Here's my /etc/exports file:
```
[root@dell2 ~]# vi /etc/exports
...
/home/nfs/zabbix/enc 192.168.100.0/24(rw,sync,no_root_squash,no_all_squash,no_subtree_check,insecure)
[root@dell2 ~]# service nfs-server restart
Redirecting to /bin/systemctl restart nfs-server.service
[root@dell2 ~]#
```
I put my psk file in the directory: /home/nfs/zabbix/enc and the kubernetes cluster accesses this directory through the following Persistant Volume / Persistant Volume Claim:
```
[jkozik@dell2 zabbix-proxy]$ cat zabbix-proxy-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: zabbixpsk
  labels:
    app: zabbix
    tier: proxy
  namespace: zabbix
spec:
  accessModes:
    - ReadOnlyMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: nfs

[jkozik@dell2 zabbix-proxy]$ cat zabbix-proxy-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: zabbixpsk
  labels:
    app: zabbix
    tier: proxy
  namespace: zabbix
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadOnlyMany
  nfs:
    server: 192.168.101.152
    path: "/home/nfs/zabbix/enc"
  storageClassName: nfs
[jkozik@dell2 zabbix-proxy]$
```
## Deploy 
Here's an example deployment:
### Persistant Volume
```
[jkozik@dell2 zabbix-proxy]$ ls
zabbix-proxy-dep.yaml  zabbix-proxy-pvc.yaml  zabbix-proxy-pv.yaml  zabbix-proxy-svc.yaml

[jkozik@dell2 zabbix-proxy]$ kubectl -n zabbix apply -f zabbix-proxy-pv.yaml -f zabbix-proxy-pvc.yaml
persistentvolume/zabbixpsk created
persistentvolumeclaim/zabbixpsk created

[jkozik@dell2 zabbix-proxy]$ kubectl -n zabbix get pv,pvc
NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                  STORAGECLASS   REASON   AGE
persistentvolume/chwcom-persistent-storage                  1Gi        ROX            Retain           Bound    default/chwcom-persistent-storage      nfs                     82d
persistentvolume/mysql-persistent-storage                   1Gi        RWX            Retain           Bound    default/mysql-persistent-storage                               102d
persistentvolume/nfs-pv                                     1Gi        RWX            Retain           Bound    default/nfs-pvc                        nfs                     103d
persistentvolume/nginx-persistent-storage                   1Gi        RWX            Retain           Bound    default/nginx-persistent-storage                               102d
persistentvolume/nwcom-persistent-storage                   1Gi        ROX            Retain           Bound    default/nwcom-persistent-storage       nfs                     79d
persistentvolume/pvc-a13d1699-0a2f-422d-bdcf-c63246bb2ee4   10Gi       RWO            Delete           Bound    grafana/grafana                        nfs-client              38d
persistentvolume/weewx-archive                              1Gi        RWO            Retain           Bound    default/weewx-archive                  nfs                     14d
persistentvolume/weewx-conf                                 1Gi        RWO            Retain           Bound    default/weewx-conf                     nfs                     14d
persistentvolume/wordpress-persistent-storage               1Gi        RWX            Retain           Bound    default/wordpress-persistent-storage                           102d
persistentvolume/zabbixpsk                                  1Gi        ROX            Retain           Bound    zabbix/zabbixpsk                       nfs                     14s

NAME                              STATUS   VOLUME      CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/zabbixpsk   Bound    zabbixpsk   1Gi        ROX            nfs            14s
```
### Deployment and Service
```
[jkozik@dell2 zabbix-proxy]$ kubectl -n zabbix apply -f zabbix-proxy-dep.yaml -f zabbix-proxy-svc.yaml
deployment.apps/zabbix-proxy-sqlite3 created
service/zabbix-proxy-sqlite3 created
[jkozik@dell2 zabbix-proxy]$ kubectl -nzabbix get all
NAME                                        READY   STATUS    RESTARTS   AGE
pod/zabbix-proxy-sqlite3-7f6569dc75-b5kv8   1/1     Running   0          18s

NAME                           TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
service/zabbix-proxy-sqlite3   ClusterIP   10.106.222.193   <none>        10051/TCP,162/UDP   19s

NAME                                   READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/zabbix-proxy-sqlite3   1/1     1            1           19s

NAME                                              DESIRED   CURRENT   READY   AGE
replicaset.apps/zabbix-proxy-sqlite3-7f6569dc75   1         1         1       18s

[jkozik@dell2 zabbix-proxy]$ kubectl -nzabbix get pods -owide
NAME                                    READY   STATUS    RESTARTS   AGE   IP             NODE       NOMINATED NODE   READINESS GATES
zabbix-proxy-sqlite3-7f6569dc75-b5kv8   1/1     Running   0          19m   10.68.140.61   kworker3   <none>           <none>
```

### check log file
```
[jkozik@dell2 zabbix-proxy]$ kubectl -nzabbix logs pod/zabbix-proxy-sqlite3-7f6569dc75-b5kv8
Preparing Zabbix proxy
** Preparing Zabbix proxy configuration file
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "ProxyMode": ''...removed
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "Server": 'linode2.kozik.net'...updated
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "ServerPort": '10051'...added
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "Hostname": 'k8szabbixproxy'...updated
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "HostnameItem": ''...removed
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "ListenIP": ''...removed
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "ListenPort": ''...removed
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "SourceIP": ''...removed
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "LogType": 'console'...added
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "LogFile": ''...removed
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "LogFileSize": ''...removed
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "PidFile": ''...removed
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "DebugLevel": ''...removed
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "EnableRemoteCommands": ''...removed
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "LogRemoteCommands": ''...removed
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "DBHost": ''...removed
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "DBName": '/var/lib/zabbix/zabbix_proxy_db'...updated
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "DBUser": ''...removed
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "DBPort": ''...removed
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "DBPassword": ''...removed
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "VaultDBPath": ''...removed
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "VaultURL": ''...removed
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "ProxyLocalBuffer": ''...removed
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "ProxyOfflineBuffer": ''...removed
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "HeartbeatFrequency": ''...removed
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "ConfigFrequency": '60'...added
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "DataSenderFrequency": ''...removed
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "StatsAllowedIP": ''...removed
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "StartPreprocessors": ''...removed
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "StartPollers": ''...removed
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "StartIPMIPollers": ''...removed
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "StartPollersUnreachable": ''...removed
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "StartTrappers": ''...removed
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "StartPingers": ''...removed
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "StartDiscoverers": ''...removed
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "StartHTTPPollers": ''...removed
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "JavaGateway": ''...removed
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "JavaGatewayPort": ''...removed
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "StartJavaPollers": ''...removed
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "StartVMwareCollectors": ''...removed
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "VMwareFrequency": ''...removed
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "VMwarePerfFrequency": ''...removed
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "VMwareCacheSize": ''...removed
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "VMwareTimeout": ''...removed
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "SNMPTrapperFile": ''...removed
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "StartSNMPTrapper": ''...removed
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "HousekeepingFrequency": ''...removed
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "CacheSize": ''...removed
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "StartDBSyncers": ''...removed
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "HistoryCacheSize": ''...removed
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "HistoryIndexCacheSize": ''...removed
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "Timeout": ''...removed
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "TrapperTimeout": ''...removed
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "UnreachablePeriod": ''...removed
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "UnavailableDelay": ''...removed
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "UnreachableDelay": ''...removed
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "AlertScriptsPath": '/usr/lib/zabbix/alertscripts'...added
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "ExternalScripts": '/usr/lib/zabbix/externalscripts'...added
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "FpingLocation": '/usr/sbin/fping'...added
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "Fping6Location": ''...removed
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "SSHKeyLocation": '/var/lib/zabbix/ssh_keys'...added
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "LogSlowQueries": ''...removed
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "SSLCertLocation": '/var/lib/zabbix/ssl/certs/'...added
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "SSLKeyLocation": '/var/lib/zabbix/ssl/keys/'...added
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "SSLCALocation": '/var/lib/zabbix/ssl/ssl_ca/'...added
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "LoadModulePath": '/var/lib/zabbix/modules/'...added
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "TLSConnect": 'psk'...added
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "TLSAccept": ''...removed
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "TLSCAFile": ''...removed
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "TLSCRLFile": ''...removed
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "TLSServerCertIssuer": ''...removed
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "TLSServerCertSubject": ''...removed
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "TLSCertFile": ''...removed
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "TLSCipherAll": ''...removed
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "TLSCipherAll13": ''...removed
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "TLSCipherCert": ''...removed
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "TLSCipherCert13": ''...removed
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "TLSCipherPSK": ''...removed
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "TLSCipherPSK13": ''...removed
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "TLSKeyFile": ''...removed
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "TLSPSKIdentity": '****'. Enable DEBUG_MODE to view value ...added
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "TLSPSKFile": 'zabbix_agentd.psk'...added
** Updating '/etc/zabbix/zabbix_proxy.conf' parameter "User": 'zabbix'...added
Starting Zabbix Proxy (active) [k8szabbixproxy]. Zabbix 5.2.7 (revision 91e8333).
Press Ctrl+C to exit.

     7:20211004:191334.809 Starting Zabbix Proxy (active) [k8szabbixproxy]. Zabbix 5.2.7 (revision 91e8333).
     7:20211004:191334.809 **** Enabled features ****
     7:20211004:191334.809 SNMP monitoring:       YES
     7:20211004:191334.809 IPMI monitoring:       YES
     7:20211004:191334.809 Web monitoring:        YES
     7:20211004:191334.809 VMware monitoring:     YES
     7:20211004:191334.809 ODBC:                  YES
     7:20211004:191334.809 SSH support:           YES
     7:20211004:191334.809 IPv6 support:          YES
     7:20211004:191334.809 TLS support:           YES
     7:20211004:191334.809 **************************
     7:20211004:191334.809 using configuration file: /etc/zabbix/zabbix_proxy.conf
     7:20211004:191334.809 cannot open database file "/var/lib/zabbix/zabbix_proxy_db": [2] No such file or directory
     7:20211004:191334.809 creating database ...
     7:20211004:191338.607 current database version (mandatory/optional): 05020000/05020002
     7:20211004:191338.607 required mandatory version: 05020000
     7:20211004:191338.608 proxy #0 started [main process]
   162:20211004:191338.608 proxy #1 started [configuration syncer #1]
   163:20211004:191338.710 proxy #2 started [trapper #1]
   164:20211004:191338.710 proxy #3 started [trapper #2]
   169:20211004:191338.711 proxy #8 started [data sender #1]
   170:20211004:191338.712 proxy #9 started [housekeeper #1]
   168:20211004:191338.712 proxy #7 started [heartbeat sender #1]
   177:20211004:191338.716 proxy #16 started [self-monitoring #1]
   165:20211004:191338.717 proxy #4 started [trapper #3]
   178:20211004:191338.718 proxy #17 started [task manager #1]
   176:20211004:191338.718 proxy #15 started [history syncer #4]
   179:20211004:191338.720 proxy #18 started [poller #1]
   166:20211004:191338.722 proxy #5 started [trapper #4]
   180:20211004:191338.723 proxy #19 started [poller #2]
   167:20211004:191338.724 proxy #6 started [trapper #5]
   181:20211004:191338.725 proxy #20 started [poller #3]
   171:20211004:191338.726 proxy #10 started [http poller #1]
   182:20211004:191338.727 proxy #21 started [poller #4]
   172:20211004:191338.727 proxy #11 started [discoverer #1]
   173:20211004:191338.729 proxy #12 started [history syncer #1]
   183:20211004:191338.730 proxy #22 started [poller #5]
   174:20211004:191338.730 proxy #13 started [history syncer #2]
   175:20211004:191338.730 proxy #14 started [history syncer #3]
   184:20211004:191338.731 proxy #23 started [unreachable poller #1]
   186:20211004:191338.732 proxy #25 started [preprocessing manager #1]
   189:20211004:191338.733 proxy #28 started [preprocessing worker #3]
   185:20211004:191338.736 proxy #24 started [icmp pinger #1]
   187:20211004:191338.736 proxy #26 started [preprocessing worker #1]
   188:20211004:191338.736 proxy #27 started [preprocessing worker #2]
   162:20211004:191338.858 received configuration data from server at "linode2.kozik.net", datalen 73589
   162:20211004:191439.159 received configuration data from server at "linode2.kozik.net", datalen 73589
[jkozik@dell2 zabbix-proxy]$
```
Note:  When I first did this, I had a series of problems:
- The version of the proxy server and the zabbix server needs to be the same.  I was initially trying to connect a 5.4 proxy to a 5.2 server. 
- Preshared keys.  I initially setup the proxy without preshared keys.  Good for debugging. But I switched to PSKs right away.  Errors with PSKs show in the logs.
- Agents in my home network were using my old proxy server.  I have to put the IP address of this new zabbix-proxy on the Server= line of the zabbix-agent.conf file.

### zabbix-agent.conf

The zabbix-proxy deployed here is configured by default to be an active proxy.  That is, the proxy queries the server and receives a list of clients that it should poll.  And then the proxy polls those agents.
Thus in my setup here, the zabbix-proxy has no ingress traffic.  It could, but not the way I am using it.  
The zabbix-agent.conf for each of the hosts served needs to know the IP address of the zabbix-proxy.  What IP address should I put there?  The pod running the zabbix-proxy is running on my node called kworker3, 10.68.140.61. A query to an agent on my homeLAN starts at this IP address, then gets SNAT'd to the ipaddress of kworker3 (192.168.100.176).
Thus the zabbix-agent.conf file needs a line that says Server=192.168.100.176. 

But, my zabbix-proxy is running on a three node cluster. The pod could move to any of the other ndoes; thus, the line should be Server=192.168.100.173,192.168.100.174,192.168.100.175,192.168.100.176

### Configure zabbix server
To get a proxy working, the server side needs to be setup.  From the GUI, click on Administration/Proxy, Create Proxy.  Select Active, set the name to agree with zabbix-proxy-deploy.yaml. Under encryption, again enter the same data as the yaml.



