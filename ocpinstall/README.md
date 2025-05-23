# OCP install  

ocpinstall 包含了幾個部分  

 | 配置的虛擬機 | 用處 | 選用OS版本 |
|-------|-------|------|
| bastion| 堡壘機，通常都會通過此台機器進行最基礎的設定，也會通過此來進行連結ocp的動作 | RHEL9.4 |
| Haproxy  | 附載平衡主機，用來安裝haproxy |  RHEL9.4 |
| DNS  | DNS Server |  windows server |
| quay(mirror-registry)  | 容器倉庫 |  RHEL9.4 |


 | 配置功能| 安裝位置 |  用處 | 
|-------|-------|-------|
| http |  Bastion | ocp安裝過程中會需要用到的服務，ocp安裝類似於pxe boot，會從一個存放點下載設定檔案近來進行設定(此檔案為.ign檔案) |
| Haproxy  |  Haproxy | 會需要有loadbalance的服務來進行，如果環境內有如F5這一種配備，可以改用 |  
| DNS  |  DNS | 建議使用windows DNS，注意正反解 |  
| quay(mirror-registry)  |  quay | 兩個設計是類似的，但是有授權相關的問題，所以本質上是不一樣的 |  

#### 環境硬體配置(建議需求)  

bastion  

 | 規格 |  | 
|-------|-------|
| OS | RHEL 8(這邊使用rhel 9.5) |
| CPU |  4cores |
| Memory  | 8 GB |  
| Disk  | 300G |  

http server (重要性不高，與bastion放在一起)  

Haproxy(建議確認route數量)  

 | 規格 |  | 
|-------|-------|
| OS | RHEL 8(這邊使用rhel 9.5) |
| CPU |  4cores |
| Memory  | 8 GB |  
| Disk  | 300G |  

SELinux: 每台都是Enforcing!  
bootstrap和master強制要用rhcos，worker則rhcos或rhel都可  

### Install bastion  

打開bastion的防火牆  

```
firewall-cmd --add-port={53,80,443,3389,5000,6443,8080,9000,22623}/tcp --permanent && firewall-cmd --reload
firewall-cmd --add-port={53,514}/udp --permanent && firewall-cmd --reload
firewall-cmd --add-service={dns,http,https,ntp,mountd,rpc-bind,nfs} --permanent && firewall-cmd --reload
```

把pool那行註解掉，然後新增server time.stdtime.gov.tw iburst ，或是自己的ntp server(可以從windows server內部來進行設定，但是一定要有) 
allow 192.168.50.0/24  
```
vim /etc/chrony.conf
```


重啟和enable chronyd  
```
systemctl enable --now chronyd  
systemctl status chronyd
```

安裝Apahe(http server)  
主要會作為放置.ign檔案以及存放raw.gz檔案的位置  
```
dnf -y install httpd
```
把port改成8080，避免跟haproxy衝到!  
```
vim /etc/httpd/conf/httpd.conf
```

```
Listen 8080
```

start和enable httpd
```
systemctl enable httpd --now
systemctl status httpd
```

把之前載下來的raw.gz放到/var/www/html/底下建立的資料夾以供之後下載  
```
mkdir -p /var/www/html/ocpinstall
```

檢查selinux context，需改成httpd_sys_content_t  
```
ll -Z /var/www/html/ocpinstall
restorecon -v rhcos-4.xx.0-x86_64-metal.x86_64.raw.gz
```

放入oc以及oc-install檔案到bastion上  
並且解壓縮之後複製到/usr/local/bin下面  
```
cd
mkdir workspace
cd workspace/
```
將檔案上傳至此資料夾  

```
tar -xvf openshift-install-linux-4.14.34.tar.gz
tar -xvf oc-4.14.34-linux.tar.gz
cp kubectl oc openshift-install /usr/local/bin
sudo rm *
```

