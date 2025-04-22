# Operator install  

通常在不對外的環境中  
operator會需要另外設定  
在ocp4.10之後可以使用oc-mirror來進行設定  


## oc-mirror  
所有的mirror都會從這邊進行  
分為兩種方式  
第一種為，下指令的這一台機器可以對外，並且可以直接連線到quay  
第二種為，下指令的這一台機器可以對外，並且不能直接連線到quay  

下指令的機器都需要能夠對外  
簡單一點的方式也不用寫pull secret  
直接登入redhat官方repo  
```
podman login
```

###  通過指令確認需要下載的package  

可以參考同一個文件夾下面的資料，裡面有對應的operator name以及channel  
接下來會用4.14以及安裝一個nfd當作範例  
```
oc-mirror list operators --catalog registry.redhat.io/redhat/redhat-operator-index:v4.14
```

會顯示如下(擷取部分)  

```
NAME                                          DISPLAY NAME                                             DEFAULT CHANNEL
netobserv-operator                            Network Observability                                    stable
nfd                                           Node Feature Discovery Operator                          stable
node-healthcheck-operator                     Node Health Check Operator                               stable
```

知道他的NAME以及channel之後，需要獲得他的版本號  

```
oc-mirror list operators --catalog registry.redhat.io/redhat/redhat-operator-index:v4.14 --package=nfd --channel=stable
```

會顯示如下(擷取部分)  

```
VERSIONS
4.14.0-202503310939
4.14.0-202502111935
4.14.0-202501280211
4.14.0-202502191335
4.14.0-202501150635
```

之後就可以通過此檔案進行配置ImageSetConfiguration.yaml  

```
apiVersion: mirror.openshift.io/v1alpha2
kind: ImageSetConfiguration
storageConfig:
  local:
    path: /root/resin_workspace/pullimages/operator
mirror:
  operators:
    - catalog: registry.redhat.io/redhat/redhat-operator-index:v4.14
      packages:
        - name: nfd
          channels:
          - name: stable
            minVersion: 4.14.0-202503310939
            maxVersion: 4.14.0-202503310939
```

寫好此份檔案之後  
以這份檔案為基礎，將內容PUSH到quay上，repo後面是分辨的子目錄  
同時間會產生result檔案，裡面包含icsp檔案以及cs檔案(建議修改名稱以方便管理)  
```
oc-mirror --config ImageSetConfiguration.yaml docker://quay.resin.lab:8443/operator/nfd/20250421  --dest-skip-tls
```

