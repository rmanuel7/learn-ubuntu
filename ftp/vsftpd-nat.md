# How to configure VSFTPD server behind a firewall for handling internal and external IPs

By default, your EC2 instance normally has an external IP address for connections coming from the public, and an internal IP address for connections coming from your internal network. The internal network is normally for other servers that want to connect to your FTP servers locally without going through the external IP address. When it comes to AWS EC2, data transfer within a local VPC should be free, but when you access your FTP server through the external network, it really costs you money if you transfer a lot of data day by day.

The idea of this configuration allows us to separate VSFTPD configuration for external connections and internal connections. The purpose is to benefit you when you transfer data from one server to the FTP server within a VPC so that it doesn’t cost you money. Otherwise, if you use default configuration, FTP server always responds back to you by external IP (is out-going data transfer) which really costs you money if your files are really heavy.  
  
Examples: Your EC2 instance server has:

*   External IP: 123.123.123.123 (which is not showing when entering the command `ip a` in the command prompt on EC2 but it’s really attached to your EC2)
*   Internal IP: 192.168.0.1

If you connect to your FTP server on _external IP: 123.123.123.123_, the FTP server IP will use this _external IP_ to respond back to you. Similarly, if you connect to your FTP server on _internal IP: 192.168.0.1_, the FTP server will use this _internal IP_ to respond back to you.  
  
