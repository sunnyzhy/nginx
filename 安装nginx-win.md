# 下载nginx
http://nginx.org/en/download.html

# 下载winsw
http://repo.jenkins-ci.org/releases/com/sun/winsw/winsw/

# 注册windows服务
1. 将 winsw 可执行程序复制到 nginx 的安装目录下，并重命名为 nginx-service

2. 新建nginx-service.xml（注：文件名必须与可执行文件名相同）
```
<service>      
 <id>nginx</id>      
  <name>nginx</name>      
  <description>nginx</description>      
  <executable>D:\nginx\nginx-1.13.6\nginx.exe</executable>      
  <logpath>D:\nginx\nginx-1.13.6</logpath>      
  <logmode>roll</logmode>      
  <depend></depend>      
  <startargument>-p D:\nginx\nginx-1.13.6</startargument>      
  <stopargument>-p D:\nginx\nginx-1.13.6 -s stop</stopargument>      
</service>  
```

3. 安装服务
```
D:\>cd D:\nginx\nginx-1.13.6

D:\nginx\nginx-1.13.6>nginx-service.exe install
```

注：
```
卸载  nginx-service.exe uninstall

停止  nginx-service.exe stop

启动  nginx-service.exe start
```

4. 浏览
在浏览器的地址栏中输入 http://localhost/ ，就会有nginx的欢迎界面 Welcome to nginx!

# windows 的 80 端口被 System 进程占用
1. IIS 默认使用 80 端口进行 HTTP 通信，关闭 IIS 服务

2. Apache 默认使用 80 端口，关闭 Apache 服务或修改 Apache 加默认的端口

3. SQL server 占用 80 端口，停止或禁用 **显示名称为"SQL Server Reporting Services (SQLEXPRESS)"（服务名称为 "ReportServer$SQLEXPRESS"）** 的服务