產生SSH KEY，以便日後連線到各node //因為rhcos限制不讓你用帳密的方式連，強制要用key的方式連，且要用core這個user來登入  

```
cd
mkdir workspace/keys
cd workspace/keys/
ssh-keygen -t ed25519 -N '' -f ocpssh
eval "$(ssh-agent -s)"
ssh-add ocpssh
```


複製pull secret，並在secret裡建立quay的帳密  
https://console.redhat.com/openshift/install/pull-secret  
Copy pull secret後，把複製下來的東西貼到pull-secret  
貼上剛剛Copy pull secret的東西  
```
mkdir -p /ocp/pull-sec
vim /ocp/pull-sec/pull-secret
```

存檔離開後，把pull-secret轉成json檔  
```
cat pull-secret | jq > pull-secret.json
```

在最上方新增地端quay的資訊  
其中auth來自於  
```
echo -n 'admin:P@ssw0rd' | base64
```

範例如下，注意連線到redhat網站的會有時間限制  
```
{
  "auths": {
    "quay.resin.lab": {
      "auth": "YWRtaW46UEBzc3cwcmQ=",
      "email": "resin.yan@kyndryl.com"
    },
    "cloud.openshift.com": {
      "auth": "b3BlbnNoaWZ0LXJlbGVhc2UtZGV2K29jbV9hY2Nlc3NfNWRjY2M4ZmU2MWYxNGIzMGEyNTRjOGQ3NmMwODU0YTY6SlgzM0g0QVFWWVFUNzk1S1QwODU5MTNCTDdJS1ZXTjdQU1pIUDlRSzlISTFSV0Q0RE41WUNQSjhZWTdMQ0NMNw==",
      "email": "resin.yan@kyndryl.com"
    },
    "quay.io": {
      "auth": "b3BlbnNoaWZ0LXJlbGVhc2UtZGV2K29jbV9hY2Nlc3NfNWRjY2M4ZmU2MWYxNGIzMGEyNTRjOGQ3NmMwODU0YTY6SlgzM0g0QVFWWVFUNzk1S1QwODU5MTNCTDdJS1ZXTjdQU1pIUDlRSzlISTFSV0Q0RE41WUNQSjhZWTdMQ0NMNw==",
      "email": "resin.yan@kyndryl.com"
    },
    "registry.connect.redhat.com": {
      "auth": "fHVoYy1wb29sLTYxYTBhYTg2LWUxNmEtNGRkZS1iMmI0LTRhZjgwNTY4NDg2NjpleUpoYkdjaU9pSlNVelV4TWlKOS5leUp6ZFdJaU9pSXpaamxoWm1NM04yTXlNVEkwTkRJeVlUQXdaRGt3T1RFMU1XSTVaREJsTlNKOS5jYXIwYXlMOTQ1Y3FlVzFVUjdyV09kNEhnT0lXeUtZT0RWdnEwMDZFRWcwUGtKOXhXM2huZFpkV0VrZnRtdURDMzlSZHdZUmFIaHRmY003bUdIY2piVWY0VDlqcUk2TUtCV1ZOZ1VOb2JjWWZWdkhmOGlLUDhMYjlpcnZkclYzVkJ3blVTajk3UGUxVmhEdUxVWmtlWmtoMzZ2RVR2RDBuNE1RanN2dDZiS0w4dVBhMzU5RFpsRFhWR2d1YWV2UGtGNk9rZUEySlA0cC1ET3Jna29RWGRxV3JHcVB3V0p6d1E3WjROa0MwQVVIQ0Qxc0JpME5Hd2xVRHVOd3pWaHFaZGhKSUEzS3FmVkxCdzdSMFNiQjhDc0tpSG1FQU5iMTBfaU1FYUNEdWxZSW1Xc2NQYmFfOHByNXNuWW1SY3NkQU5jYTVLR2tXUmdYck13ekFHajVpV05kaFhwbm1DUlA5ZWFtZlE1bWhrMER1aEJLN1RhRmFLTFZ3b05qS1R3b0pDbUlOR0RNVWVWbGFyWmg0LWEyZWJaY2U5MEhCcUNFbmtTUkV5R1JxTk1KcnhvOHZTQl9yREN1LTAxMmdPR0VoQWJTSUJ0NVZFSlNlY1NUUHlkU2ZvTEwxVkh3S1RXbGVYQ2U4d0NDVXhoZVdxbmkyNWtGMmNadGxFMUtJWlItY01FVmp3VmttcWNrb1dwdmdHcy13YWttdE96TDIwZl9TTHV2c0pxMElzOXhoeTFzNUgxYlllTVczN1dJS2JJUUVLc292TXRCZktmbHFPSFE2V3ROWG5XWVdHUmhvVGJDQzlENmhDMUphV21ob1ZzYnJkWklodW8xbTZBaXU3UGdjU25jUWhrWUpCY1NrWDJzM3R0OXVBcml5bXN5d3NTczE5MmZIOWQzeGg1cw==",
      "email": "resin.yan@kyndryl.com"
    },
    "registry.redhat.io": {
      "auth": "fHVoYy1wb29sLTYxYTBhYTg2LWUxNmEtNGRkZS1iMmI0LTRhZjgwNTY4NDg2NjpleUpoYkdjaU9pSlNVelV4TWlKOS5leUp6ZFdJaU9pSXpaamxoWm1NM04yTXlNVEkwTkRJeVlUQXdaRGt3T1RFMU1XSTVaREJsTlNKOS5jYXIwYXlMOTQ1Y3FlVzFVUjdyV09kNEhnT0lXeUtZT0RWdnEwMDZFRWcwUGtKOXhXM2huZFpkV0VrZnRtdURDMzlSZHdZUmFIaHRmY003bUdIY2piVWY0VDlqcUk2TUtCV1ZOZ1VOb2JjWWZWdkhmOGlLUDhMYjlpcnZkclYzVkJ3blVTajk3UGUxVmhEdUxVWmtlWmtoMzZ2RVR2RDBuNE1RanN2dDZiS0w4dVBhMzU5RFpsRFhWR2d1YWV2UGtGNk9rZUEySlA0cC1ET3Jna29RWGRxV3JHcVB3V0p6d1E3WjROa0MwQVVIQ0Qxc0JpME5Hd2xVRHVOd3pWaHFaZGhKSUEzS3FmVkxCdzdSMFNiQjhDc0tpSG1FQU5iMTBfaU1FYUNEdWxZSW1Xc2NQYmFfOHByNXNuWW1SY3NkQU5jYTVLR2tXUmdYck13ekFHajVpV05kaFhwbm1DUlA5ZWFtZlE1bWhrMER1aEJLN1RhRmFLTFZ3b05qS1R3b0pDbUlOR0RNVWVWbGFyWmg0LWEyZWJaY2U5MEhCcUNFbmtTUkV5R1JxTk1KcnhvOHZTQl9yREN1LTAxMmdPR0VoQWJTSUJ0NVZFSlNlY1NUUHlkU2ZvTEwxVkh3S1RXbGVYQ2U4d0NDVXhoZVdxbmkyNWtGMmNadGxFMUtJWlItY01FVmp3VmttcWNrb1dwdmdHcy13YWttdE96TDIwZl9TTHV2c0pxMElzOXhoeTFzNUgxYlllTVczN1dJS2JJUUVLc292TXRCZktmbHFPSFE2V3ROWG5XWVdHUmhvVGJDQzlENmhDMUphV21ob1ZzYnJkWklodW8xbTZBaXU3UGdjU25jUWhrWUpCY1NrWDJzM3R0OXVBcml5bXN5d3NTczE5MmZIOWQzeGg1cw==",
      "email": "resin.yan@kyndryl.com"
    }
  }
}
```

