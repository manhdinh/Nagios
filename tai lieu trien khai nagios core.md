##Mục lục

*	[I.	Mô hình Logical](#mh)
*	[II. Triển khai Nagios ] (tk)
	*	[1 Chuẩn bị] (#cbi)
		*	[1.1 Mô hình mạng]	(#mhm)	
		*	[1.2 Danh sách server]	(#ds)	
		*	[1.3 Cấu hình phần cứng yêu cầu]	(#chpc)		
	*	[2 Cài đặt Nagios] (#cdnagios)
			*	[2.1 Cài đặt phía Server]	(#cdsrv)	
				*	[2.1.1 Cài đặt Nagios Primary]	(#cdpri)	
				*	[2.1.2 Cài đặt Nagios Secondary] (#cdse)					
			*	[2.2 Cài đặt Rsync]	(#cdrsync)	
				*	[2.2.1 Trên Nagios Server] (#trensrv)	
				*	[2.2.2 Trên Nagios Backup] (#trenbka)					
			*	[2.3 Cài đặt Nagios Client]	(#cdclient)			
				*	[2.3.1 Trên Centos Client] (#trence)	
					*	[2.3.1.1 Trên Centos 6.x] (#cent6)	
					*	[2.3.1.2 TrênCentos 7 ] (#cent7)					
				*	[2.3.2 Trên Ubuntu/Debian Client ] (#ubuntu)	
	*	[3 Cấu hình cho host client] (#chclient)
		*	[3.1 Tạo cấu hình cho host client]	(#taoch)	
		*	[3.2 Cấu hình cho các thông số phần cứng]	(#chtspc)	
		*	[3.3 Cấu hình cho service SSH ]	(#chssh)			
	*	[4 Cấu hình gửi mail cảnh báo cho Nagios] (#chmail)

#I.	Mô hình Logical
<a name="mh"> </a> 

Mô hình triển khai Nagios Core High Avaibility
![nagios](/images/nagios01.png)

Trong mô hình này, hệ thống sẽ gồm 2 máy Nagios Core server, 1 máy đóng vai trò Primary, 1 máy đóng vai trò Secondary làm backup. Mô hình sử dụng 
kỹ thuật Rsync, cung cấp cơ chế làm việc giữa máy Primary và Secondary. Khi hệ thống hoạt động bình thường, máy Primary sẽ tiếp nhận toàn nhận công
 việc của 1 máy Nagios Core server. Tuy nhiên, khi máy Primary gặp sự cố, thông qua cơ chế Rsync, máy Sendondary trong thời gian ngắn nhất sẽ lên 
 thay thế vai trò của máy Primary. Khi sự cố được khắc phục, máy Primary sẽ đảm nhận lại vai tro cũ. Mô hình này đảm bảo việc khi có sự cố xảy ra,
 việc monitor hệ thống sẽ không bị gián đoạn quá lâu. 
 
#II. Triển khai Nagios 
<a name="tk"> </a> 

##1 Chuẩn bị
<a name="cbi"> </a> 

###1.1 Mô hình mạng
<a name="mhm"> </a> 

![nagios](/images/nagios06.png)

###1.2 Danh sách server
<a name="ds"> </a> 

|STT|Tên|IP|OS|
|-------|---------------|--------------|---------|
|1|Nagios-server|172.16.69.221|Centos 7|
|2|Nagios-backup|172.16.69.223|Centos 7|

###1.3 Cấu hình phần cứng yêu cầu
<a name="chpc"> </a> 

|Số lượng host giám sát|Số lượng service giám sát|RAM|CPU|Disk|
|----------------------|-------------------------|---|---|----|
|50|250|1-4GB|1-2 Core|40GB|
|100|500|4-8GB|2-4 Core|80GB|
|>500|2500|>8GB|>4 Core|120GB|

##2 Cài đặt Nagios
<a name="cdnagios"> </a> 

###2.1 Cài đặt phía Server
<a name="cdsrv"> </a> 

####2.1.1 Cài đặt Nagios Primary
<a name="cdpri"> </a> 

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
Cài đặt Nagios Core

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
Cài đặt Nagios Plugin

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

####2.1.2 Cài đặt Nagios Secondary
<a name="cdse"> </a> 

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
Cài đặt Nagios Core

```sh
./configure --with-command-group=nagcmd
make all
make install
make install-init
make install-config
make install-commandmode
make install-webconf
```

Cài đặt Nagios Plugin

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
Khởi động http 

```sh
service httpd start
```

###2.2 Cài đặt Rsync
<a name="cdrsync"> </a> 

####2.2.1 Trên Nagios Server
<a name="trensrv"> </a> 

```sh
yum install rsync -y
```
####2.2.2 Trên Nagios Backup
<a name="trenbka"> </a> 

```sh
yum install rsync -y
```
Tạo script backup cho Nagios Server
```sh
vi /opt/back_up_nagios.sh
```
Copy các dòng sau vào file 

```sh
!/bin/bash
{
echo -e "\nDIR: /usr/local/nagios"
/usr/bin/rsync -Hxva --delete --progress --exclude="perl" --exclude="share" --exclude="var" 172.16.69.221:/usr/local/nagios/ /usr/local/nagios/
}
```
Lưu file và thoát

Phân quyền cho file script
```sh
chmod +x /opt/back_up_nagios.sh
```
Tạo crontab
```sh
crontab -e
	0 1 * * * sh /opt/back_up_nagios.sh >/dev/null
```
Lưu và thoát

Backup thử dữ liệu từ Nagios Server 
```sh
bash /opt/back_up_nagios.sh
```

###2.3 Cài đặt Nagios Client ( sử dụng NRPE)
<a name="cdclient"> </a> 

####2.3.1 Trên Centos Client 
<a name="trence"> </a> 

####2.3.1.1 Trên Centos 6.x
<a name="cent6"> </a> 

**Bước 1** : Cài đăt gói EPEL :

```sh
wget http://epel.mirror.net.in/epel/6/i386/epel-release-6-8.noarch.rpm
rpm -Uvh epel-release-6-8.noarch.rpm
```
**Bước 2** : Cài đặt **nrpe** và **nagios plugin**

```sh
yum install nrpe nagios-plugins-all openssl
```
**Bước 3** : Chỉnh sửa file cấu hình NRPE và khởi động 
```sh
vi /etc/nagios/nrpe.cfg
```
Tìm và chỉnh sửa như sau :

```sh
allowed_hosts=127.0.0.1, 172.16.69.221
```
Lưu file và thoát

**Chú ý** :  Thay IP 172.16.69.221 với Nagios server IP của bạn 

**Bước 4** : Khởi động Nagios NRPE

```sh
service nrpe start
chkconfig nrpe on
```

####2.3.1.2 TrênCentos 7 
<a name="cent7"> </a> 

**Bước 1** : Cài đăt gói EPEL :

```sh
yum install epel-release -y
```
**Bước 2** : Cài đặt **nrpe** và **nagios plugin**

```sh
yum install nrpe nagios-plugins-all openssl
```
**Bước 3** : Chỉnh sửa file cấu hình NRPE và khởi động 
```sh
vi /etc/nagios/nrpe.cfg
```
Tìm và chỉnh sửa như sau :

```sh
allowed_hosts=127.0.0.1, 172.16.69.221
```
Lưu file và thoát

**Chú ý** :  Thay IP 172.16.69.221 với Nagios server IP của bạn 

**Bước 4** : Khởi động Nagios NRPE

```sh
systemctl start nrpe
chkconfig nrpe on
```

###2.3.2 Trên Ubuntu/Debian Client :
<a name="ubuntu"> </a> 

**Bước 1** : Cài đặt **nrpe** và **nagios plugin**

```sh
apt-get install nagios-nrpe-server nagios-plugins -y
```
**Bước 2** : Chỉnh sửa file cấu hình NRPE và khởi động 
```sh
vi /etc/nagios/nrpe.cfg
```
Tìm và chỉnh sửa như sau :

```sh
allowed_hosts=127.0.0.1, 172.16.69.221
```
Lưu file và thoát

**Chú ý** :  Thay IP 172.16.69.221 với Nagios server IP của bạn 

**Bước 4** : Khởi động Nagios NRPE

```sh
/etc/init.d/nagios-nrpe-server restart
```

**Chú ý** : Sau khi cài đặt xong tại phía client, quay lại Nagios-Server và tiếp tục mục **3** để tạo file cấu hình cho các host client vừa thêm.

##3 Cấu hình cho host client
<a name="chclient"> </a> 

###3.1 Tạo cấu hình cho host client
<a name="taoch"> </a> 

Chỉnh file cấu hình của Nagios, đặt tất cả các file cấu hình của host client là server vào cùng 1 thư mục :

```sh
vi /usr/local/nagios/etc/nagios.cfg
```
Tìm và bỏ dấu "#" ở trước dòng sau : 

```sh
cfg_dir=/usr/local/nagios/etc/servers
```
Lưu file và thoát

Tạo thư mục chứa các file cấu hình và tạo file cấu hình cho client, ví dụ ở đây client có hostname là zabbix : 

```sh
mkdir /usr/local/nagios/etc/servers
vi /usr/local/nagios/etc/servers/zabbix.cfg
```
Định nghĩa host client với các thông tin sau

```sh
define host{
use                             linux-server
host_name                       zabbix			#hostname host client
alias                           zabbix			#bi danh của host client
address                         172.16.69.230	#IP  host client
max_check_attempts              5
check_period                    24x7
notification_interval           30
notification_period             24x7
}
```
Lưu file và thoát

Restart service và vào giao diện Web kiểm tra
```sh
systemctl restart nagios
```
![nagios](/images/nagios03.png)

###3.2 Cấu hình cho các thông số phần cứng
<a name="chtspc"> </a> 

###3.3 Cấu hình cho service SSH 
<a name="chssh"> </a> 

```sh
vi /usr/local/nagios/etc/servers/zabbix.cfg

define host{
		use                             linux-server
		host_name                       zabbix			#hostname của host client
		alias                           zabbix			#bí danh của host client
		address                         172.16.69.230	#IP của host client
		max_check_attempts              5
		check_period                    24x7
		notification_interval           30
		notification_period             24x7
}
define service {
        use                             generic-service
        host_name                       zabbix
        service_description             SSH
        check_command                   check_ssh
        notifications_enabled           0
        }
define service {
        use                             generic-service
        host_name                       zabbix
        service_description             PING2
        check_command                   check-host-alive
        notifications_enabled           0
        }
```
Lưu file và thoát

Restart service và vào giao diện Web kiểm tra

```sh
systemctl restart nagios
```
![nagios](/images/nagios04.png)
##4. Cấu hình gửi mail cảnh báo cho Nagios
<a name="chmail"> </a> 

**Chú ý** : Làm trên cả Nagios Server và Nagios Backup

**Bước 1** : Cài đặt mail postfix
```sh
yum -y install postfix cyrus-sasl-plain mailx
```
Khởi động lại postfix để nhận SASL framework và cho phép postfix khởi động cùng hệ thống :

```sh
systemctl restart postfix
systemctl enable postfix
```
Mở file cấu hình `/etc/postfix/main.cf` và thêm các dòng sau 
```sh
relayhost = [smtp.gmail.com]:587
smtp_use_tls = yes
smtp_sasl_auth_enable = yes
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
smtp_tls_CAfile = /etc/ssl/certs/ca-bundle.crt
smtp_sasl_security_options = noanonymous
smtp_sasl_tls_security_options = noanonymous
```
Lưu file và thoát

Tạo 1 file `/etc/postfix/sasl_passwd` và thêm các thông số :
```sh
vi /etc/postfix/sasl_passwd
	 [smtp.gmail.com]:587 username:password
```
Thay `username:password` với email và password của bạn

Phân quyền cho thư mục 
```sh
chown root:postfix /etc/postfix/sasl_passwd*
chmod 640 /etc/postfix/sasl_passwd*
systemctl reload postfix
```
Gửi mail test 
```sh
echo "This is a test." | mail -s "test message" manhdinh@gmail.com
```
Kiểm tra hòm thư của bạn.

**Bước 2** : Cấu hình trên Nagios Core để gửi mail notify
Mở file `/usr/local/nagios/etc/objects/contacts.cfg` và thay thế email của bạn
![nagios](/images/nagios07.png)

**Bước 3** : Mở file cấu hình của host client và thêm contact 
![nagios](/images/nagios08.png)

Restart service nagios
```sh
systemctl restart nagios
```
**Bước 4** : Kiểm tra
Tắt thử host client, Nagios server sẽ gửi mail thông báo
![nagios](/images/nagios09.png)
![nagios](/images/nagios10.png)