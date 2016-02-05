# Ubuntu Launch Codes
1. Starting off - enter root user by using `su - root` or just `su`.
2. Update the crap out of it.
    - if you get a cdrom error when running `apt-get update` or `install` then go into `/etc/apt/sources.list` and comment out the cdrom lines
    ```
    apt-get update //update sources.list
    apt-get upgrade //upgrade out of date software
    ```
3. Check release version using `lsb_release -a` and if necessary run `apt-get dist-upgrade`.
4. Configure IPTables (if it is not there use `apt-get install iptables`)
    - IPTables locks down ports etc...
    - Check what is/isn't locked down using `iptables --list`
    - Simple things => `iptables -A INPUT -s 192.168.1.10 -d 10.1.15.1 -p tcp --dport 22 -j ACCEPT`
        - -A => Tells iptables to 'append' this rule to the INPUT Chain
        - -s => Source Address. This rule only pertains to traffic coming FROM this IP. Substitute with the IP address you are SSHing from.
        - -d => Destination Address. This rule only pertains to traffic going TO this IP. Substitute with the IP of this server.
        - -p => Protocol. Specifying traffic which is TCP.
        - --dport => Destination Port. Specifying traffic which is for TCP Port 22 (SSH)
        - -j => Jump. If everything in this rule matches then 'jump' to ACCEPT
    - Do something along these lines:
    ```
    //Individual rejections first:
    //Bad peoples (Block Source IP Address):
    iptables -A INPUT -s 172.34.5.8 -j DROP
    //Spammers (notice the use of FQDN):
    iptables -A INPUT -s mail.spammer.org -d 10.1.15.1 -p tcp --dport 25 -j REJECT
    ```
    - Then open up stuff:
    ```
    //MYSQL (Allow Remote Access To Particular IP):
    iptables -A INPUT -s 172.50.3.45 -d 10.1.15.1 -p tcp --dport 3306 -j ACCEPT
    //SSH:
    iptables -A INPUT -d 10.1.15.1 -p tcp --dport 22 -j ACCEPT
    //Sendmail/Postfix:
    iptables -A INPUT -d 10.1.15.1 -p tcp --dport 25 -j ACCEPT

    //FTP: (Notice how you can specify a range of ports 20-21)
    iptables -A INPUT -d 10.1.15.1 -p tcp --dport 20:21 -j ACCEPT

    //Passive FTP Ports Maybe: (Again, specifying ports 50000 through 50050 in one rule)
    iptables -A INPUT -d 10.1.15.1 -p tcp --dport 50000:50050 -j ACCEPT

    //HTTP/Apache
    iptables -A INPUT -d 10.1.15.1 -p tcp --dport 80 -j ACCEPT

    //SSL/Apache
    iptables -A INPUT -d 10.1.15.1 -p tcp --dport 443 -j ACCEPT

    //IMAP
    iptables -A INPUT -d 10.1.15.1 -p tcp --dport 143 -j ACCEPT

    //IMAPS
    iptables -A INPUT -d 10.1.15.1 -p tcp --dport 993 -j ACCEPT

    //POP3
    iptables -A INPUT -d 10.1.15.1 -p tcp --dport 110 -j ACCEPT

    //POP3S
    iptables -A INPUT -d 10.1.15.1 -p tcp --dport 995 -j ACCEPT

    //Any Traffic From Localhost:
    iptables -A INPUT -d 10.1.15.1 -s 127.0.0.1 -j ACCEPT

    //ICMP/Ping:
    iptables -A INPUT -d 10.1.15.1 -p icmp -j ACCEPT
    ```
    - Global rejects last:
    ```
    //Reject everything else to that IP:
    iptables -A INPUT -d 10.1.15.1 -j REJECT

    //Or, reject everything else coming through to any IP:
    iptables -A INPUT -j REJECT
    iptables -A FORWARD -j REJECT
    ```
5. Lock down SSH
    - `nano /etc/ssh/sshd_config` and find the line that looks like so
    ```
    PermitRootLogin <not no>
    ...
    PermitEmptyPasswords <not no>
    ...
    PasswordAuthentication <not no>
    ```
    and change it so that it says
    ```
    PermitRootLogin no
    ...
    PermitEmptyPasswords no
    ...
    PasswordAuthentication no
    ```
    Do ^X and save. Then run `service ssh restart && service sshd restart`.