把pull-secret.json複製到~/.docker/下以便認證，檔名改為config.json  

```
mkdir -p ~/.docker
cp pull-secret.json ~/.docker/config.json
```


從quay那台上面，把之前產的rootCA.pem複製到這邊(bastiondns)的/etc/pki/ca-trust/source/anchors/  
```
scp rootCA.pem root@192.168.50.103:/etc/pki/ca-trust/source/anchors/
```

檢查  
```
update-ca-trust extract
```



### Install Haproxy(if need)  

打開Haproxy的防火牆  

```
firewall-cmd --add-port={80,443,6443,22623,9000}/tcp --permanent && firewall-cmd --reload
semanage port --add --type http_port_t --proto tcp 6443
semanage port --add --type http_port_t --proto tcp 22623
semanage port -l | grep http_port_t
```

SELinux確保關閉
```
setsebool -P haproxy_connect_any=1
```

安裝haproxy  
建議網路能夠對外比較方便安裝  
```
dnf -y install haproxy 
```
驗證安裝完成  
```
getsebool haproxy_connect_any 
```

設定haproxy  
```
vim /etc/haproxy/haproxy.cfg
```

這邊會將haproxy作為ocp的vip  
由於設計的關係，也會將這一台haproxy作為route的設定  
所以設定參考如下  
如果使用的話請修改自行的設定(主要是IP,hostname,以及應用vip以及route)  

