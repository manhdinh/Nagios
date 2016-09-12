##I.	Mô hình triển khai dự kiến 
Mô hình triển khai Nagios Core High Avaibility
![nagios](/images/nagios01.png)
Trong mô hình này, hệ thống sẽ gồm 2 máy Nagios Core server, 1 máy đóng vai trò Primary, 1 máy đóng vai trò Secondary làm backup. Mô hình sử dụng 
kỹ thuật Rsync, cung cấp cơ chế làm việc giữa máy Primary và Secondary. Khi hệ thống hoạt động bình thường, máy Primary sẽ tiếp nhận toàn nhận công
 việc của 1 máy Nagios Core server. Tuy nhiên, khi máy Primary gặp sự cố, thông qua cơ chế Rsync, máy Sendondary trong thời gian ngắn nhất sẽ lên 
 thay thế vai trò của máy Primary. Khi sự cố được khắc phục, máy Primary sẽ đảm nhận lại vai tro cũ. Mô hình này đảm bảo việc khi có sự cố xảy ra,
 việc monitor hệ thống sẽ không bị gián đoạn quá lâu. 
##II. Triển khai Nagios Core
###1 Cấu hình khuyến cáo
List server 

|STT|Tên|IP|OS|
|----------------|---|--|--|
|1|Nagios-server|172.16.69.221|Centos 7|
|2|Nagios-backup|172.16.69.223|Centos 7|

Cấu hình phần cứng yêu cầu :

|Số lượng host giám sát|Số lượng service giám sát|RAM|CPU|Disk|
|----------------------|-------------------------|---|---|----|
|50|250|1-4GB|1-2 Core|40GB|
|100|500|4-8GB|2-4 Core|80GB|
|>500|2500|>8GB|>4 Core|120GB|

####2. Cài đặt Nagios

**Tại máy Nagios Primary**
**Bước 1** : Tắt Selinux
```sh
setenforce 0
```
Sửa file `/etc/selinux/config` và chuyển `enforcing` thành `disabled`
**Bước 2** : Cài đặt các gói điều kiện
Cài các gói cần thiết :
```sh
yum install httpd php php-cli gcc glibc glibc-common gd gd-devel net-snmp openssl-devel wget unzip -y
```
Tạo user, group cho Nagios và phân quyền

```sh
useradd nagios
groupadd nagcmd
usermod -a -G nagcmd nagios
usermod -a -G nagcmd apache
```
**Bước 3** : Dowload và cài đặt Nagios

Dowload và giải nén các gói Nagios

```sh
cd /tmp
wget https://assets.nagios.com/downloads/nagioscore/releases/nagios-4.1.1.tar.gz
wget http://www.nagios-plugins.org/download/nagios-plugins-2.1.1.tar.gz
tar zxf nagios-4.1.1.tar.gz
tar zxf nagios-plugins-2.1.1.tar.gz
cd nagios-4.1.1
```
Compile các gói Nagios vừa được giải nén 

```sh
./configure --with-command-group=nagcmd
make all
make install
make install-init
make install-config
make install-commandmode
make install-webconf
```
Tạo password cho nagiosadmin

```sh
htpasswd -c /usr/local/nagios/etc/htpasswd.users nagiosadmin
```
Cài đặt Nagios Server

```sh
cd /tmp/nagios-plugins-2.1.1
./configure --with-nagios-user=nagios --with-nagios-group=nagios --with-openssl
make all
make install
```
Tạo rule firewall cho port http 

```sh
firewall-cmd --zone=public --add-port=80/tcp --permanent
firewall-cmd --reload
```
Khởi động Nagios Server
```
service httpd start
service nagios start
```
Đăng nhập vào giao diện web interface của Nagios : http://ip_nagioserver/nagios/, nhập username : nagiosadmin và password

![nagios](/images/nagios02.png)

###3. Hướng dẫn cài đặt Nagios client sử dụng NRPE

####Trên Centos Client : 

**Bước 1** : Cài đăt gói EPEL :

Centos 6.x : 

```sh
wget http://epel.mirror.net.in/epel/6/i386/epel-release-6-8.noarch.rpm
rpm -Uvh epel-release-6-8.noarch.rpm
```
Centos 7 

```sh
yum install epel-release -y
```
**Bước 2** : Cài đặt **nrpe** và **nagios plugin**

```sh
yum install nrpe nagios-plugins-all openssl
```
Với Ubuntu/Debian Client :

```sh
apt-get install nagios-nrpe-server nagios-plugins -y
```
**Bước 3** : Chỉnh sửa file cấu hình NRPE và khởi động 
```sh
vi /etc/nagios/nrpe.cfg
```
Tìm và chỉnh sửa như sau :

```sh
allowed_hosts=127.0.0.1, 10.0.0.20
```
*Chú ý :  Thay IP 10.0.0.20 với Nagios server IP của bạn 

Khởi động Nagios NRPE

Centos 6.x
```sh
service nrpe start
chkconfig nrpe on
```
Centos 7

```sh
systemctl start nrpe
chkconfig nrpe on
```
Debian/Ubuntu

```sh
/etc/init.d/nagios-nrpe-server restart
```

####Trên máy Nagios Server

Chỉnh file cấu hình của Nagios, đặt tất cả các file cấu hình của host client là server vào cùng 1 thư mục :

```sh
vi /usr/local/nagios/etc/nagios.cfg
```
Tìm và bỏ dấu "#" ở trước dòng sau : 

```sh
cfg_dir=/usr/local/nagios/etc/servers
```
Tạo thư mục chứa các file cấu hình và tạo file cấu hình cho client :

```sh
mkdir /usr/local/nagios/etc/servers
vi /usr/local/nagios/etc/servers/clients.cfg
```
Định nghĩa host client với các thông tin sau

```sh
define host{
use                             linux-server
host_name                       client			#hostname của host client
alias                           client			#bí danh của host client
address                         192.168.1.152	#IP của host client
max_check_attempts              5
check_period                    24x7
notification_interval           30
notification_period             24x7
}
```
Restart service và vào giao diện Web kiểm tra
```sh
systemctl restart nagios
```