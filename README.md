# Tích hợp LDAP (LDAPS) cho vcenter - netbox - graylog bằng freeipa và kích hoạt OTP
## Giới thiệu

### Ưu điểm

- Chỉ cần một máy linux cấu hình vừa phải, 4CPU, 8GB RAM, 100GB disk.
- Không cần kết nối ra Internet để nhận OPT.
- Không bị limit user sử dụng khi tích hợp
- Free

### Nhược điểm

- Phương án không thấy vlware công bố hỗ trợ nên phải tự vọc, tự xử lý.

## Môi trường

Centos 8, RHEL8, Centos 9 hoặc RHEL 9

Lab này sử dụng RHEL8

Lưu ý trong lab này dùng freeipa làm dns cho domain hcdlab.local luôn mặc dù trước đó với vcenter đã có dns server khác.

Vcenter 7.0.3, domain vcenter.labhtv.local (10.10.240.245)

Freeipa 4.9.11, domain ipa.congtolab.local (10.10.240.186)

## Cài đặt FreeIPA

Setup ip tĩnh nếu cần, giả sử ip là 10.10.240.186 

Cấu hình hostname

```
hostnamectl set-hostname ipa.conglab.local

echo "10.10.240.186  ipa.conglab.local ipa" >> /etc/hosts
```

Cấu hình timezone

Cấu hình firewalld, selinux với centos, rhel

Cài đặt module hỗ trợ bổ sung gói freeipa

```
dnf module enable idm:DL1
```

Cài đặt gói freeipa 

```
dnf install ipa-server ipa-server-dns -y
```

Cấu hình freeipa có tích hợp DNS (DNS server là máy cài freeipa luôn)

```
ipa-server-install --setup-dns
```

Trong các màn hình khai báo, nhập các tham số cần thiết.

```
Server host name [ipa.conglab.local]: ipa.conglab.local

Please confirm the domain name [conglab.local]: conglab.local

Please provide a realm name [CONGLAB.LOCAL]: CONGLAB.LOCAL

Directory Manager password:
Password (confirm):

IPA admin password:
Password (confirm):

Do you want to configure DNS forwarders? [yes]: yes

Do you want to configure these servers as DNS forwarders? [yes]:yes

Enter an IP address for a DNS forwarder, or press Enter to skip:

Do you want to search for missing reverse zones? [yes]: no

NetBIOS domain name [CONGLAB]: CONGLAB

Do you want to configure chrony with NTP server or pool address? [no]: yes
Enter NTP source server addresses separated by comma, or press Enter to skip:

Enter a NTP source pool address, or press Enter to skip:

Continue to configure the system with these values? [no]: yes
```

Chờ màn hình cài đặt thực hiện các bước 

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/bcc1499b-1250-4b7e-bbf9-510813d2d689/6c2243d9-b0ff-46c8-8f1e-37394649cd55/Untitled.png)

Sau khi cài xong sẽ có màn hình sau

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/bcc1499b-1250-4b7e-bbf9-510813d2d689/70db3a59-3889-4b2b-8989-772343021e32/Untitled.png)

Trường hợp có dùng firewalld thì cần allow các port ở trên

```
firewall-cmd --permanent --add-service=ntp
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-service=https
firewall-cmd --permanent --add-service=ldap
firewall-cmd --permanent --add-service=ldaps
firewall-cmd --permanent --add-service=kerberos
firewall-cmd --permanent --add-service=kpasswd
firewall-cmd --reload
```

Kiểm tra xem freeipa hoạt động chưa bằng lệnh `ipactl status` kết quả như sau là ok

```
[root@ipa ~]# ipactl status
Directory Service: RUNNING
krb5kdc Service: RUNNING
kadmin Service: RUNNING
named Service: RUNNING
httpd Service: RUNNING
ipa-custodia Service: RUNNING
pki-tomcatd Service: RUNNING
ipa-otpd Service: RUNNING
ipa-dnskeysyncd Service: RUNNING
ipa: INFO: The ipactl command was successful

```

Cần thiết khởi động lại máy sau khi cài và kiểm tra status lại cho chắc.

Xác nhận lại token admin bằng lệnh `kinit admin` và `klist` , nếu đăng nhập thành công và in ra kết quả token thì freeipa đã hoạt động.

```
[root@ipa ~]# kinit admin
Password for admin@CONGLAB.LOCAL:
[root@ipa ~]# klist
Ticket cache: KCM:0
Default principal: admin@CONGLAB.LOCAL

Valid starting       Expires              Service principal
06/07/2024 14:06:30  06/08/2024 13:57:22  krbtgt/CONGLAB.LOCAL@CONGLAB.LOCAL

```

