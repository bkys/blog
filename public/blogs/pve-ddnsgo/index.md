ddnsgo最简单，后台修改配置，开机运行

创建目录

mkdir -p /opt/ddns-go

进入目录

cd /opt/ddns-go/

下载附件上传或者直接下载

附件链接

https://github.com/jeessy2/ddns-go/releases/download/v5.6.6/ddns-go_5.6.6_linux_x86_64.tar.gz

上传ddns-go_5.6.6_linux_x86_64.tar.gz到/opt/ddns-go/

也可以wget -c https://github.com/jeessy2/ddns-go/releases/download/v5.6.6/ddns-go_5.6.6_linux_x86_64.tar.gz直接下载

 

解压

tar -zxvf ./ddns-go_5.6.6_linux_x86_64.tar.gz

安装

./ddns-go -s install

使用

浏览器访问pve管理ip:9876