```
#---------------------------------------------------------------------
# Example configuration for a possible web application.  See the
# full configuration options online.
#
#   https://www.haproxy.org/download/1.8/doc/configuration.txt
#
#---------------------------------------------------------------------

#---------------------------------------------------------------------
# Global settings
#---------------------------------------------------------------------
global
    # to have these messages end up in /var/log/haproxy.log you will
    # need to:
    #
    # 1) configure syslog to accept network log events.  This is done
    #    by adding the '-r' option to the SYSLOGD_OPTIONS in
    #    /etc/sysconfig/syslog
    #
    # 2) configure local2 events to go to the /var/log/haproxy.log
    #   file. A line like the following can be added to
    #   /etc/sysconfig/syslog
    #
    #    local2.*                       /var/log/haproxy.log
    #
    log         127.0.0.1 local2

    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon

    # turn on stats unix socket
    stats socket /var/lib/haproxy/stats

    # utilize system-wide crypto-policies
    ssl-default-bind-ciphers PROFILE=SYSTEM
    ssl-default-server-ciphers PROFILE=SYSTEM

#---------------------------------------------------------------------
# common defaults that all the 'listen' and 'backend' sections will
# use if not designated in their block
#---------------------------------------------------------------------
defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000
#---------------------------------------------------------------------
#frontend stats
#---------------------------------------------------------------------
frontend stats
  bind *:9000
  mode            http
  log             global
  maxconn 10
  stats enable
  stats hide-version
  stats refresh 30s
  stats show-node
  stats show-desc Stats for ocp cluster
  stats auth admin:P@ssw0rd
  stats uri /stats
  stats admin if TRUE
#---------------------------------------------------------------------
#openshift-api-server
#---------------------------------------------------------------------
frontend openshift-api-server
  bind *:6443
  default_backend openshift-api-server
  mode tcp
  option tcplog

backend openshift-api-server
  balance source
  mode tcp
  server master1 master1.resin.lab:6443 check
  server master2 master2.resin.lab:6443 check
  server master3 master3.resin.lab:6443 check
  server bootstrap bootstrap.resin.lab:6443 check

#---------------------------------------------------------------------
#machine-config-server
#---------------------------------------------------------------------
frontend machine-config-server
  bind *:22623
  default_backend machine-config-server
  mode tcp
  option tcplog

backend machine-config-server
  balance source
  mode tcp
  server bootstrap bootstrap.resin.lab:22623 check inter 1s backup
  server master1 master1.resin.lab:22623 check
  server master2 master2.resin.lab:22623 check
  server master3 master3.resin.lab:22623 check

#---------------------------------------------------------------------
#ingress-http
#---------------------------------------------------------------------
frontend ingress-http
  bind *:80
  default_backend ingress-http
  mode tcp
  option tcplog

backend ingress-http
  balance source
  mode tcp
  server router1 router1.resin.lab:80 check
  server router2 router2.resin.lab:80 check

#---------------------------------------------------------------------
#ingress-https
#---------------------------------------------------------------------
frontend ingress-https
  bind *:443
  default_backend ingress-https
  mode tcp
  option tcplog

backend ingress-https
  balance source
  mode tcp
  server router1 router1.resin.lab:443 check
  server router2 router2.resin.lab:443 check
```