Truy cập vào web bằng URL [`https://ipa.conglab.local/ipa/ui/`](https://ipa.conglab.local/ipa/ui/) 

Nhập tài khoản admin và mật khẩu ở bước cài đặt trước đó.

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/bcc1499b-1250-4b7e-bbf9-510813d2d689/f149370e-e611-4fd2-9031-b095ef71bdde/Untitled.png)

Đăng nhập

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/bcc1499b-1250-4b7e-bbf9-510813d2d689/1a3b0cac-0f81-42c0-b053-46530b60a231/Untitled.png)

Tới bước này đã hoàn thành việc cài đặt freeipa, chuyển sang các bước tích hợp với các nền tảng như vcenter, netbox và kích hoạt OTP

## Tích hợp freeipa với vcenter và kích hoạt OTP

Mặc định và háng vlware không tuyên bố hỗ trợ freeipa nên trước khi tích hợp freeipa với vcenter và sử dụng được OPT thì cần điều chỉnh lại schema cho freeipa tương thích với openldap (cái mà vlware hỗ trợ).

Cấu hình điều chỉnh schemal của freeipa để tương thích với vcenter.

Tạo file `vsphere_usermod.ldif` trên máy chủ freeipa

```
dn: cn=users,cn=Schema Compatibility,cn=plugins,cn=config
changetype: modify
add: schema-compat-entry-attribute
schema-compat-entry-attribute: objectclass=inetOrgPerson
-
add: schema-compat-entry-attribute
schema-compat-entry-attribute: sn=%{sn}
-
```

Tạo file `vsphere_groupmod.ldif` trên máy chủ freeipa 

```
dn: cn=groups,cn=Schema Compatibility,cn=plugins,cn=config
changetype: modify
add: schema-compat-entry-attribute
schema-compat-entry-attribute: objectclass=groupOfUniqueNames
-
add: schema-compat-entry-attribute
schema-compat-entry-attribute: uniqueMember=%mregsub("%{member}","^(.*)accounts(.*)","%1compat%2")
-
```

Thực hiện lệnh sau để apply thay đổi các điều chỉnh trên

```
ldapmodify -x -D "cn=Directory Manager" -f vsphere_groupmod.ldif -W -v
ldapmodify -x -D "cn=Directory Manager" -f vsphere_usermod.ldif -W -v
```

Sau khi cấu hình xong, truy cập vào giao diện freeipa khai báo thêm các user để sử dụng đăng nhập vào vcenter sau này.

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/bcc1499b-1250-4b7e-bbf9-510813d2d689/bd7c9501-ebaa-4872-a162-641e275e77eb/Untitled.png)

### Tạo user cho freeipa

Tạo một vài user để kiểm tra

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/bcc1499b-1250-4b7e-bbf9-510813d2d689/1c3300a4-bfb1-47ff-b435-1666f75f4a61/Untitled.png)

Ta có 2 user 

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/bcc1499b-1250-4b7e-bbf9-510813d2d689/8b59cc7a-202f-48be-a16b-1e3ff21ad7d3/Untitled.png)

Mở các phiên đăng nhập khác để login vào các user vừa tạo để đổi mật khẩu lần đầu và xác nhận việc truy cập thành công.

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/bcc1499b-1250-4b7e-bbf9-510813d2d689/d4362813-d5b5-4801-8db6-c1d2c0940b99/Untitled.png)

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/bcc1499b-1250-4b7e-bbf9-510813d2d689/96dc3aeb-fa45-473b-8b72-6b2127ef1e41/Untitled.png)

Tới bước này ta đã có user trên freeipa để sử dụng 

### Thực hiện tích hợp freeipa và khai báo trên vcenter

Trước khi vào vcenter, ssh vào máy freeipa hoặc dùng winscp down file CA của freeipa về để dùng cho bước sau.

Tải file `/etc/ipa/ca.crt` về máy tính 

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/bcc1499b-1250-4b7e-bbf9-510813d2d689/fea96856-4e2d-4d35-b999-5d29a2ad1dbf/Untitled.png)

Khai báo file host trong vcenter để trỏ được dns của máy freeipa vì trong lab này sử dụng 2 domain khác nhau.

```
root@vcenter [ ~ ]# cat /etc/hosts
# Begin /etc/hosts (network card version)

# VAMI_EDIT_BEGIN
# Generated by Studio VAMI service. Do not modify manually.
127.0.0.1  vcenter.labhtv.local vcenter localhost
::1  vcenter.labhtv.local vcenter localhost ipv6-localhost ipv6-loopback
# VAMI_EDIT_END

10.10.240.186 ipa.conglab.local

```

Đăng nhập vào web vcenter và chọn theo hương dẫn

Chọn tab administrator ⇒ Single Sign On ⇒ Configuration ⇒ Add

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/bcc1499b-1250-4b7e-bbf9-510813d2d689/90706382-f7b4-4bfc-9c04-803843709e7c/Untitled.png)

