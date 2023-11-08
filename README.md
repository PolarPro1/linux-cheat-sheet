# linux commands (commands used in network services)

## Common Commands
```pwd``` - used to find absolute path
<br>
```cat``` - display contents
<br>
```ls``` - list contents
-  ```-Z``` finds selinux labels for files/directories

```cd``` - change directory

<br>

```wc``` - wordcount
- ```-l``` for line count

```which (insert name of syslog daemon)``` - to get absolute path name

```mkdir``` - create a directory

```rmdir``` or ```rm``` - delete a directory or a file

```cp``` - copy a file/directory

```mv``` - move or rename files/directories

```touch``` - create an empty file

```nano``` - create and edit a file

```chmod``` - change file permissions

```chown``` - change file ownership

```grep``` - search for a specified thing

```ps``` - lists the running processes

```kill``` - terminates a process

```top``` or ```htop``` - monitor system and process activity

```useradd``` - create a new user

```passwd``` - change the user's password

```userdel``` - delete a user

```sudo``` - execute commands with superuser privilages



## Example Questions 

#### Gives the name of the first directory (alphabetically) of / that has no read permission for other.
```find / -maxdepth 1 -type d -not -perm /o=r -print | sort | head -n 1```

#### Create a symbolic link so that the file /usr/share/dict/words appears as /root/words
```ln -s /usr/share/dict/words /root/words```

#### Change the owner of the file /etc/ntp.conf to operator 
```sudo chown operator /etc/ntp.conf```

#### Change the SELinux type
```chcon -t [TYPE] [file/folder path]```

#### Get the default SELinux security context for the specified path from the file contexts configuration
```matchpathcon [FILEPATH]```

#### Search for allow rules that are affected by a specific Boolean
```sesearch --allow --bool [BOOLEAN_NAME]```

#### The syslog daemon you investigated is allowed to open a number of ports, both tcp and udp. Use the sesearch on the syslogd_t type, focusing on tcp sockets and the name_bind permission. Include the -C to better understand conditional rules.
#### You should ignore rules where the line begins with DT or DF. This indicates the conditional rule is currently disabled.
```sesearch -s syslogd_t -c tcp_socket -AC -p name_bind | grep -vE "^(DT|DF)"```


# FIREWALL LINUXZOO LAB

#### basic firewall settings that limit and log pings to 1 a second. port 80 is also closed
```
#!/bin/bash
#
iptables -F INPUT
iptables -F OUTPUT
iptables -F FORWARD
iptables -P INPUT ACCEPT
iptables -P OUTPUT ACCEPT
iptables -P FORWARD DROP
#
# Accept ongoing connections
iptables -A INPUT -p icmp --icmp-type echo-request -m limit --limit 1/s -j ACCEPT
iptables -A INPUT -p icmp --icmp-type echo-request -j LOG --log-prefix " ICMP PING Exceeded REate: " --log-level 4
iptables -A INPUT -p icmp --icmp-type echo-request -j DROP
iptables -A INPUT -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
#
# For your own safety, stop users logging in from other VMs
#
iptables -A INPUT -m conntrack --ctstate NEW -p tcp --dport 22 ! -s 10.0.0.0/16 -j ACCEPT
iptables -A INPUT -m conntrack --ctstate NEW -p tcp --dport 23 ! -s 10.0.0.0/16 -j ACCEPT
iptables -A INPUT -i ens3 -p tcp --dport 80 -j DROP
iptables -A INPUT -i ens3 -p tcp -s 20.0.0.0/24 -j DROP
iptables -A FORWARD -j REJECT

Advanced firewall setting complete
#!/bin/bash
#
iptables -F INPUT
iptables -F OUTPUT
iptables -F FORWARD
iptables -P INPUT DROP
iptables -P OUTPUT DROP
iptables -P FORWARD DROP
#
#
iptables -A INPUT  -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A OUTPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
#
# --- Put a rule here if you want to be inserting at the start of INPUT
iptables -I INPUT -p tcp --dport telnet ! -s 10.200.0.1 -j REJECT
# ---
#
# Make sure ssh and telnet stay working, and that users on
# other VMs cannot log in.
#
iptables -A INPUT -m conntrack --ctstate NEW -p tcp --dport ssh ! -s 10.0.0.0/16 -j ACCEPT
iptables -A INPUT -m conntrack --ctstate NEW -p tcp --dport telnet ! -s 10.0.0.0/16 -j ACCEPT
iptables -A OUTPUT -m conntrack --ctstate NEW -p tcp --dport http -d 10.200.0.1 -j ACCEPT

iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A FORWARD -o eth9 -m conntrack --ctstate NEW -p tcp --dport http -d 192.168.1.5 -j ACCEPT
iptables -A FORWARD -o eth9 -m conntrack --ctstate NEW -p icmp --icmp-type echo-request -d 192.168.1.5 -j ACCEPT
```


