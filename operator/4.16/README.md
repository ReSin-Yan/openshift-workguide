# OCP install operator 

### 前置準備  

使用Red Hat OpenShift Container Platform Operator Update Information Checker來查詢operator當前支援的版本(可以從當前OCP版本對應的operator)  
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
其中的版本是升級的版本  
代表意思是  
我去抓ocp4.16的operator(redhat-operator-index)  
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

### oc-mirror mirror to mirro    

```
2025/04/27 23:01:34  [INFO]   : === Results ===
2025/04/27 23:01:34  [INFO]   :  ✓  575 / 575 release images mirrored successfully
2025/04/27 23:01:34  [INFO]   : 📄 Generating IDMS file...
2025/04/27 23:01:34  [INFO]   : /s3/mirror/working-dir/cluster-resources/idms-oc-mirror.yaml file created
2025/04/27 23:01:34  [INFO]   : 📄 Generating ITMS file...
2025/04/27 23:01:34  [INFO]   : /s3/mirror/working-dir/cluster-resources/itms-oc-mirror.yaml file created
2025/04/27 23:01:34  [INFO]   : 📄 No catalogs mirrored. Skipping CatalogSource file generation.
2025/04/27 23:01:34  [INFO]   : 📄 No catalogs mirrored. Skipping ClusterCatalog file generation.
2025/04/27 23:01:34  [INFO]   : 📄 Generating Signature Configmap...
2025/04/27 23:01:34  [INFO]   : /s3/mirror/working-dir/cluster-resources/signature-configmap.json file created
2025/04/27 23:01:34  [INFO]   : /s3/mirror/working-dir/cluster-resources/signature-configmap.yaml file created
2025/04/27 23:01:34  [INFO]   : mirror time     : 29m26.448950564s
2025/04/27 23:01:34  [INFO]   : 👋 Goodbye, thank you for using oc-mirror
```

正常PUSH完成之後會產生如下的檔案  
總共產生以下檔案  
```
idms-oc-mirror.yaml  itms-oc-mirror.yaml  signature-configmap.yaml signature-configmap.json  
```

把idms, itms, signature-configmap三個yaml複製出來並更改檔名和裡面的name   
在可以連線到OCP的環境中  
分別apply其中前三份檔案  
```
idms-oc-mirror.yaml  itms-oc-mirror.yaml  signature-configmap.yaml signature-configmap.json  
```

