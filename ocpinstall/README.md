# OCP install  

ocpinstall 包含了幾個部分  

 | 配置腳色 | 用處 | 
|-------|-------|
| bastion| 堡壘機，通常都會通過此台機器進行最基礎的設定 |
| http |  ocp安裝過程中會需要用到的服務，ocp安裝類似於pxe boot，會從一個存放點下載設定檔案近來進行設定(此檔案為.ign檔案) |
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



### Install Mirror Registry  