## Rules with descriptions

#### Adds a rule to the iptables firewall that drops (rejects) any incoming TCP packets on port 80 (HTTP) that arrive on the <main_interface> network interface.
```iptables -A INPUT -i <main_interface> -p tcp --dport 80 -j DROP```

#### Adds a rule to the iptables firewall that drops (rejects) any incoming TCP packets from the source IP address range 20.0.0.0/24 that arrive on the main network device specified by <main_interface>
```iptables -A INPUT -i <main_interface> -s 20.0.0.0/24 -p tcp -j DROP```

#### Any packet that is passed through the FORWARD chain will be rejected with the default settings
```iptables -A FORWARD -j REJECT```

#### Inserts a new rule at the beginning of the INPUT chain that drops (rejects) any incoming ICMP echo requests (PING). This means that any PING that matches the criteria specified in the rule will be dropped (rejected)
```iptables -I INPUT -p icmp --icmp-type echo-request -j DROP```

#### Inserts a new rule at the beginning of the INPUT chain that accepts incoming ICMP echo requests (PING) at a rate limit of 1 PING per second. This means that any PING that matches the criteria specified in the rule will be accepted, as long as it does not exceed the rate limit
```iptables -I INPUT -p icmp --icmp-type echo-request -m limit --limit 1/second -j ACCEPT```

#### Log incoming pings that are faster than 1 per second
```iptables -I INPUT 2 -p icmp --icmp-type echo-request -m limit --limit 1/second -j LOG --log-prefix "Ping log: "```

#### To insert a single rule at the start of the INPUT chain that only allows telnet connections from 10.200.0.1 and rejects any other telnet connections, you can use the following command:
```iptables -I INPUT -p tcp --dport telnet ! -s 10.200.0.1 -j REJECT```

#### To insert a single rule at the end of the OUTPUT chain that allows http connections to the ip 10.200.0.1
```iptables -A OUTPUT -m conntrack --ctstate NEW -p tcp --dport http -d 10.200.0.1 -j ACCEPT```

#### If the machine was a router, to allow the RELATED and ESTABLISHED traffic flow in both directions
```iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT```

#### Still for a router, to allow HTTP requests to an intranet/network
```iptables -A FORWARD -o <interface> -m conntrack --ctstate NEW -p tcp --dport http -d <network/destination ip> -j ACCEPT```

#### again for the router, to allow all PING requests to an intranet/network
```iptables -A FORWARD -o <interface> -m conntrack --ctstate NEW -p icmp --icmp-type echo-request -d <network/destination ip> -j ACCEPT```


# DNS LAB

#### Basic setup of the DNS, adding a configuration to the 'named' service
```nano /etc/named.conf```
#### Append to lines in the options part: 
```forwarders { 10.200.0.1; };```
<br>
```forward only;```

#### Running the service (a restart of the firewall is needed)
```systemctl restart iptables```
<br>
```systemctl start named.service```

#### Checking if the service and the configurations work:
```dig +timeout=3 +tries=1 localhost @loalhost```

#### Creating a forwward zone for the sillynet.net
```cp /var/named/named.localhost /var/named/sillynet.zone```

#### Changing the ownership of the file to 'named'
```chown named:named sillynet.zone```

#### example forward zone
```
$TTL 3D
@       IN SOA  advanced.com me.advanced.com. (     ** primary nameserver and email **
                                        0       ; serial
                                        1D      ; refresh
                                        1H      ; retry
                                        1W      ; expire
                                        3H )    ; minimum

                                        IN NS ns1.advanced.com.  ** name sever record **

advanced.com. IN A 172.16.1.1                ** maps hostname to ip address **
              IN MX 10 mail.advanced.com.    ** mail exchange record with priority 10 **
              IN MX 20 mail.offsite.com.     ** mail exchange record priority 20 so if first email server is unreachable **
www.advanced.com. IN A 172.16.1.10           ** multiple Ip address for load balancing
                  IN A 172.16.1.11
                  IN A 172.16.1.12
mail.advanced.com. IN A 172.16.1.1           
ns1.advanced.com. IN A 172.16.1.1

examle reverse zone

$TTL 3D
@   IN  SOA     advanced.com. me.advanced.com. (      ** advanced.com is primary name server and me.advnaced.com is the email **
        0           ; Serial
        8H          ; Refresh
        2H          ; Retry
        4W          ; Expire
        1D)         ; Minimum TTL

        IN  NS  ns1.advanced.com.    ** name server record

1   IN  PTR     advanced.com.
10  IN  PTR     www.advanced.com.
11  IN  PTR     www.advanced.com.
12  IN  PTR     www.advanced.com.

Example etc/named.conf file

zone "advanced.com" IN {
        type master;
        file "/var/named/advanced.zone";
};
zone "1.16.172.in-addr.arpa" IN {
        type master;
        file "/var/named/advanced.rev";
};
```


