
## 1.覆盖vsftpd.conf
## 2. 添加用户密码/etc/vsftpd/ftpusers
## 
```
db_load -T -t hash -f /etc/vsftpd/ftpusers /etc/vsftpd/login.db
mkdir -p /etc/vsftpd/vsftpd_user_conf/
echo "local_root=/" > /etc/vsftpd/vsftpd_user_conf/root
echo "/root" >  /etc/vsftpd.chroot_list

echo "auth required pam_userdb.so db=/etc/vsftpd/login" > /etc/pam.d/vsftpd
echo "account required pam_userdb.so db=/etc/vsftpd/login" >>/etc/pam.d/vsftpd
```