signature-configmap.yaml  
```
apiVersion: v1
binaryData:
  sha256-7039e7bd34fd024ff619df914528c8d85e62b847291f6684096fd4ca8dba2bbd-2: owGbwMvMwMEoOU9/4l9n2UDGtYwpSWLxRQW5xZnpukWphbqG5fHmvuF6SZl56T9szaqVkosySzKTE3OUrBSqlTJzE9NTwayU/OTs1CLd3MS8zLTU4hLdlMx0IAWUUirOSDQyNbMyNzC2TDVPSjE2SUsxMDJJSzMztExJszQ0MTWySLZIsTBNNTNKsjAxN7I0TDMzszAxsDRLSzFJTrRISUo0SkpKUarVUVAqqSwAWaeUWJKfm5mskJyfV5KYmZdapAB0bV5iSWlRqhJQVWZKal5JZkklssOKUtNSi1LzksHaC0sTK/Uy8/XzC1LzijMy00qA0jmpicWpuimpZfr5yQUwvpWJnqGpnomlboWFWbyZiVItyBH5BSWZ+XnQEEguSgU6pghkalBqioJHYomCP9DUYJCpCsFAVwHDTcGxtCQjHxhulQoGegZ6hkBjOplkWBgYORjYWJlAwcrAxSkAiwKvaP4/3HqJh3UYyn9lnVnG9Lrnp7Pzhtkh3t2nz54qft6wVS/35TyutokOFgu036qtrUyQvKRfaLlYdMFNtugzR/tEMyMrrdlfWfHc/1wW4dl5o+Nxtvu7s2HnGBpu3XNYk3QvdAcfR0XlBO/M2I9TOt5UrXRRPZAupxh9okt887En0Wp2ycJiQoveM9e47OXZe/iGSxNfmPK0ODn1iTITkupmlhR+fn1rnlF8j96n90umbZNcf3nGfEXbX+JFXPmylcwy97iN67Ifc/PpbpR4p1odce/LmdNmv3hkQ0qKFu9gqDvA2xUtuTaSyZczoWP7Ho1n/pqLTszoF53k8WzXksRAnjKx/+XhD//NWJxxTfkF75lUz/cy+7/byPVc3bC/L2htmMhRkZKI2MsnTD8br2fvZRfgXd7yJYqP+6DRu/dBC5//VJ+YExEwL0x86dlLS3Te8Gyr5My0ag/gDGt/1TZ3Tb/UndD7K/QUf2k+7svh4+gonRb0WqNlF9eR8KjDG9ftN8zqYuR5t7hu3uJ3VZlu93jqhZ1PxfZNF2zVVttSkeeZ3zDpXDGf3oaYvNA5K+ZsUPPOuibNJJcWflvwaqBGotrk3mkdbkd2XLNue1l43jf+c9+9PZEXQuZ9NT05933K3KXsFRtcNp9Zd2qSoeOnbRvOFK89t3vFhIXmXP8WOAWcftbrbxK1zMfv5nHmy/8ufFlaum1W3cXad7wA
  sha256-0065822bf39b11e3a3113eb2dee4bddf0a07fe0967892498771bd9728e145ad0-3: owGbwMvMwMEoOU9/4l9n2UDGtYwpSWLxRQW5xZnpukWphbqlzpYW5aV6SZl56QetJlQrJRdllmQmJ+YoWSlUK2XmJqanglkp+cnZqUW6uYl5mWmpxSW6KZnpQAoopVSckWhkamZlYGBmamFklJRmbJlkaJhqnGhsaGicmmSUkppqkpSSkmaQaGCelmpgaWZuYWlkYmlhbm6YlGJpbmSRamhimphioFSro6BUUlkAsk4psSQ/NzNZITk/ryQxMy+1SAHo2rzEktKiVCWgqsyU1LySzJJKZIcVpaalFqXmJYO1F5YmVupl5uvnF6TmFWdkppUApXNSE4tTdVNSy/TzkwtgfCsTPUMzPWNz3QoLs3gzE6VakCPyC0oy8/OgIZBclAp0TBHI1KDUFAWPxBIFf6CpwSBTFYKBrgKGm4JjaUlGPjDcKhUM9Az0DIHGdDLJsDAwcjCwsTKBgpWBi1MAFgXxrfz/XYp28Ir8m1rXVfkqeOnyTJGNAo0Nl1cu5tPZo/WC78uV2bGC6l4lOtunKE57od5uKlijqt1Xqb/CRTun7dTW7peyXz/vbahy/blu8ufOFC+hCv2WjvULOHt2zNjN8J8nY9+OXwoM60xl58WW7Klctra53c312n2BRvtIx5W1ghzfVv1yva66S4lHXyd13eWIM6s9+z/ZTk47l/5qp33uqrRfsyZkCF2QUmg5psXzJ1rGUfen/9/V+ecf8uq7/b38MOZA7KG3J/vsdWP23j7VsLdqqe60Fyky8dOr7/btiY1pnTDx9IrXdrKCk3oKVB9uvK8Vwv+SWc1h1u+z08W+2T/b+8r0Ufz11JetlyZ8TWFbNXeHmUHZ0n0qFxTTnYJvplYHa2+54r73yRSFx29Ftsw7m7lmbkDZVg7hX8bKT6sMY+9oZqhKKRlqKc/1arq8lEOf2dFPaMnS8gM7Hj/uXcsrV+qSP0n7y6u0mqRL9apNVX+eeJTc+Kj7+XfYu/Lzp7IO+p2yEdhwOtLOdnf5esZzcvrTn8rF5H2IzzJc1LPPqvrP90oR5Tdy5cV9AqzfZCTbV0o5JqyOPvBNp375jgcz9k5wEmipepsakWbXzZ77a/flY8IbxXaeqxD0uWg2lXWfd3e5wI2emuvKMcqH+/9621zplrvL+Hib2Gx3aQ62u9VyARor9v07oGr3q0HdPesl25st95k17lwBAA==
  sha256-ea0429e14dc9ff007f56d5db2b75209c2510d81e6869194e33a53352d2b6a4fa-1: owGbwMvMwMEoOU9/4l9n2UDGtYwpSWLxRQW5xZnpukWphbrZVZEm6VF6SZl56dtdc6qVkosySzKTE3OUrBSqlTJzE9NTwayU/OTs1CLd3MS8zLTU4hLdlMx0IAWUUirOSDQyNbNKTTQwMbJMNTRJSbZMSzMwME8zNUsxTUkySjI3NTKwTDYyNTRIsTBMNbMwszS0NEk1Nk40NTY2NUoxSjJLNElLVKrVUVAqqSwAWaeUWJKfm5mskJyfV5KYmZdapAB0bV5iSWlRqhJQVWZKal5JZkklssOKUtNSi1LzksHaC0sTK/Uy8/XzC1LzijMy00qA0jmpicWpuimpZfr5yQUwvpWJnqGpnomZboWFWbyZiVItyBH5BSWZ+XnQEEguSgU6pghkalBqioJHYomCP9DUYJCpCsFAVwHDTcGxtCQjHxhulQoGegZ6hkBjOplkWBgYORjYWJlAwcrAxSkAi4IVTPz/UxQrL/+qfZmUtf7pTh413udfDDt1nG++PLz67xG+E3lznicKaqza2uN+Jbz+zb0pwRPy91100hM98HnGSWHdk16yv/YzR7FtXlLl4tOkwFMnNYlXu2Kxy/EtGWGea0qvNzOdcuGRvpl/tr247kaAdefU73+eHC0OVd0xR77r8q4fJ/P8ny0/t+JwUtvcDVN5nMJWGlXzJD0/lKbP9EDs10cdr/janQG8YT3C2u8717PcaerMP8XU+8twIyNv/KTNNcpv7J32cThv1jCPfal5L7U/vmF74rKGmkfdfuenttgvjlmbKnRh3svN3PbRU/Y9CU+9dJ6l4+aavNDCxB35k5O9t/R3nPJZ+Mc9KP2khbdhuFaHkpnarjtPHasvPKrRfnjb+sF+jv0n5bX0nqycnRnB5M18+eM9zsvh36+fleHN3RKaFvPlnk5b+tP9WdkPcwNX3jFbebQtV3lesbrUwg1W8mturbDafrZ5LdeuGapyX4r9z4vWxbDHfpyV98UwYNrOqrlTjcR8+i648qafEfh44MfKQ9unz9lx/0jU4sf/Cjmmx7hOuL/youKB5Fc716l27hEWXtZ/x117Rl9w8h2Dg5NkNt6uCDewmC1z5HV5gXdv6so3RTLXOX45zai4Ui/xua/ZW2PtgSu77frPr2m6YedwaLqhwrIl39jfLQkUa1b/pf4gIyxz4e9LL6S6anbXP7x90rD8zH2ZeQA=
kind: ConfigMap
metadata:
  labels:
    release.openshift.io/verification-signatures: ""
  name: ocpupgrade415416-mirrored-release-signatures
  namespace: openshift-config-managed
```

idms-oc-mirror.yaml  
```
---
apiVersion: config.openshift.io/v1
kind: ImageDigestMirrorSet
metadata:
  name: ocpupgrade415416-idms-release-0
spec:
  imageDigestMirrors:
  - mirrors:
    - quay.kyndryl.tw/olm2/openshift/release
    source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
  - mirrors:
    - quay.kyndryl.tw/olm2/openshift/release-images
    source: quay.io/openshift-release-dev/ocp-release
status: {}

```

itms-oc-mirror.yaml  
```
---
apiVersion: config.openshift.io/v1
kind: ImageTagMirrorSet
metadata:
  name: ocpupgrade415416-itms-release-0
spec:
  imageTagMirrors:
  - mirrors:
    - quay.kyndryl.tw/olm2/openshift/release-images
    source: quay.io/openshift-release-dev/ocp-release
status: {}
```
