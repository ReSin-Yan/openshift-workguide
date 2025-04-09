Last update: 2025/04/09
# openshift-workguide 

因緣際會下會需要協助ocp的專案
會以第一次接觸的使用者視角來觀看整個ocp的常見操作  
會以離線安裝的方式來進行  

## Quay、Mirror Registry     

Quay跟Mirror Registry是不同東西  
Mirror Registry 最高是對應 Quay 3.12  
Quay 3.13 應該是需要額外購買 License  
其實底層是一樣的東西，但是 Mirror Registry 的 License 是包含在 OpenShift 的 License 裡面  
架構差不多，但是安裝的檔案來源不同  
Mirror Registry的使用限制都寫在這邊了，基本上就是不能放 User 的檔案上去  
詳細參考:[Chapter 3. Mirroring in disconnected environments](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/disconnected_environments/mirroring-in-disconnected-environments#installing-mirroring-disconnected-about "link")  


### Operator   

ocp自己有的套件，將各種服務拆分成單一

