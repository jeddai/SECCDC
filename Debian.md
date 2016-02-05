# Debian Email Launch Codes
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
    - Simple things => iptables -A INPUT -s 192.168.1.10 -d 10.1.15.1 -p tcp --dport 22 -j ACCEPT
        - -A => Tells iptables to 'append' this rule to the INPUT Chain
        - -s => Source Address. This rule only pertains to traffic coming FROM this IP. Substitute with the IP address you are SSHing from.
        - -d => Destination Address. This rule only pertains to traffic going TO this IP. Substitute with the IP of this server.
        - -p => Protocol. Specifying traffic which is TCP.
        - --dport => Destination Port. Specifying traffic which is for TCP Port 22 (SSH)
        - -j => Jump. If everything in this rule matches then 'jump' to ACCEPT
5. Lock down SSH
    - `nano /etc/ssh/sshd_config` and find the line that looks like so
    ```
    PasswordAuthentication <something here other than no>
    ```
    and change it so that it says
    ```
    PasswordAuthentication no
    ```
    Do ^X and save. Then run `service ssh restart && service sshd restart`.
    -
