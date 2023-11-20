# BIND-CloudFlare-DoT
Configure Bind 9.19 to query CloudFlare using DNS over TLS

## Server Details
* Virtual machine running Debian 12, 1 core, 1GB of RAM, 16GB of storage, 1 NIC
* Debian configured with testing repositories to get BIND9.19.x

## Setting the repositories to the Development
1. Make sure you're up to date by simply running apt update && apt upgrade -y
2. To change the repos, run apt edit-sources and select an editor, use nano unless you've installed other editors.
3. In nano, type
 *  **alt+r** 
 * **enter**
 * ```bookworm```
 * **enter**
 * ```testing```
 * **enter**
 * **A**
4. All the repos should be changed to testing now. **CTRL+X, Y** to write it to file
5. Run ```sudo apt update && upgrade -y```
6. When you see the change log simply type Q to exit to allow the install to continue.

## Installing Bind 9.19

Now install Bind, run the commdn apt install bind9 and verify the version shows 9.19.x

Once it's installed, you'll need to edit a few config files. All of these files will be located in /etc/bind/

I was lazy and included all the config in named.conf.local but you could create a new file for the TLS configuration and call it from named.conf. At some point CloudFlare is going to change their certificate and you'll need to update the config.

1. Create a TLS entry specific for CloudFlare:
```
   tls cloudflare-dot {
    ca-file "/etc/ssl/certs/DigiCert_TLS_ECC_P384_Root_G5.pem";
    remote-hostname "one.one.one.one";
    prefer-server-ciphers yes;
    session-tickets no;
    };
```
> [!IMPORTANT]
>   The important line is the ca-file, you can check this be browsing to https://1.1.1.1 and check the root cert of the cert used to encrypt the web page. 
>   Then compare it to the certs in your /etc/ssh/certs folder. It's typically not named exactly but you'll see it when you see it.

2. Then add this line into your options file. This code can be added to additional options in your file, just make sure to format it right. 
```
    options {
      forwarders port 853 tls cloudflare-dot {
            1.1.1.1;
            1.0.0.1;
    };	
  };
```
3. Test your code. Run the command ```named-checkconf``` to validate your configurations. Fix anything that it complains about.
4. Restart Bind ```sudo systemctl restart named```

## Testing DNS Traffic for TLS

To test to see if it's actually forwarding over TLS, you can use a simple network capture tool. 

> [!TIP]
> If this is a base install of Debian, you'll need to install tcpdump using ```apt install tcpdump -y```
```
tcpdump -ni eth0 -p port 53 or port 853
```
Then begin sending DNS queries to it from another server, you should see your DNS server's console light up

My test lab server is at 10.0.100.5, as you can see 1.0.0.1 and my server are communicating over TCP port 853, TLS. Port 53 is not being used.

```
17:32:57.920725 IP 1.0.0.1.853 > 10.0.100.5.35369: Flags [.], ack 304, win 7, options [nop,nop,TS val 702996937 ecr 2435609577], length 0
17:32:57.920754 IP 1.0.0.1.853 > 10.0.100.5.35369: Flags [P.], seq 2890:2914, ack 304, win 8, options [nop,nop,TS val 702996937 ecr 2435609577], length 24
17:34:19.192090 IP 1.0.0.1.853 > 10.0.100.5.33129: Flags [.], ack 296, win 7, options [nop,nop,TS val 1276556294 ecr 2435690850,nop,nop,sack 1 {303:304}], length 0
17:34:19.192102 IP 1.0.0.1.853 > 10.0.100.5.33129: Flags [.], ack 304, win 7, options [nop,nop,TS val 1276556294 ecr 2435690851], length 0
17:34:19.192121 IP 1.0.0.1.853 > 10.0.100.5.33129: Flags [P.], seq 2891:2915, ack 304, win 8, options [nop,nop,TS val 1276556294 ecr 2435690851], length 24
17:34:19.192135 IP 10.0.100.5.33129 > 1.0.0.1.853: Flags [R], seq 1058532512, win 0, length 0
17:34:19.192142 IP 1.0.0.1.853 > 10.0.100.5.33129: Flags [F.], seq 2915, ack 304, win 8, options [nop,nop,TS val 1276556294 ecr 2435690851], length 0
```

## Footnotes

[^1] Reference link to BIND 9.19 Documentation for DoT
[^2] Reference link to changing repos to testing stream
