<!-- Vendored from https://github.com/trimstray/the-book-of-secret-knowledge (MIT License). Snapshot: 2026-06-14. 本文件是该知识库按分类拆分出来的一部分，供 Argus secret-knowledge skill 按需 Grep 检索。原始版权归 trimstray 及贡献者所有，保留 MIT 许可。 -->

##### Tool: [curl](https://curl.haxx.se)

```bash
curl -Iks https://www.google.com
```

  * `-I` - show response headers only
  * `-k` - insecure connection when using ssl
  * `-s` - silent mode (not display body)

```bash
curl -Iks --location -X GET -A "x-agent" https://www.google.com
```

  * `--location` - follow redirects
  * `-X` - set method
  * `-A` - set user-agent

```bash
curl -Iks --location -X GET -A "x-agent" --proxy http://127.0.0.1:16379 https://www.google.com
```

  * `--proxy [socks5://|http://]` - set proxy server

```bash
curl -o file.pdf -C - https://example.com/Aiju2goo0Ja2.pdf
```

  * `-o` - write output to file
  * `-C` - resume the transfer

###### Find your external IP address (external services)

```bash
curl ipinfo.io
curl ipinfo.io/ip
curl icanhazip.com
curl ifconfig.me/ip ; echo
```

###### Repeat URL request

```bash
# URL sequence substitution with a dummy query string:
curl -ks https://example.com/?[1-20]

# With shell 'for' loop:
for i in {1..20} ; do curl -ks https://example.com/ ; done
```

###### Check DNS and HTTP trace with headers for specific domains

```bash
### Set domains and external dns servers.
_domain_list=(google.com) ; _dns_list=("8.8.8.8" "1.1.1.1")

for _domain in "${_domain_list[@]}" ; do

  printf '=%.0s' {1..48}

  echo

  printf "[\\e[1;32m+\\e[m] resolve: %s\\n" "$_domain"

  for _dns in "${_dns_list[@]}" ; do

    # Resolve domain.
    host "${_domain}" "${_dns}"

    echo

  done

  for _proto in http https ; do

    printf "[\\e[1;32m+\\e[m] trace + headers: %s://%s\\n" "$_proto" "$_domain"

    # Get trace and http headers.
    curl -Iks -A "x-agent" --location "${_proto}://${_domain}"

    echo

  done

done

unset _domain_list _dns_list
```

___

##### Tool: [httpie](https://httpie.org/)

```bash
http -p Hh https://www.google.com
```

  * `-p` - print request and response headers
    * `H` - request headers
    * `B` - request body
    * `h` - response headers
    * `b` - response body

```bash
http -p Hh https://www.google.com --follow --verify no
```

  * `-F, --follow` - follow redirects
  * `--verify no` - skip SSL verification

```bash
http -p Hh https://www.google.com --follow --verify no \
--proxy http:http://127.0.0.1:16379
```

  * `--proxy [http:]` - set proxy server

##### Tool: [ssh](https://www.openssh.com/)

###### Escape Sequence

```
# Supported escape sequences:
~.  - terminate connection (and any multiplexed sessions)
~B  - send a BREAK to the remote system
~C  - open a command line
~R  - Request rekey (SSH protocol 2 only)
~^Z - suspend ssh
~#  - list forwarded connections
~&  - background ssh (when waiting for connections to terminate)
~?  - this message
~~  - send the escape character by typing it twice
```

###### Compare a remote file with a local file

```bash
ssh user@host cat /path/to/remotefile | diff /path/to/localfile -
```

###### SSH connection through host in the middle

```bash
ssh -t reachable_host ssh unreachable_host
```

###### Run command over SSH on remote host

```bash
cat > cmd.txt << __EOF__
cat /etc/hosts
__EOF__

ssh host -l user $(<cmd.txt)
```

###### Get public key from private key

```bash
ssh-keygen -y -f ~/.ssh/id_rsa
```

###### Get all fingerprints

```bash
ssh-keygen -l -f .ssh/known_hosts
```

###### SSH authentication with user password

