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
| Haproxy  | 會需要有loadbalance的服務來進行，如果環境內有如F5這一種配備，可以改用 |  
| DNS  | 建議使用windows DNS，注意正反解 |  
| quay(mirror-registry)  | 兩個設計是類似的，但是有授權相關的問題，所以本質上是不一樣的 |  

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
```
vim /etc/chrony.conf
```


重啟和enable chronyd  
```
systemctl enable --now chronyd  
systemctl status chronyd
```