Ở cửa sổ khai báo ADD, chọn Identity Source Type là “Open LDAP” 

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/bcc1499b-1250-4b7e-bbf9-510813d2d689/3d0e666f-45af-43a7-9c9a-e82a025f9dd8/Untitled.png)

Và khai báo các tham số nhử bên dưới, lưu ý bước chọn Certificate ta brower tới file ca.crt đã tải về trước đó.

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/bcc1499b-1250-4b7e-bbf9-510813d2d689/5aaebf8b-378d-4200-a004-b02759266234/Untitled.png)

Sau khi add thành công, ta thiết lập mặc định cơ chế đăng nhập cho domain trên freeipa. 

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/bcc1499b-1250-4b7e-bbf9-510813d2d689/8dd74d58-7f25-4d8c-b75d-de68e0ba3d91/Untitled.png)

Chuyển sang tab User and Group để kiểm tra xem user đã đồng bộ sang hay chưa.

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/bcc1499b-1250-4b7e-bbf9-510813d2d689/ad6cedb6-9910-4426-b4a4-edaa7497716f/Untitled.png)

Tới đây đã đồng bộ user từ freeipa sang nhưng chưa được phân quyền. Tiếp tục bước phân quyền trên vcenter để có thể sử dụng user của freeipa để đăng nhập vào vcenter.

Chọn cluster trên vcenter, sau đó vào tab permision 

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/bcc1499b-1250-4b7e-bbf9-510813d2d689/3df971d7-ca3e-4cd8-b4a9-102b6b197286/Untitled.png)

Chọn add thêm usre với domain của freeipa

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/bcc1499b-1250-4b7e-bbf9-510813d2d689/9a606c62-e36f-4d84-8660-4c6a57d0721e/Untitled.png)

Sau đó mở một trình duyệt khác để đăng nhập thử.

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/bcc1499b-1250-4b7e-bbf9-510813d2d689/a72ccfe3-3fe3-4c35-9f57-a0cc2696310c/Untitled.png)

Ta sẽ thấy màn hình đăng nhập của user hcd1@conglab.local. Tới bước này tôi chưa kích hoạt OTP để kiểm tra việc tích hợp trước.

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/bcc1499b-1250-4b7e-bbf9-510813d2d689/2d08b877-4d8b-49ce-9804-8252e5e154e4/Untitled.png)

### Kích hoạt tính năng OPT cho user của freeipa để sử dụng khi đăng nhập

Đăng nhập vào user admin của freeipa để kích hoạt OTP đối với các user cần thiết, tại đây ta chọn chế độ đăng nhập sử dụng bằng cách xác thực nào cho user

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/bcc1499b-1250-4b7e-bbf9-510813d2d689/43bdf481-79a6-4cf7-b474-1bef5272001f/Untitled.png)

Sau đó save lại

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/bcc1499b-1250-4b7e-bbf9-510813d2d689/3db9bd1b-1256-41d3-990a-8dc522003b4c/Untitled.png)

Tiếp tục chọn tab Action để tạo QR code cho user hcd1 

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/bcc1499b-1250-4b7e-bbf9-510813d2d689/04eb80a5-e292-4e3e-b1c1-52346f29e652/Untitled.png)

Ở màn hình khai báo dưới, có thể nhập thêm tham số, nếu không cần thì chọn ADD

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/bcc1499b-1250-4b7e-bbf9-510813d2d689/caace521-33c8-493f-aa8d-d3964867bce2/Untitled.png)

Sau khi add sẽ có QR code để gửi cho user và user cần dùng các tool quản lý QR code như google authen để quét và dùng sau này.

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/bcc1499b-1250-4b7e-bbf9-510813d2d689/f07a3190-6ac8-47b8-bd76-f8ce38470a45/Untitled.png)

Lúc này user hcd1 đã có qr code và nhận được các chuỗi số random. 

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/bcc1499b-1250-4b7e-bbf9-510813d2d689/7a0d06fb-0096-49be-95bf-01ed181b8aeb/Untitled.png)

User hcd1 bắt đầu mở trình duyệt đăng nhập của freeipa hoặc vcenter để nhập mật khẩu + chuối số trên công cụ quản lý code ở điện thoại theo dạng `Mật khẩu và nối tiếp chuỗi OPT sinh ra ở ứng dụng điện thoại"

Giả sử mật khẩu là “Hocchud0ng” và OPT là 231234 thì nhập vào khung mật khẩu là `Hocchud0ng231234` 

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/bcc1499b-1250-4b7e-bbf9-510813d2d689/70b7f5f0-0492-44c7-8008-a0188fa73529/Untitled.png)

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/bcc1499b-1250-4b7e-bbf9-510813d2d689/4ce67152-43a9-43a0-a477-53eff9329ce3/Untitled.png)

Tới đây đã hoàn thành bước cấu hình.