```bash
ssh -o PreferredAuthentications=password -o PubkeyAuthentication=no user@remote_host
```

###### SSH authentication with publickey

```bash
ssh -o PreferredAuthentications=publickey -o PubkeyAuthentication=yes -i id_rsa user@remote_host
```

###### Simple recording SSH session

```bash
function _ssh_sesslog() {

  _sesdir="<path/to/session/logs>"

  mkdir -p "${_sesdir}" && \
  ssh $@ 2>&1 | tee -a "${_sesdir}/$(date +%Y%m%d).log"

}

# Alias:
alias ssh='_ssh_sesslog'
```

###### Using Keychain for SSH logins

```bash
### Delete all of ssh-agent's keys.
function _scl() {

  /usr/bin/keychain --clear

}

### Add key to keychain.
function _scg() {

  /usr/bin/keychain /path/to/private-key
  source "$HOME/.keychain/$HOSTNAME-sh"

}
```

###### SSH login without processing any login scripts

```bash
ssh -tt user@host bash
```

###### SSH local port forwarding

Example 1:

```bash
# Forwarding our local 2250 port to nmap.org:443 from localhost through localhost
host1> ssh -L 2250:nmap.org:443 localhost

# Connect to the service:
host1> curl -Iks --location -X GET https://localhost:2250
```

Example 2:

```bash
# Forwarding our local 9051 port to db.d.x:5432 from localhost through node.d.y
host1> ssh -nNT -L 9051:db.d.x:5432 node.d.y

# Connect to the service:
host1> psql -U db_user -d db_dev -p 9051 -h localhost
```

  * `-n` - redirects stdin from `/dev/null`
  * `-N` - do not execute a remote command
  * `-T` - disable pseudo-terminal allocation

###### SSH remote port forwarding

```bash
# Forwarding our local 9051 port to db.d.x:5432 from host2 through node.d.y
host1> ssh -nNT -R 9051:db.d.x:5432 node.d.y

# Connect to the service:
host2> psql -U postgres -d postgres -p 8000 -h localhost
```

___

##### Tool: [linux-dev](https://www.tldp.org/LDP/abs/html/devref1.html)

###### Testing remote connection to port

```bash
timeout 1 bash -c "</dev/<proto>/<host>/<port>" >/dev/null 2>&1 ; echo $?
```

  * `<proto` - set protocol (tcp/udp)
  * `<host>` - set remote host
  * `<port>` - set destination port

###### Read and write to TCP or UDP sockets with common bash tools

```bash
exec 5<>/dev/tcp/<host>/<port>; cat <&5 & cat >&5; exec 5>&-
```

___

##### Tool: [tcpdump](http://www.tcpdump.org/)

###### Filter incoming (on interface) traffic (specific <ip:port>)

```bash
tcpdump -ne -i eth0 -Q in host 192.168.252.1 and port 443
```

  * `-n` - don't convert addresses (`-nn` will not resolve hostnames or ports)
  * `-e` - print the link-level headers
  * `-i [iface|any]` - set interface
  * `-Q|-D [in|out|inout]` - choose send/receive direction (`-D` - for old tcpdump versions)
  * `host [ip|hostname]` - set host, also `[host not]`
  * `[and|or]` - set logic
  * `port [1-65535]` - set port number, also `[port not]`

###### Filter incoming (on interface) traffic (specific <ip:port>) and write to a file

```bash
tcpdump -ne -i eth0 -Q in host 192.168.252.1 and port 443 -c 5 -w tcpdump.pcap
```

  * `-c [num]` - capture only num number of packets
  * `-w [filename]` - write packets to file, `-r [filename]` - reading from file

###### Capture all ICMP packets

```bash
tcpdump -nei eth0 icmp
```

###### Check protocol used (TCP or UDP) for service

```bash
tcpdump -nei eth0 tcp port 22 -vv -X | egrep "TCP|UDP"
```

###### Display ASCII text (to parse the output using grep or other)

```bash
tcpdump -i eth0 -A -s0 port 443
```

###### Grab everything between two keywords