# ADMIN
#### Get the number of partitions using block with 'sfdisk' command from a directory
```sfdisk -l [directory]```

#### Finding what physical volume is being managed by the LVM
```pvdisplay```

#### Using info from previous part to find the full path of the physical volume
```lvdisplay centos_lvm (or [VG Name])```

#### Get the absolute device name using the long listing ls command
```ls -l /dev/centos_lvm (or [directory])```

#### Finding where the partition is mounted on the device
```cat /etc/fstab```
<br>
OR
<br>
```mount -l | grep 'centos_lvm-root'```

#### To find the block id for the mapper filesystem
```blkid /dev/mapper/centos_lvm-root```

#### Find the major and the  minor number of the mapper device (in this case the soft link was used to find the numbers)
```ls -l /dev (look for the soft link associated with the mapper, here it is dm-0)```

#### Find how much swap space has been allocated on the computer from the '/proc' directory
```cat /proc/swaps (the size is in bytes)```

#### Find the process id of a process
```ps -C [process name]```

#### Kill a process (the process id must be used)
```kill [process id]```

#### Find the full path to the systemd config file which controls a service (in this case rsyslog)
```systemctl show [process name] (look for FragmentPath)```

#### Find the line that configures the environmental variables of a process (in this case rsyslog)
```systemctl show [process name] | grep 'EnvironmentFile'```

#### Restart a process
```systemctl restart [process name]```

#### Start a service/process
```systemctl start [service name]```

#### Find the main PID of a service or a process
```systemctl show -p MainPID [service name]```

#### Find the user that is the owner of the process/service
```ps -o user= -p [process name]```

#### Find the PID of the first child in the process/service
```pstree -p [process name/id]```

#### Set the process/service to run everytime the VM boots
```systemctl enable [process name]```

#### Find the number of processes running at boot
```systemctl list-unit-files | grep 'enabled' | wc```

#### See the number of processes which are socket units
```systemctl list-unit-files --type=socket | grep 'enabled' | wc```

#### Set the process so it doesn't run when the VM boots
```systemctl disable [process name]```

# Apache

#### Start the HTTP service
```systemctl start httpd.service```

#### Reload the HTTP service
```systemctl reload httpd.service```

#### Reset the firewall if previously changed
```systemctl restart iptables.service```

#### Enabling the feature to have the HTML being public from a user's home directory
(the server will only show contents from this directory ```/home/dave/public_html/hello.html```)

From ```/etc/httpd/conf.d/userdir.conf```

The line ```UserDir disable``` needs to be commented and ```UserDir public_html``` needs to be uncommented

By default SELinux is forbidden from reading any files in ```/home```

Check this using ```getsebool httpd_read_user_content```

If needed the SELinux can be chnaged to have access by:

```setsebool -P httpd_read_user_content 1```


- **remember others need execute priveledges on the directory**
- **example virtual hosts named web and vm**
- **vm has a rule to redirect host-2-161.linuxzoo.net to vm-2-...**
- **unless in /~dave**
```
<VirtualHost *:80>
    ServerAdmin bob@gmail.com
    DocumentRoot /home/dave/public_html/web
    ServerName web-2-161.linuxzoo.net

    ErrorLog logs/web-error_log
    CustomLog logs/web-access_log common
</VirtualHost>

<VirtualHost *:80>
    ServerAdmin bob@gmail.com
    DocumentRoot /home/dave/public_html/vm
    ServerName vm-2-161.linuxzoo.net
    ServerAlias host-2-161.linuxzoo.net

    ErrorLog logs/vm-error_log
    CustomLog logs/vm-access_log common


    RewriteEngine On
    RewriteCond %{HTTP_HOST} ^host-2-161.linuxzoo.net [NC]
    RewriteCond %{REQUEST_URI} !^/~dave
    RewriteRule ^(.*)$ http://vm-2-161.linuxzoo.net$1 [L]
</VirtualHost>
```
