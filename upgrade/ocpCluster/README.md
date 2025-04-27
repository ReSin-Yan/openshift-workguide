# Upgrade OCP Cluster

### 前置準備(文件查詢與準備)  
前置準備包括，準備images(上傳至任意一個repo皆可)、operatror版本確認以及升級(看你裝了哪幾個)  
OCP的升級版本如果跨越版本幅度較大，就會需要跨版本升級兩次以上  
通常先升級Master之後在將Worker進行升級  

確認當前channel  
```
oc describe clustervison  
```
使用Red Hat OpenShift Container Platform Update Graph來查詢版本升級流程(選擇對應的版本)   
[查詢update path](https://access.redhat.com/labs/ocpupgradegraph/update_path/ "link")  

使用Red Hat OpenShift Container Platform Operator Update Information Checker來查詢已安裝的operator的版本是否有支援升級後的版本，若否，ocp升級完後需再升級各operator  
[查詢版本支援](https://access.redhat.com/labs/ocpouic/   "link")  


並在OpenShift Operator Life Cycles確認版本所支援的時間(EOS end of support)  
[查詢EOS](https://access.redhat.com/support/policy/updates/openshift_operators "link")  


### 安裝oc-mirror插件  
安裝oc-mirror插件在可以對外的環境上  
有兩種方式  
下載下來之後弄成TAR黨，再攜帶到可以連線到OCP quay的位置  
如果可以這一台可以對外，並且可以直接連線到OCP，可以直接從此台機器推上去，不需要額外做成TAR檔案  

[下載插件](https://console.redhat.com/openshift/downloads "link")   
版本看起來都只有4.18的oc-mirror，但是並不影響功能  
在4.16之後有出了oc-mirror --v2，介面更完成熟，也建議全部轉換至v2  

載下來後解壓縮和搬移指令到/usr/local/bin  
```
tar -xvzf oc-mirror.tar.gz
chmod +x oc-mirror
mv oc-mirror /usr/local/bin/
```

確認oc-mirror是否正常  
```
oc-mirror version
```

對外下載的虛擬機需要確認可以podlogin  registry.redhat.io  
```
podman login registry.redhat.io
```

對內的虛擬機需要確認可以podlogin  local quay  
```
podman login local.quay
```

### oc-mirror v2    

建立以下yaml  
其中的版本是升級的版本(Update Graph)  
例如當下版本是4.14.22，目標升級到4.16.73  
就會需要先升級到4.15.46再到4.16.37  

```
apiVersion: mirror.openshift.io/v2alpha1
kind: ImageSetConfiguration
mirror:
  platform:
    channels:
      - name: stable-4.15
        minVersion: 4.15.46
        maxVersion: 4.15.46
        type: ocp
      - name: stable-4.16
        minVersion: 4.16.37
        maxVersion: 4.16.37
        type: ocp
```

參考oc-mirror v2 說明  
總共有三種模式  
```
- mirrorToDisk - pulls the container images from the source specified in the image set configuration and packs them into a tar archive on disk (local directory).
- diskToMirror - copy the containers images from the tar archive to a container registry (--from flag is required on this workflow).
- mirrorToMirror - copy the container images from the source specified in the image set configuration to the destination (container registry).

Examples:
  # Mirror To Disk
  oc-mirror -c ./isc.yaml file:///home/<user>/oc-mirror/mirror1 --v2

  # Disk To Mirror
  oc-mirror -c ./isc.yaml --from file:///home/<user>/oc-mirror/mirror1 docker://localhost:6000 --v2

  # Mirror To Mirror
  oc-mirror -c ./isc.yaml --workspace file:///home/<user>/oc-mirror/mirror1 docker://localhost:6000 --v2

  # Delete Phase 1 (--generate)
  oc-mirror delete -c ./delete-isc.yaml --generate --workspace file:///home/<user>/oc-mirror/delete1 --delete-id delete1-test docker://localhost:6000 --v2

  # Delete Phase 2
  oc-mirror delete --delete-yaml-file /home/<user>/oc-mirror/delete1/working-dir/delete/delete-images-delete1-test.yaml docker://localhost:6000 --v2
```
通常前兩項會綁再一起  
後一項單獨  

### oc-mirror mirror to disk and disk to mirro     

第一種方式  
從外部下載之後，將檔案上傳到離線環境(通常是具備oc-mirro以及可以連線到quay之環境)  
mirror to disk
```
oc-mirror --v2 --config=/root/WS_Resin/ocpupgrade/isc.yaml file:///s3/mirror --retry-times=10 --image-timeout=120m0s --retry-delay=10s
```

disk to mirro  
```
oc-mirror --v2 --config=/root/WS_Resin/ocpupgrade/isc.yaml --from file:///s3/mirror --retry-times=10 --image-timeout=120m0s --retry-delay=10s docker://quay.kyndryl.tw/olm2
```

儲存目錄，由於此種方式會壓縮一個tar檔案，通常不會太小  
所以另外指定一個具備足夠儲存空間的路徑  
```
前面必須寫這樣
file://  

正確寫法如下
file:///root/storage/  
```
其餘路徑檔案也需要確認好範圍  

### oc-mirror mirror to mirro    

如果當前環境是可以直些連線到外部網站下載  
並且可以直接輸入以下內容  

```
oc-mirror --v2 --config=/root/WS_Resin/ocpupgrade/isc.yaml --workspace file:///s3/mirror --retry-times=10 --image-timeout=120m0s --retry-delay=10s docker://quay.kyndryl.tw/olm2
```

### TIPS  

關於推送路徑olm2  
需要事先在quay上面新增好org  
但是哪一個org是可以進行討論  
主要取決於管理難度  
olm2單純只是名子而已  
所以要思考怎樣管理是最不會混亂的  
```
docker://quay.kyndryl.tw/olm2
```