```bash
tcpdump -i eth0 port 80 -X | sed -n -e '/username/,/=ldap/ p'
```

###### Grab user and pass ever plain http

```bash
tcpdump -i eth0  port http -l -A | egrep -i \
'pass=|pwd=|log=|login=|user=|username=|pw=|passw=|passwd=|password=|pass:|user:|username:|password:|login:|pass |user ' \
--color=auto --line-buffered -B20
```

###### Extract HTTP User Agent from HTTP request header

```bash
tcpdump -ei eth0 -nn -A -s1500 -l | grep "User-Agent:"
```

###### Capture only HTTP GET and POST packets

```bash
tcpdump -ei eth0 -s 0 -A -vv \
'tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x47455420' or 'tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x504f5354'
```

or simply:

```bash
tcpdump -ei eth0 -s 0 -v -n -l | egrep -i "POST /|GET /|Host:"
```

###### Rotate capture files

```bash
tcpdump -ei eth0 -w /tmp/capture-%H.pcap -G 3600 -C 200
```

  * `-G <num>` - pcap will be created every `<num>` seconds
  * `-C <size>` - close the current pcap and open a new one if is larger than `<size>`

###### Top hosts by packets

```bash
tcpdump -ei enp0s25 -nnn -t -c 200 | cut -f 1,2,3,4 -d '.' | sort | uniq -c | sort -nr | head -n 20
```

###### Excludes any RFC 1918 private address

```bash
tcpdump -nei eth0 'not (src net (10 or 172.16/12 or 192.168/16) and dst net (10 or 172.16/12 or 192.168/16))'
```

___

##### Tool: [tcpick](http://tcpick.sourceforge.net/)

###### Analyse packets in real-time

```bash
while true ; do tcpick -a -C -r dump.pcap ; sleep 2 ; clear ; done
```

___

##### Tool: [ngrep](http://ngrep.sourceforge.net/usage.html)

```bash
ngrep -d eth0 "www.domain.com" port 443
```

  * `-d [iface|any]` - set interface
  * `[domain]` - set hostname
  * `port [1-65535]` - set port number

```bash
ngrep -d eth0 "www.domain.com" src host 10.240.20.2 and port 443
```

  * `(host [ip|hostname])` - filter by ip or hostname
  * `(port [1-65535])` - filter by port number

```bash
ngrep -d eth0 -qt -O ngrep.pcap "www.domain.com" port 443
```

  * `-q` - quiet mode (only payloads)
  * `-t` - added timestamps
  * `-O [filename]` - save output to file, `-I [filename]` - reading from file

```bash
ngrep -d eth0 -qt 'HTTP' 'tcp'
```

  * `HTTP` - show http headers
  * `tcp|udp` - set protocol
  * `[src|dst] host [ip|hostname]` - set direction for specific node

```bash
ngrep -l -q -d eth0 -i "User-Agent: curl*"
```

  * `-l` - stdout line buffered
  * `-i` - case-insensitive search

___

##### Tool: [hping3](http://www.hping.org/)

```bash
hping3 -V -p 80 -s 5050 <scan_type> www.google.com
```

  * `-V|--verbose` - verbose mode
  * `-p|--destport` - set destination port
  * `-s|--baseport` - set source port
  * `<scan_type>` - set scan type
    * `-F|--fin` - set FIN flag, port open if no reply
    * `-S|--syn` - set SYN flag
    * `-P|--push` - set PUSH flag
    * `-A|--ack` - set ACK flag (use when ping is blocked, RST response back if the port is open)
    * `-U|--urg` - set URG flag
    * `-Y|--ymas` - set Y unused flag (0x80 - nullscan), port open if no reply
    * `-M 0 -UPF` - set TCP sequence number and scan type (URG+PUSH+FIN), port open if no reply

```bash
hping3 -V -c 1 -1 -C 8 www.google.com
```

  * `-c [num]` - packet count
  * `-1` - set ICMP mode
  * `-C|--icmptype [icmp-num]` - set icmp type (default icmp-echo = 8)

