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
firewall-cmd --add-port={6443,22623,9000}/tcp --permanent && firewall-cmd --reload
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
  - cidr: 192.168.0.0/16
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
envsubst < install-config.yaml.bak > /root/workspace/installocp/install-config.yaml
```
