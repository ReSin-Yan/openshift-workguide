# OCP install operator 

### 前置準備  

使用Red Hat OpenShift Container Platform Operator Update Information Checker來查詢operator當前支援的版本(可以從當前OCP版本對應的operator)  
用這種方式可以跳過去查詢的時間  
[查詢版本支援](https://access.redhat.com/labs/ocpouic/   "link")  

### 安裝oc-mirror插件  
安裝oc-mirror插件在可以對外的環境上  
有兩種方式  
下載下來之後弄成TAR黨，再攜帶到可以連線到OCP quay的位置  
如果可以這一台可以對外，並且可以直接連線到OCP，可以直接從此台機器推上去，不需要額外做成TAR檔案  

[下載插件](https://console.redhat.com/openshift/downloads "link")   
版本目前只有4.18的oc-mirror，但是並不影響功能  
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
去找ocp4.16的operator(redhat-operator-index)  
packages名稱是tempo-product  
安裝的版本是0.14.1-1跟0.14.1-2
版本建議一個版本就好(但是遇過一個版本無法下載的問題，所以可以多試幾次)  

```
apiVersion: mirror.openshift.io/v2alpha1
kind: ImageSetConfiguration
mirror:
  operators:
  - catalog: registry.redhat.io/redhat/redhat-operator-index:v4.16
    packages:
    - name: tempo-product
      channels:
      - name: stable
        minVersion: 0.14.1-1
        maxVersion: 0.14.1-2
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
需要更換的是config、file的路徑  
config對應yaml檔案的位置  
會把檔案下載到file的路徑當中  

```
oc-mirror --v2 --config=/root/resin_workspace/pullimages/operator/ImageSetConfiguration2.yaml  file:///root/resin_workspace/pullimages/operator/copymirror --retry-times=10 --image-timeout=120m0s --retry-delay=10s
```

過程如下  
```
2025/05/01 10:53:54  [INFO]   : 👋 Hello, welcome to oc-mirror
2025/05/01 10:53:54  [INFO]   : ⚙️  setting up the environment for you...
2025/05/01 10:53:54  [INFO]   : 🔀 workflow mode: mirrorToDisk
2025/05/01 10:53:54  [INFO]   : 🕵  going to discover the necessary images...
2025/05/01 10:53:54  [INFO]   : 🔍 collecting release images...
2025/05/01 10:53:54  [INFO]   : 🔍 collecting operator images...
 ✓   (1m51s) Collecting catalog registry.redhat.io/redhat/redhat-operator-index:v4.16
2025/05/01 10:55:46  [INFO]   : 🔍 collecting additional images...
2025/05/01 10:55:46  [INFO]   : 🔍 collecting helm images...
2025/05/01 10:55:46  [INFO]   : 🔂 rebuilding catalogs
 ✓   () Rebuilding catalog docker://registry.redhat.io/redhat/redhat-operator-index:v4.16
2025/05/01 10:55:46  [INFO]   : 🚀 Start copying the images...
2025/05/01 10:55:46  [INFO]   : 📌 images to copy 12
 ✓   (22s) ose-kube-rbac-proxy@sha256:7efeeb8b29872a6f0271f651d7ae02c91daea16d853c50e374c310f044d8c76c ➡️  cache
 ✓   (31s) jaeger-es-index-cleaner-rhel8@sha256:d0ad37a50a8b5f2e816e03f9894ad96d9914aa31fb24d684f98bd557cc406718 ➡️  cache
 ✓   (31s) jaeger-es-rollover-rhel8@sha256:236798724a8bbf974bd24cc698af982ad96ef43e92e9c7751727ee3ae4d68823 ➡️  cache
 ✓   (5s) jaeger-operator-bundle@sha256:f8ed2eb7191cb6199fbe36a02727865b1c418aa0d90ce01d6b76e3b9c7768f33 ➡️  cache
 ✓   (38s) jaeger-agent-rhel8@sha256:11012d44dfefd66f82b551e06757464898c9d16e7a85c34fd0e0ffca00ac5421 ➡️  cache
 ✓   (43s) jaeger-all-in-one-rhel8@sha256:2beb3661869af4971f8e789464ebf06372dc1cc8aef42eff6574d0602bbf0ad5 ➡️  cache
 ✓   (10s) redhat-operator-index:v4.16 ➡️  cache
 ✓   (48s) jaeger-ingester-rhel8@sha256:7a4b5d397712fd050eba79abcb1cce0a90231e3f09014b425362d524b89b1dc1 ➡️  cache
 ✓   (48s) ose-oauth-proxy@sha256:234af927030921ab8f7333f61f967b4b4dee37a1b3cf85689e9e63240dd62800 ➡️  cache
 ✓   (57s) jaeger-collector-rhel8@sha256:79948c384908d72f87a9bd018f3a230a2bc38ff32cac1c17ce9bf2e62f7a92dc ➡️  cache
 ✓   (51s) jaeger-query-rhel8@sha256:f6e489b27ffc438645c3c8e203f3c98733dfed0ccacbd9e9b69079f0b5b8b693 ➡️  cache
12 / 12 (1m27s) [=====================================================================================================================================================================================================================] 100 %
 ✓   (56s) jaeger-rhel8-operator@sha256:259f3fcb7c05183f4879a512f502c0346a0401aee5775ac9de6f424729fd6e83 ➡️  cache
2025/05/01 10:57:14  [INFO]   : === Results ===
2025/05/01 10:57:14  [INFO]   :  ✓  12 / 12 operator images mirrored successfully
2025/05/01 10:57:14  [INFO]   : 📦 Preparing the tarball archive...
2025/05/01 10:59:17  [INFO]   : mirror time     : 5m22.906209797s
2025/05/01 10:59:17  [INFO]   : 👋 Goodbye, thank you for using oc-mirror
```

之後在資料夾內可以找到檔案mirror_000001.tar  
將這個檔案放到離線環境中  
disk to mirro  
```
oc-mirror --v2 --config=/root/resin_workspace/pullimages/operator/ImageSetConfiguration2.yaml --from file:///root/resin_workspace/pullimages/operator/copymirror --retry-times=10 --image-timeout=120m0s --retry-delay=10s docker://quay.resin.lab:8443/olm2/jaeger-product
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
參考以下指令  
需要更換的是config、workspace的路徑以及repo的路徑  
config對應yaml檔案的位置
workspace會存放working-dir/cluster-resources這一個檔案路徑，裡面放著三個檔案   
repo路徑後面需要進行更換，不能相同，會蓋過前一個步驟的  

```
oc-mirror --v2 --config=/root/resin_workspace/pullimages/operator/ImageSetConfiguration2.yaml --workspace file:///root/resin_workspace/pullimages/operator/mirror --retry-times=10 --image-timeout=120m0s --retry-delay=10s docker://quay.resin.lab:8443/olm2/tempo-product  
```

working-dir/cluster-resources檔案  
```
cc-redhat-operator-index-v4-16.yaml  cs-redhat-operator-index-v4-16.yaml  idms-oc-mirror.yaml
```

輸入指令之後如下  
```
2025/05/01 09:35:44  [INFO]   : 👋 Hello, welcome to oc-mirror
2025/05/01 09:35:44  [INFO]   : ⚙️  setting up the environment for you...
2025/05/01 09:35:44  [INFO]   : 🔀 workflow mode: mirrorToMirror
2025/05/01 09:35:44  [INFO]   : 🕵  going to discover the necessary images...
2025/05/01 09:35:44  [INFO]   : 🔍 collecting release images...
2025/05/01 09:35:44  [INFO]   : 🔍 collecting operator images...
 ✓   (1m21s) Collecting catalog registry.redhat.io/redhat/redhat-operator-index:v4.16
2025/05/01 09:37:06  [INFO]   : 🔍 collecting additional images...
2025/05/01 09:37:06  [INFO]   : 🔍 collecting helm images...
2025/05/01 09:37:06  [INFO]   : 🔂 rebuilding catalogs
 ✓   (0s) Rebuilding catalog docker://registry.redhat.io/redhat/redhat-operator-index:v4.16
2025/05/01 09:37:06  [INFO]   : 🚀 Start copying the images...
2025/05/01 09:37:06  [INFO]   : 📌 images to copy 18
 ✓   (1m31s) ose-kube-rbac-proxy@sha256:7efeeb8b29872a6f0271f651d7ae02c91daea16d853c50e374c310f044d8c76c ➡️  quay.resin.lab:8443/olm2/tempo-product/openshift4/
 ✓   (1m48s) tempo-query-rhel8@sha256:b43c3af00d557a549a1ab7737583e80c16896dcec1d379087f517ee080ecde74 ➡️  quay.resin.lab:8443/olm2/tempo-product/rhosdt/
 ✓   (2m16s) tempo-gateway-opa-rhel8@sha256:6f91ab07ee9b0361fd4f26d0d380c09a43a1839099a97103e6c130b6cf926be8 ➡️  quay.resin.lab:8443/olm2/tempo-product/rhosdt/
 ✓   (2m24s) tempo-rhel8-operator@sha256:81bf303fe624a69857e4b4a0e3e14494b818824f04d2c4ccdfdc0a02743ebaf2 ➡️  quay.resin.lab:8443/olm2/tempo-product/rhosdt/
 ✓   (2m26s) tempo-gateway-rhel8@sha256:d129101bf8563715cf8f2776a8359316d8dde35899af7f736fe9cbb380f4530c ➡️  quay.resin.lab:8443/olm2/tempo-product/rhosdt/
 ✓   (2m32s) tempo-jaeger-query-rhel8@sha256:ebef9709e328cf9918ff99ed3dcac6abb1a38c2cd3b46af4fd61cc0b87c0e165 ➡️  quay.resin.lab:8443/olm2/tempo-product/rhosdt/
 ✓   (50s) tempo-query-rhel8@sha256:9c910e8ba1433e6bffb74f0211dd81f8647184351621ea7ba001382c6ea3e08f ➡️  quay.resin.lab:8443/olm2/tempo-product/rhosdt/
 ✓   (1m17s) tempo-rhel8@sha256:f7a68277533ff937ca012b6114443416cd11b853783795116e40cedec21fd8e4 ➡️  quay.resin.lab:8443/olm2/tempo-product/rhosdt/
 ✓   (2m49s) tempo-rhel8-operator@sha256:89110559c33c815b59aa914b5dbb170242dbbdbf9cf1e9f75f63875fb6d9e895 ➡️  quay.resin.lab:8443/olm2/tempo-product/rhosdt/
 ✓   (17s) tempo-operator-bundle@sha256:fd49fd51bd9c033317ca2ea172e6a21c84ccc17b609f9e5543ece39dd5ec8808 ➡️  quay.resin.lab:8443/olm2/tempo-product/rhosdt/
 ✓   (2m57s) tempo-rhel8@sha256:7ad3f3e5f32457a2f2bf79e0ceffb9f988183a8fb4654d39f5d6496ca0ae9b70 ➡️  quay.resin.lab:8443/olm2/tempo-product/rhosdt/
 ✓   (10s) tempo-operator-bundle@sha256:a980e21c5cf96387bee07f2f271e73060bb5032ac969d678dc1f718841531ecb ➡️  quay.resin.lab:8443/olm2/tempo-product/rhosdt/
 ✓   (4s) redhat-operator-index:v4.16 ➡️  cache
 ✓   (11s) redhat-operator-index:v4.16 ➡️  quay.resin.lab:8443/olm2/tempo-product/redhat/
 ✓   (57s) tempo-gateway-opa-rhel8@sha256:747128f0fa372e44872674b5bd54f3479e8bedc839c0914fe1a038c36a8ecdd7 ➡️  quay.resin.lab:8443/olm2/tempo-product/rhosdt/
 ✓   (55s) tempo-jaeger-query-rhel8@sha256:ee550055792ded0c3d2783664166351120db35ba754b0ecb58a158d82ab5bb80 ➡️  quay.resin.lab:8443/olm2/tempo-product/rhosdt/
 ✓   (1m6s) tempo-gateway-rhel8@sha256:39189db648e2ac617b94424a6b4f556f645ce26c7f235c36bcf2df74e226e72b ➡️  quay.resin.lab:8443/olm2/tempo-product/rhosdt/
18 / 18 (4m18s) [=====================================================================================================================================================================================================================] 100 %
 ✓   (2m0s) ose-oauth-proxy@sha256:234af927030921ab8f7333f61f967b4b4dee37a1b3cf85689e9e63240dd62800 ➡️  quay.resin.lab:8443/olm2/tempo-product/openshift4/
2025/05/01 09:41:24  [INFO]   : === Results ===
2025/05/01 09:41:24  [INFO]   :  ✓  18 / 18 operator images mirrored successfully
2025/05/01 09:41:24  [INFO]   : 📄 Generating IDMS file...
2025/05/01 09:41:24  [INFO]   : /root/resin_workspace/pullimages/operator/mirror/working-dir/cluster-resources/idms-oc-mirror.yaml file created
2025/05/01 09:41:24  [INFO]   : 📄 No images by tag were mirrored. Skipping ITMS generation.
2025/05/01 09:41:24  [INFO]   : 📄 Generating CatalogSource file...
2025/05/01 09:41:24  [INFO]   : /root/resin_workspace/pullimages/operator/mirror/working-dir/cluster-resources/cs-redhat-operator-index-v4-16.yaml file created
2025/05/01 09:41:24  [INFO]   : 📄 Generating ClusterCatalog file...
2025/05/01 09:41:24  [INFO]   : /root/resin_workspace/pullimages/operator/mirror/working-dir/cluster-resources/cc-redhat-operator-index-v4-16.yaml file created
2025/05/01 09:41:24  [INFO]   : mirror time     : 5m39.746468817s
2025/05/01 09:41:24  [INFO]   : 👋 Goodbye, thank you for using oc-mirror
```


 
在可以連線到OCP的環境中  
分別apply cs-redhat-operator-index-v4-16.yaml以及idms-oc-mirror.yaml  
務必要更改裡面的name(建立出來的name名稱都會一樣)
```
cc-redhat-operator-index-v4-16.yaml  cs-redhat-operator-index-v4-16.yaml  idms-oc-mirror.yaml
```

cs-redhat-operator-index-v4-16.yaml  
下面是修改過版本  
```
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: cs-tempo-product-operator-index-v4-16
  namespace: openshift-marketplace
spec:
  image: quay.resin.lab:8443/olm2/tempo-product/redhat/redhat-operator-index:v4.16
  sourceType: grpc
status: {}
```

cs-redhat-operator-index-v4-16.yaml  
下面是修改過版本  

### Debug   

可以通過指令檢查是否有成功下載operator  

```
oc get pod -n openshift-marketplace
```

觀察到其中的容器正在建立
```
cs-opentelemetry-product-operator-index-xdpc7   1/1     Running             0          66m
cs-quay-operator-index-v4-16-ckpzq              1/1     Running             0          66m
cs-servicemesh-operator-index-dg5pr             1/1     Running             0          66m
cs-tempo-product-operator-index-v4-16-rclrt     0/1     ContainerCreating   0          63s
marketplace-operator-c5d475c88-rc7fn            1/1     Running             0          63m
```

確定建立完畢之後  
及代表operatorhub有多出東西  
可以從UI方面查詢是否有成功指向想要的operator  

### TIPS    

務必要確認oc-mirror指令中，repo對應的路徑，要每一筆都不相同，不然會蓋掉  
oc-mirror會產生檔案，檔案預設的名稱也會相同，所以也要記得修改掉  


