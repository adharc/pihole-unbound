# Install & Setup Unbound with Pi-hole

### What is Unbound?
[Unbound](https://github.com/NLnetLabs/unbound) is a recursive DNS resolver developed by NLnet Labs that can cache and validate DNS queries using DNSSEC. Unbound also supports encrypting queries using modern protocols such as [DNS-Over-TLS](https://datatracker.ietf.org/doc/html/rfc7858) (DoT)

### Installing Unbound
Install the Unbound recursive DNS resolver:
```
sudo apt install unbound -y
```
The [root.hints](https://www.internic.net/domain/named.root) file will be automatically installed with the dependency `dns-root-data`. The file will be automatically updated by your package manager.

If you did not use a package manager, you will need to download the root.hints file manually and move it to `/var/lib/unbound/`:
```
wget -O root.hints https://www.internic.net/domain/named.root
sudo mv root.hints /var/lib/unbound/
```

### Configure `unbound`

Create the config file in  `/etc/unbound/unbound.conf.d/`:
```
sudo nano /etc/unbound/unbound.conf.d/pi-hole.conf
```

And add the folowing configuration:

```yaml
server:
    # If no logfile is specified, syslog is used
    # logfile: "/var/log/unbound/unbound.log"
    verbosity: 0

    interface: 127.0.0.1
    port: 5335
    do-ip4: yes
    do-udp: yes
    do-tcp: yes

    # Set to "yes" if you have IPv6 connectivity
    do-ip6: no

    # Leave this to no unless you have native IPv6
    prefer-ip6: no

    # Use this only when you downloaded the list of primary root servers!
    # If you use the default dns-root-data package, unbound will find it automatically
    #root-hints: "/var/lib/unbound/root.hints"

    # Trust glue only if it is within the server's authority
    harden-glue: yes

    # Require DNSSEC data for trust-anchored zones, if such data is absent, the zone becomes BOGUS
    harden-dnssec-stripped: yes

    # Don't use Capitalization randomization as it known to cause DNSSEC issues sometimes
    # see https://discourse.pi-hole.net/t/unbound-stubby-or-dnscrypt-proxy/9378 for further details
    use-caps-for-id: no

    # Reduce EDNS reassembly buffer size.
    edns-buffer-size: 1232

    # Perform prefetching of close to expired message cache entries
    # This only applies to domains that have been frequently queried
    prefetch: yes

    # One thread should be sufficient, can be increased on beefy machines. 
    # In reality for most users running on small networks or on a single machine, 
    # it should be unnecessary to seek performance enhancement by increasing num-threads above 1.
    num-threads: 1

    # Ensure kernel buffer is large enough to not lose messages in traffic spikes
    so-rcvbuf: 1m
    so-sndbuf: 1m

    # Ensure privacy of local IP ranges
    private-address: 192.168.0.0/16
    private-address: 169.254.0.0/16
    private-address: 172.16.0.0/12
    private-address: 10.0.0.0/8
    private-address: fd00::/8
    private-address: fe80::/10


    #### DNS-Over-TLS ####
    # You can use any DNS server such as Cloudflare, Google & Quad9
    # Uncomment all lines below to enable DNS-Over-TLS
    
    #tls-cert-bundle: /etc/ssl/certs/ca-certificates.crt
    #forward-zone:
    #name: "."
    #forward-tls-upstream: yes
    
    #forward-addr: 1.1.1.1@853#cloudflare-dns.com
    #forward-addr: 1.0.0.1@853#cloudflare-dns.com
    #forward-addr: 2606:4700:4700::1111@853#cloudflare-dns.com
    #forward-addr: 2606:4700:4700::1001@853#cloudflare-dns.com
```
If you have manually downloaded the root.hints file uncomment  `root-hints: "/var/lib/unbound/root.hints"`

Exit & Save the config file by pressing `Ctrl+X`, then `Y` and `Enter`


### Check unbound config file for errors
Check the config file for errors by:
```
unbound-checkconf  /etc/unbound/unbound.conf.d/pi-hole.conf 
```
This should return `no errors in /etc/unbound/unbound.conf.d/pi-hole.conf`.

Start unbound service and check whether the domain is resolving. The first query will be slow but the subsequent queries will resolve under 1ms.
```
sudo service unbound restart
dig github.com @127.0.0.1 -p 5335
```
      
### Test domain resolving & DNSSEC validation
You can test DNSSEC validation using
```
dig sigfail.verteiltesysteme.net @127.0.0.1 -p 5335
```
The first command should give a status of `SERVFAIL` with no IP address.
```
dig sigok.verteiltesysteme.net @127.0.0.1 -p 5335
```
The second command should give `NOERROR` with an IP address.

If this is not the case, you should reboot your system & try the two commands again.

### Configure Pi-hole
Configure Pi-hole to use unbound as your recursive DNS server and untick any other upstream DNS Server:

**Settings -> DNS -> Custom 1 (IPv4)**

```
127.0.0.1#5335
```

Click save.

![screenshot at 2022-09-06](https://i.imgur.com/AgEbDwm.png)