If you don’t configure  the `pasv_address` directive in `vsftpd.conf`, by default, _“the address is taken from the incoming connected socket”_ as mentioned in [manual](http://vsftpd.beasts.org/vsftpd_conf.html). Thus, you might end up transferring data by your external IP even if you connect to your FTP server through the FTP server’s internal network. **This really costs you money for outgoing data transfer on AWS**.

In this blog, I’d like to guide you on configuring vsftpd service on an EC2 instance for handling either external or internal connections and returning the corresponding FTP’s IP address that users used to connect to.

Requirements:

*   EC2: Ubuntu 20.04 AMI (you can choose your Linux Platform)
*   External IP: 123.123.123.123/24
*   Internal IP: 192.168.0.123/24

## Step 1 – Install vsftpd, iptables, and iptables-persistence

1.  First, you’ll need to install vsftpd. Click [here](https://turndevopseasier.wordpress.com/2023/10/24/how-to-configure-ftp-on-aws-ec2/) to know how to install vsftpd.

2.  Install `iptables` and `iptables-persistent`

When running this command, `iptables-persistent` package will have a dependency package called `netfilter-persistent`. We will know what this package is used for later.

```
sudo apt-get install iptables iptables-persistent -y
```

3.  Make sure `vsftpd` service is started and set to start on boot:

```
sudo systemctl start vsftpd
sudo systemctl enable vsftpd
```

## Step 2 – Configure vsftpd for handling external and internal networks

### Add `vsftpd.conf` for `internal` network:

1.  Configure `vsftpd.conf` for listening on port 21 to respond to the internal network:

```
sudo nano /etc/vsftpd.conf
```

2.  Add the following content to the file:

```
allow_writeable_chroot=YES
anonymous_enable=NO
chmod_enable=NO
chroot_local_user=YES
guest_enable=YES
guest_username=ftpusers
hide_ids=YES
listen=YES
local_enable=YES
ls_recurse_enable=YES
max_clients=128
max_per_ip=16
nopriv_user=ftpnobody
pam_service_name=vsftpd
pasv_enable=YES
pasv_min_port=10000
pasv_max_port=11000
connect_from_port_20=NO
secure_chroot_dir=/var/run/vsftpd/empty
local_root=/home/$USER/ftp
userlist_enable=YES
userlist_file=/etc/vsftpduserlist.conf
userlist_deny=NO
write_enable=YES
xferlog_enable=YES
listen_port=21
```

3.  Save and close the files.

4.  Restart the vsftpd service to apply these changes

```
sudo systemctl restart vsftpd
```

### Forward port 21 from the firewall (using iptables) to port 2121 on the ftp server

1.  If you’re using EC2, remember open port `21` and port range from `10000 - 11000` on VPC Security Group. Click [here](https://turndevopseasier.wordpress.com/2023/10/24/how-to-configure-ftp-on-aws-ec2/#:~:text=Step%202%20%E2%80%93%20Open%20the%20FTP%20ports%20in%20VPC%20on%20your%20EC2%20instance) to see the guide
2.  Create a `/etc/iptables/rules.v4` file

```
sudo nano /etc/iptables/rules.v4
```

3.  And add the content below:

```
*nat
:PREROUTING ACCEPT [0:0]
:INPUT ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
:POSTROUTING ACCEPT [0:0]
:ftpport - [0:0]
-A PREROUTING -p tcp -m tcp --dport 21 -j ftpport

-A ftpport -s 192.168.0.0/24 -j RETURN
-A ftpport -s 127.0.0.0/24 -j RETURN
-A ftpport -p tcp -j DNAT --to-destination :2121
-A POSTROUTING -p tcp -m tcp --sport 2120 -j SNAT --to-source :20
COMMIT
```

4.  Start `netfilter-persistent` service to import the iptables rules above, also set it to start on boot:

```
sudo systemctl start netfilter-persistentsudo systemctl enable netfilter-persistent
```

### Add a second vsftpd service (named vsftpd-nat) for external network

1.  Create a `vsftpd-nat.conf` file with content that is almost similar to `vsftpd.conf`. This `vsftpd-nat` service is listening on port `2121` and respond with external IP.

```
sudo nano /etc/vsftpd-nat.conf
```

2.  Add the following content to the file:

```
allow_writeable_chroot=YES
anonymous_enable=NO
chmod_enable=NO
chroot_local_user=YES
guest_enable=YES
guest_username=ftpusers
hide_ids=YES
listen=YES
local_enable=YES
ls_recurse_enable=YES
max_clients=128
max_per_ip=16
nopriv_user=ftpnobody
pam_service_name=vsftpd
pasv_enable=YES
pasv_min_port=10000
pasv_max_port=11000
secure_chroot_dir=/var/run/vsftpd/empty
local_root=/home/$USER/ftp
userlist_enable=YES
userlist_file=/etc/vsftpduserlist.conf
userlist_deny=NO
write_enable=YES
xferlog_enable=YES
# Here is the difference we make vsftpd-nat.conf listen on port 2121 and handle connections from external ip
listen_port=2121
pasv_address=123.123.123.123
connect_from_port_20=YES
ftp_data_port=2120
```

### Add a custom systemd service for vsftpd-nat.config

1.  Let’s create `/etc/systemd/system/vsftpd-nat.service`

```
sudo nano /etc/systemd/system/vsftpd-nat.service
```

2.  Add the following content to the file:

```
[Unit]
Description=vsftpd-nat FTP server
After=network.target

[Service]
Type=simple
ExecStart=/usr/sbin/vsftpd /etc/vsftpd-nat.conf
ExecReload=/bin/kill -HUP $MAINPID
ExecStartPre=-/bin/mkdir -p /var/run/vsftpd/empty

[Install]
WantedBy=multi-user.target
```

3.  Start vsftpd-nat service and set it to start on boot

```
sudo systemctl start vsftpd-nat
sudo systemctl enable vsftpd-nat
```

Done! Now we have finished configuring vsftpd service for handling either external or internal networks.

If you’re interested in building up an SFTP server, I also wrote [Secure File Transfers: Setting Up SFTP on Ubuntu 24.04](https://turndevopseasier.com/2025/05/25/secure-file-transfers-setting-up-sftp-on-ubuntu-24-04/) to walk you through the process. Hope you enjoy reading that

- - -

### Discover more from Turn DevOps Easier

Subscribe to get the latest posts sent to your email.

Type your email… 

        Subscribe

### Share this:

*   [Click to share on X (Opens in new window) X](https://turndevopseasier.com/2023/10/22/how-to-set-up-vsftpd-server-behind-a-firewall-for-handling-internal-and-external-ips/?share=twitter&nb=1)
*   [Click to share on Facebook (Opens in new window) Facebook](https://turndevopseasier.com/2023/10/22/how-to-set-up-vsftpd-server-behind-a-firewall-for-handling-internal-and-external-ips/?share=facebook&nb=1)
*   [Click to share on LinkedIn (Opens in new window) LinkedIn](https://turndevopseasier.com/2023/10/22/how-to-set-up-vsftpd-server-behind-a-firewall-for-handling-internal-and-external-ips/?share=linkedin&nb=1)
*   [Click to share on Reddit (Opens in new window) Reddit](https://turndevopseasier.com/2023/10/22/how-to-set-up-vsftpd-server-behind-a-firewall-for-handling-internal-and-external-ips/?share=reddit&nb=1)
*   [Click to share on Threads (Opens in new window) Threads](https://turndevopseasier.com/2023/10/22/how-to-set-up-vsftpd-server-behind-a-firewall-for-handling-internal-and-external-ips/?share=threads&nb=1)
*   [Click to share on Telegram (Opens in new window) Telegram](https://turndevopseasier.com/2023/10/22/how-to-set-up-vsftpd-server-behind-a-firewall-for-handling-internal-and-external-ips/?share=telegram&nb=1)
*   [Click to share on Bluesky (Opens in new window) Bluesky](https://turndevopseasier.com/2023/10/22/how-to-set-up-vsftpd-server-behind-a-firewall-for-handling-internal-and-external-ips/?share=bluesky&nb=1)

Like Loading...

### _Related_

[Avoid unexpected Regional Data Transfer from an FTP server in AWS](/2025/06/13/avoid-unexpected-regional-data-transfer-from-an-ftp-server-in-aws/?relatedposts_hit=1&relatedposts_origin=111&relatedposts_position=0 "Avoid unexpected Regional Data Transfer from an FTP server in&nbsp;AWS")June 13, 2025In "AWS"

[How to configure FTP on AWS EC2](/2023/10/24/how-to-configure-ftp-on-aws-ec2/?relatedposts_hit=1&relatedposts_origin=111&relatedposts_position=1 "How to configure FTP on AWS&nbsp;EC2")October 24, 2023In "FTP"

[Secure File Transfers: Setting Up SFTP on Ubuntu 24.04](/2025/05/25/secure-file-transfers-setting-up-sftp-on-ubuntu-24-04/?relatedposts_hit=1&relatedposts_origin=111&relatedposts_position=2 "Secure File Transfers: Setting Up SFTP on Ubuntu&nbsp;24.04")May 25, 2025In "Tutorials"

[aws cost optimization](https://turndevopseasier.com/tag/aws-cost-optimization/)[aws data transfer](https://turndevopseasier.com/tag/aws-data-transfer/)[configure vsftpd](https://turndevopseasier.com/tag/configure-vsftpd/)[File transfer softwares](https://turndevopseasier.com/tag/file-transfer-softwares/)[ftp setup](https://turndevopseasier.com/tag/ftp-setup/)

![Unknown's avatar](https://1.gravatar.com/avatar/41f2a59f73db4af7564b76a93312897cd3b7c3cddb5c26a15c3ded47ac24255f?s=100&d=identicon&r=G)

## Published by binh

[View all posts by binh](https://turndevopseasier.com/author/binhdt26/)