```bash
hping3 -V -c 1000000 -d 120 -S -w 64 -p 80 --flood --rand-source <remote_host>
```

  * `--flood` - sent packets as fast as possible (don't show replies)
  * `--rand-source` - random source address mode
  * `-d --data` - data size
  * `-w|--win` - winsize (default 64)

___

##### Tool: [nmap](https://nmap.org/)

###### Ping scans the network

```bash
nmap -sP 192.168.0.0/24
```

###### Show only open ports

```bash
nmap -F --open 192.168.0.0/24
```

###### Full TCP port scan using with service version detection

```bash
nmap -p 1-65535 -sV -sS -T4 192.168.0.0/24
```

###### Nmap scan and pass output to Nikto

```bash
nmap -p80,443 192.168.0.0/24 -oG - | nikto.pl -h -
```

###### Recon specific ip:service with Nmap NSE scripts stack

```bash
# Set variables:
_hosts="192.168.250.10"
_ports="80,443"

# Set Nmap NSE scripts stack:
_nmap_nse_scripts="+dns-brute,\
                   +http-auth-finder,\
                   +http-chrono,\
                   +http-cookie-flags,\
                   +http-cors,\
                   +http-cross-domain-policy,\
                   +http-csrf,\
                   +http-dombased-xss,\
                   +http-enum,\
                   +http-errors,\
                   +http-git,\
                   +http-grep,\
                   +http-internal-ip-disclosure,\
                   +http-jsonp-detection,\
                   +http-malware-host,\
                   +http-methods,\
                   +http-passwd,\
                   +http-phpself-xss,\
                   +http-php-version,\
                   +http-robots.txt,\
                   +http-sitemap-generator,\
                   +http-shellshock,\
                   +http-stored-xss,\
                   +http-title,\
                   +http-unsafe-output-escaping,\
                   +http-useragent-tester,\
                   +http-vhosts,\
                   +http-waf-detect,\
                   +http-waf-fingerprint,\
                   +http-xssed,\
                   +traceroute-geolocation.nse,\
                   +ssl-enum-ciphers,\
                   +whois-domain,\
                   +whois-ip"

# Set Nmap NSE script params:
_nmap_nse_scripts_args="dns-brute.domain=${_hosts},http-cross-domain-policy.domain-lookup=true,"
_nmap_nse_scripts_args+="http-waf-detect.aggro,http-waf-detect.detectBodyChanges,"
_nmap_nse_scripts_args+="http-waf-fingerprint.intensive=1"

# Perform scan:
nmap --script="$_nmap_nse_scripts" --script-args="$_nmap_nse_scripts_args" -p "$_ports" "$_hosts"
```

___

##### Tool: [netcat](http://netcat.sourceforge.net/)

```bash
nc -kl 5000
```

  * `-l` - listen for an incoming connection
  * `-k` - listening after client has disconnected
  * `>filename.out` - save receive data to file (optional)

```bash
nc 192.168.0.1 5051 < filename.in
```

  * `< filename.in` - send data to remote host

```bash
nc -vz 10.240.30.3 5000
```

  * `-v` - verbose output
  * `-z` - scan for listening daemons

```bash
nc -vzu 10.240.30.3 1-65535
```

  * `-u` - scan only udp ports

###### Transfer data file (archive)

```bash
server> nc -l 5000 | tar xzvfp -
client> tar czvfp - /path/to/dir | nc 10.240.30.3 5000
```

###### Launch remote shell

```bash
# 1)
server> nc -l 5000 -e /bin/bash
client> nc 10.240.30.3 5000

# 2)
server> rm -f /tmp/f; mkfifo /tmp/f
server> cat /tmp/f | /bin/bash -i 2>&1 | nc -l 127.0.0.1 5000 > /tmp/f
client> nc 10.240.30.3 5000
```

###### Simple file server

```bash
while true ; do nc -l 5000 | tar -xvf - ; done
```

###### Simple minimal HTTP Server

```bash
while true ; do nc -l -p 1500 -c 'echo -e "HTTP/1.1 200 OK\n\n $(date)"' ; done
```

###### Simple HTTP Server

  > Restarts web server after each request - remove `while` condition for only single connection.

```bash
cat > index.html << __EOF__
<!doctype html>
    <head>
        <meta charset="utf-8">
        <meta http-equiv="X-UA-Compatible" content="IE=edge,chrome=1">
        <title></title>
        <meta name="description" content="">
        <meta name="viewport" content="width=device-width, initial-scale=1">
    </head>
    <body>

    <p>

      Hello! It's a site.

    </p>

    </body>
</html>
__EOF__
```

```bash
server> while : ; do \
(echo -ne "HTTP/1.1 200 OK\r\nContent-Length: $(wc -c <index.html)\r\n\r\n" ; cat index.html;) | \
nc -l -p 5000 \
; done
```

  * `-p` - port number

###### Simple HTTP Proxy (single connection)

```bash
#!/usr/bin/env bash

if [[ $# != 2 ]] ; then
  printf "%s\\n" \
         "usage: ./nc-proxy listen-port bk_host:bk_port"
fi

_listen_port="$1"
_bk_host=$(echo "$2" | cut -d ":" -f1)
_bk_port=$(echo "$2" | cut -d ":" -f2)

printf "  lport: %s\\nbk_host: %s\\nbk_port: %s\\n\\n" \
       "$_listen_port" "$_bk_host" "$_bk_port"

_tmp=$(mktemp -d)
_back="$_tmp/pipe.back"
_sent="$_tmp/pipe.sent"
_recv="$_tmp/pipe.recv"

trap 'rm -rf "$_tmp"' EXIT

mkfifo -m 0600 "$_back" "$_sent" "$_recv"

sed "s/^/=> /" <"$_sent" &
sed "s/^/<=  /" <"$_recv" &

nc -l -p "$_listen_port" <"$_back" | \
tee "$_sent" | \
nc "$_bk_host" "$_bk_port" | \
tee "$_recv" >"$_back"
```

```bash
server> chmod +x nc-proxy && ./nc-proxy 8080 192.168.252.10:8000
  lport: 8080
bk_host: 192.168.252.10
bk_port: 8000

client> http -p h 10.240.30.3:8080
HTTP/1.1 200 OK
Accept-Ranges: bytes
Cache-Control: max-age=31536000
Content-Length: 2748
Content-Type: text/html; charset=utf-8
Date: Sun, 01 Jul 2018 20:12:08 GMT
Last-Modified: Sun, 01 Apr 2018 21:53:37 GMT
```

###### Create a single-use TCP or UDP proxy

```bash
### TCP -> TCP
nc -l -p 2000 -c "nc [ip|hostname] 3000"

### TCP -> UDP
nc -l -p 2000 -c "nc -u [ip|hostname] 3000"

### UDP -> UDP
nc -l -u -p 2000 -c "nc -u [ip|hostname] 3000"

### UDP -> TCP
nc -l -u -p 2000 -c "nc [ip|hostname] 3000"
```

___

##### Tool: [gnutls-cli](https://gnutls.org/manual/html_node/gnutls_002dcli-Invocation.html)

###### Testing connection to remote host (with SNI support)

```bash
gnutls-cli -p 443 google.com
```

###### Testing connection to remote host (without SNI support)

```bash
gnutls-cli --disable-sni -p 443 google.com
```

___

##### Tool: [socat](http://www.dest-unreach.org/socat/doc/socat.html)

###### Testing remote connection to port

```bash
socat - TCP4:10.240.30.3:22
```

  * `-` - standard input (STDIO)
  * `TCP4:<params>` - set tcp4 connection with specific params
    * `[hostname|ip]` - set hostname/ip
    * `[1-65535]` - set port number

###### Redirecting TCP-traffic to a UNIX domain socket under Linux

```bash
socat TCP-LISTEN:1234,bind=127.0.0.1,reuseaddr,fork,su=nobody,range=127.0.0.0/8 UNIX-CLIENT:/tmp/foo
```

  * `TCP-LISTEN:<params>` - set tcp listen with specific params
    * `[1-65535]` - set port number
    * `bind=[hostname|ip]` - set bind hostname/ip
    * `reuseaddr` - allows other sockets to bind to an address
    * `fork` - keeps the parent process attempting to produce more connections
    * `su=nobody` - set user
    * `range=[ip-range]` - ip range
  * `UNIX-CLIENT:<params>` - communicates with the specified peer socket
    * `filename` - define socket

___

##### Tool: [p0f](http://lcamtuf.coredump.cx/p0f3/)

###### Set iface in promiscuous mode and dump traffic to the log file

```bash
p0f -i enp0s25 -p -d -o /dump/enp0s25.log
```

  * `-i` - listen on the specified interface
  * `-p` - set interface in promiscuous mode
  * `-d` - fork into background
  * `-o` - output file

___

##### Tool: [netstat](https://en.wikipedia.org/wiki/Netstat)

###### Graph # of connections for each hosts

```bash
netstat -an | awk '/ESTABLISHED/ { split($5,ip,":"); if (ip[1] !~ /^$/) print ip[1] }' | \
sort | uniq -c | awk '{ printf("%s\t%s\t",$2,$1) ; for (i = 0; i < $1; i++) {printf("*")}; print "" }'
```

###### Monitor open connections for specific port including listen, count and sort it per IP

```bash
watch "netstat -plan | grep :443 | awk {'print \$5'} | cut -d: -f 1 | sort | uniq -c | sort -nk 1"
```

###### Grab banners from local IPv4 listening ports

```bash
netstat -nlt | grep 'tcp ' | grep -Eo "[1-9][0-9]*" | xargs -I {} sh -c "echo "" | nc -v -n -w1 127.0.0.1 {}"
```

___

##### Tool: [rsync](https://en.wikipedia.org/wiki/Rsync)

###### Rsync remote data as root using sudo

```bash
rsync --rsync-path 'sudo rsync' username@hostname:/path/to/dir/ /local/
```

___

##### Tool: [host](https://en.wikipedia.org/wiki/Host_(Unix))

###### Resolves the domain name (using external dns server)

```bash
host google.com 9.9.9.9
```

###### Checks the domain administrator (SOA record)

```bash
host -t soa google.com 9.9.9.9
```

___

##### Tool: [dig](https://en.wikipedia.org/wiki/Dig_(command))

###### Resolves the domain name (short output)

```bash
dig google.com +short
```

###### Lookup NS record for specific domain

```bash
dig @9.9.9.9 google.com NS
```

###### Query only answer section

```bash
dig google.com +nocomments +noquestion +noauthority +noadditional +nostats
```

###### Query ALL DNS Records

```bash
dig google.com ANY +noall +answer
```

###### DNS Reverse Look-up

```bash
dig -x 172.217.16.14 +short
```

___

##### Tool: [certbot](https://certbot.eff.org/)

###### Generate multidomain certificate

```bash
certbot certonly -d example.com -d www.example.com
```

###### Generate wildcard certificate

```bash
certbot certonly --manual --preferred-challenges=dns -d example.com -d *.example.com
```

###### Generate certificate with 4096 bit private key

```bash
certbot certonly -d example.com -d www.example.com --rsa-key-size 4096
```

___

##### Tool: [network-other](https://github.com/trimstray/the-book-of-secret-knowledge#tool-network-other)

###### Get all subnets for specific AS (Autonomous system)

```bash
AS="AS32934"
whois -h whois.radb.net -- "-i origin ${AS}" | \
grep "^route:" | \
cut -d ":" -f2 | \
sed -e 's/^[ \t]//' | \
sort -n -t . -k 1,1 -k 2,2 -k 3,3 -k 4,4 | \
cut -d ":" -f2 | \
sed -e 's/^[ \t]/allow /' | \
sed 's/$/;/' | \
sed 's/allow  */subnet -> /g'
```

###### Resolves domain name from dns.google.com with curl and jq

```bash
_dname="google.com" ; curl -s "https://dns.google.com/resolve?name=${_dname}&type=A" | jq .
```

