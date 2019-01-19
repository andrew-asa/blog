# wifi相关

## 零散笔记
+ OSX本身有个命令行工具：airport，利用这个工具可以方便的实现airmon-ng中的功能
  + 软链接: 
  `sudo ln -s /System/Library/PrivateFrameworks/Apple80211.framework/Versions/Current/Resources/airport /usr/local/bin/airport`
  + airport 
     + 扫描 `airprt en0 scan` 或 `airport -s`
     SSID 是 wifi名称，RSSI 是信号强度，CHANNEL 是信道。
     + airport en0 sniff 6  


+ 破解步骤
   + 抓包。主要是抓握手包
   + 密码破解
+ aircrack-ng
    ` aircrack-ng -w 01.txt 01.cap`

     
## Q & A
+ WPA和WPA2的区别
     

  
  
  
##  参考资料
+ [https://blog.csdn.net/jpiverson/article/details/22663101](https://blog.csdn.net/jpiverson/article/details/22663101)
+ []()
+ []()
+ []()
+ []()
+ []()
+ []()
+ []()
+ []()
+ []()
+ []()