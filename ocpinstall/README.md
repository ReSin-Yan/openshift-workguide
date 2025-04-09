# OCP install  

ocpinstall 包含了幾個部分


#### 環境硬體配置(建議需求)  

使用的環境為  
OS      :RHEL 8(這邊使用rhel 9.5)  
CPU     :2cores    
Memory  :8 GB  
Disk    :500G  

#### 預先準備  
DNS正解，反解  



## 安裝  


### Install Mirror Registry  

將mirror-registry-amd64.tar  複製到os內(範例:/quay/registry.tar.gz , /quay/quayroot空間為主要放置image的路徑 )  
要注意空間要足夠  

```
tar -zxvf mirror-registry-amd64.tar.gz
./mirror-registry install
```

裝完後會得到如下的資訊,提供credentials  
```
INFO[2025-04-09 20:55:58] Quay installed successfully, config data is stored in ~/quay-install
INFO[2025-04-09 20:55:58] Quay is available at https://quay.resin.lab:8443 with credentials (init, vxQ2mZzWfp071lgMw5c8VNa49ISrO3H6)


podman login -u init -p 3lQso4O0Jx5zP976q8rHReuVIMk1DLi2 quay.resin.lab:8443 --tls-verify=false
```


接下來會通過反解自動設定FQND  
接下來可以通過網頁來進行連線  

### oc-mirror  

```
tar -zxvf oc-mirror.rhel9.tar.gz
chmod +x oc-mirror
mv oc-mirror /usr/local/bin/
```