改好後，start和enable haproxy  
確認好狀態ctrl+c離開  
```
systemctl enable haproxy.service --now
systemctl status haproxy.service
```

設定log  
```
vim /etc/haproxy/haproxy.cfg
```
log  127.0.0.1 local2 這行要打開  
預設已經開了  

設定rsyslog.conf  
```
vim /etc/rsyslog.conf
```
在MODULES這邊新增  
```
$ModLoad imudp
$UDPServerRun 514
$UDPServerAddress 127.0.0.1
```
在RULES這邊新增一行  
```
local2.*                       /var/log/haproxy.log
```

restart rsyslog & haproxy  
```
systemctl restart rsyslog
systemctl status rsyslog
systemctl restart haproxy
```

接下來在同網段的位置利用網頁IP進行驗證  
預設port號是9000  

[haproxyIP:9000/stats](http://192.168.50.105:9000/stats "link")  




### Install ocp    



先建立含有變數的install-config.yaml.bak  

```
cd
mkdir workspace/installocp/install-config
cd workspace/installocp/install-config/
vim install-config.yaml.bak
```

```
apiVersion: v1
baseDomain: resin.lab
compute:
- hyperthreading: Enabled
  name: worker
  replicas: 0
controlPlane:
  hyperthreading: Enabled
  name: master
  replicas: 3
metadata:
  name: ocp4
networking:
  clusterNetwork:
  - cidr: 172.10.0.0/16
    hostPrefix: 24
  networkType: OVNKubernetes
  serviceNetwork:
  - 10.244.0.0/16
platform:
  none: {}
fips: false
pullSecret: '${PULL_SECRET}'
sshKey: '${OCP_SSH_KEY}'
additionalTrustBundle: |
${ROOT_CA}
ImageDigestSources:
- mirrors:
  - quay.resin.lab:8443/ocp414/ocp-release
  source: quay.io/openshift-release-dev/ocp-release
- mirrors:
  - quay.resin.lab:8443/ocp414/ocp-release
  source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
```

建立填入參數的文件  
```
sed 's/^/  /g' /etc/pki/ca-trust/source/anchors/rootCA.pem > /root/workspace/keys/reform.txt
```


建立env_value  
```
cd 
vim env_value
```

貼入以下資訊  
```
export PULL_SECRET=$(cat /root/workspace/keys/pull-secret.json | jq -c)
export OCP_SSH_KEY=$(cat /root/workspace/keys/ocpssh.pub)
export ROOT_CA=$(cat /root/workspace/keys/reform.txt)
```

```
source env_value
```

產生正確install_config檔案  
```
cd workspace/installocp/install-config/
envsubst < install-config.yaml.bak > install-config.yaml
cp install-config.yaml ../
```

產生manifests  
```
openshift-install create manifests
sed -i 's/mastersSchedulable: true/mastersSchedulable: false/g' manifests/cluster-scheduler-02-config.yml
```

產生ignition檔案  
```
openshift-install create ignition-configs
```

將.ign檔案複製到http server  
```
chmod 755 *.ign
cp *.ign /var/www/html/ocpinstall/
```

在/var/www/html/ocpinstall建立安裝腳本，以利bootstrap, master, worker node安裝  

```
cd /var/www/html/ocpinstall
vim install.sh
```

貼上  
```
set -x

BASTION_IP="192.168.50.103:8080"

sudo coreos-installer install /dev/sda --insecure --insecure-ignition -u http://${BASTION_IP}/ocpinstall/rhcos-4.14.34-x86_64-metal.x86_64.raw.gz -I http://${BASTION_IP}/ocpinstall/${CLUSTER_ROLE}.ign --firstboot-args 'rd.neednet=1' --copy-network
```

bootstrap 虛擬機  
開機完設定好網路  

分別確認反解、DNS  
之後跟bastion 利用curl的方式拿取ign檔案  
之後進行設定   
```
dig -x [ip]
export CLUSTER_ROLE=bootstrap
curl http://192.168.50.103:8080/ocpinstall/install.sh | sh

reboot
```



master 虛擬機(會有三台)  
開機完設定好網路  

分別確認反解、DNS  
之後設定
```
dig -x [ip]
export CLUSTER_ROLE=master
curl http://192.168.50.103:8080/ocpinstall/install.sh | sh

reboot
```


剩下所有的 虛擬機(會有N台)都設定為worker  
開機完設定好網路  

分別確認反解、DNS  
之後設定
```
dig -x [ip]
export CLUSTER_ROLE=worker
curl http://192.168.50.103:8080/ocpinstall/install.sh | sh

reboot
```


進行基礎設定  
先將kubeconfig複製出來   
```
cp auth/kubeconfig /root/.kube/config
source <(oc completion bash)
oc completion bash > oc_bash_completion
mv oc_bash_completion /etc/bash_completion.d/

```

Approce csr  
reboot後，回到bastion上，監控pending的csr  
```
watch -n 2 "oc get csr | grep -i pending"
```

另外開啟一個連線到bastion的畫面   
依序approve看到的csr  
```
oc get csr |grep -i pending | awk '{print $1}' | xargs -I {} oc adm certificate approve {}
```
大約會重複兩到三次  

接下來通過  
```
watch -n 0.5 oc get node
```
確保所有節點ready




所有node安裝完後  
調整ingresscontrollers裡的nodeselector的label  
```
oc edit ingresscontrollers -n openshift-ingress-operator
```
要在spec修改成如下  
```
spec:
  clientTLS:
    clientCA:
      name: ""
    clientCertificatePolicy: ""
  httpCompression: {}
  httpEmptyRequestsPolicy: Respond
  httpErrorCodePages:
    name: ""
  nodePlacement:
    nodeSelector:
      matchLabels:
         ingress: default
  replicas: 2
  tuningOptions:
    reloadInterval: 0s
  unsupportedConfigOverrides: null
```

然後給router node打標籤，打ingress=default  
重啟服務之後、確認有長在route node上  
```
oc label node route1.resin.lab ingress=default
oc label node route2.resin.lab ingress=default
oc get no --show-labels |grep ingress
oc rollout restart deployment router-default -n openshift-ingress
oc get po -n openshift-ingress -o wide
```


更改worker在oc get no的ROLES及設定各node的machine config pool  
```
oc label nodes route1.resin.lab node-role.kubernetes.io/worker-
oc label nodes route2.resin.lab node-role.kubernetes.io/worker-
oc label nodes worker1.resin.lab node-role.kubernetes.io/worker-
oc label nodes worker2.resin.lab node-role.kubernetes.io/worker-
oc label nodes logger1.resin.lab node-role.kubernetes.io/worker-
oc label nodes logger2.resin.lab node-role.kubernetes.io/worker-


oc label nodes route1.resin.lab node-role.kubernetes.io/router=""
oc label nodes route2.resin.lab node-role.kubernetes.io/router=""
oc label nodes worker1.resin.lab node-role.kubernetes.io/worker=""
oc label nodes worker2.resin.lab node-role.kubernetes.io/worker=""
oc label nodes logger1.resin.lab node-role.kubernetes.io/logger=""
oc label nodes logger2.resin.lab node-role.kubernetes.io/logger=""

oc get node
```

建資料夾存放machine config相關的file  
建立router node的mcp和logger node的mcp  
```
vim router-mcp.yaml
```

貼入如下  
```
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfigPool
metadata:
  name: router
spec:
  machineConfigSelector:
    matchExpressions:
    - key: machineconfiguration.openshift.io/role
      operator: In
      values:
      - worker
      - router
  nodeSelector:
    matchLabels:
      node-role.kubernetes.io/router: ""
  paused: false
```

```
vim logger-mcp.yaml
```

貼入如下  
```
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfigPool
metadata:
  name: logger
spec:
  machineConfigSelector:
    matchExpressions:
    - key: machineconfiguration.openshift.io/role
      operator: In
      values:
      - worker
      - logger
  nodeSelector:
    matchLabels:
      node-role.kubernetes.io/logger: ""
```


分別部屬下去  
```
oc create -f router-mcp.yaml --save-config=true
oc create -f logger-mcp.yaml --save-config=true
```



編寫chrony.conf檔案  
讓每一台對準ntp server  
```
server 192.168.50.110 iburst
driftfile /var/lib/chrony/drift
makestep 1.0 3
rtcsync
logdir /var/log/chrony
```

建立chrony-template.yaml  

```
vim chrony-template.yaml
```

貼入如下  
```
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: ${MC_ROLE}
  name: 99-${MC_ROLE}s-chrony-configuration
spec:
  config:
    ignition:
      config: {}
      security:
        tls: {}
      timeouts: {}
      version: 3.4.0
    networkd: {}
    passwd: {}
    storage:
      files:
      - contents:
          source: data:text/plain;charset=utf-8;base64,${CHRONY_BASE64}
        mode: 420
        overwrite: true
        path: /etc/chrony.conf
  osImageURL: ""
```

之後分別產生master以及worker  
```
export CHRONY_BASE64=$(cat /root/workspace/installocp/mc-config/chrony.conf | base64 -w 0)
export MC_ROLE="master"
envsubst < chrony-template.yaml > 99-master-chrony.yaml
export MC_ROLE="worker"
envsubst < chrony-template.yaml > 99-worker-chrony.yaml

oc create -f  99-master-chrony.yaml
oc create -f  99-worker-chrony.yaml
```

建立別的admin，名稱叫admin  
並且新增密碼  
```
oc create user admin
oc adm policy add-cluster-role-to-user cluster-admin admin
htpasswd -c -B -b users.htpasswd admin P@ssw0rd
```

為oauth設置htpasswd認證  
```
oc create secret generic htpass-secret --from-file=htpasswd=users.htpasswd -n openshift-config
```

編輯oauth  
```
oc edit oauth
```

輸入如下
```
identityProviders:
  - name: htpass-login
    mappingMethod: claim
    type: HTPasswd
    htpasswd:
      fileData:
        name: htpass-secret
```
之後等待幾分鐘之後重開頁面   


oc login 測試新帳號  
正常會遇到問題  
原因是因為從bastion登入原本的帳號沒問題，但是另外的帳號就需要CA憑證  
同理如果要在其他虛擬機進行登入  
一樣要把憑證放入其他虛擬機內  
```
oc login -u admin
```
跳出x509 error  


獲取憑證  
```
oc project openshift-authentication
oc get po
oc rsh oauth-openshift-xxxx  cat /run/secrets/kubernetes.io/serviceaccount/ca.crt > ocp4-ingress-ca.crt
mv ocp4-ingress-ca.crt /etc/pki/ca-trust/source/anchors/
update-ca-trust extract
```

移除kubeadmin，並建立別的admin帳號(可選)  
```
oc delete secrets kubeadmin -n kube-system
```



