<!-- Vendored from https://github.com/trimstray/the-book-of-secret-knowledge (MIT License). Snapshot: 2026-06-14. 本文件是该知识库按分类拆分出来的一部分，供 Argus secret-knowledge skill 按需 Grep 检索。原始版权归 trimstray 及贡献者所有，保留 MIT 许可。 -->

##### Tool: [openssl](https://www.openssl.org/)

###### Testing connection to the remote host

```bash
echo | openssl s_client -connect google.com:443 -showcerts
```

###### Testing connection to the remote host (debug mode)

```bash
echo | openssl s_client -connect google.com:443 -showcerts -tlsextdebug -status
```

###### Testing connection to the remote host (with SNI support)

```bash
echo | openssl s_client -showcerts -servername google.com -connect google.com:443
```

###### Testing connection to the remote host with specific ssl version

```bash
openssl s_client -tls1_2 -connect google.com:443
```

###### Testing connection to the remote host with specific ssl cipher

```bash
openssl s_client -cipher 'AES128-SHA' -connect google.com:443
```

###### Verify 0-RTT

```bash
_host="example.com"

cat > req.in << __EOF__
HEAD / HTTP/1.1
Host: $_host
Connection: close
__EOF__

openssl s_client -connect ${_host}:443 -tls1_3 -sess_out session.pem -ign_eof < req.in
openssl s_client -connect ${_host}:443 -tls1_3 -sess_in session.pem -early_data req.in
```

###### Generate private key without passphrase

```bash
# _len: 2048, 4096
( _fd="private.key" ; _len="2048" ; \
openssl genrsa -out ${_fd} ${_len} )
```

###### Generate private key with passphrase

```bash
# _ciph: aes128, aes256
# _len: 2048, 4096
( _ciph="aes128" ; _fd="private.key" ; _len="2048" ; \
openssl genrsa -${_ciph} -out ${_fd} ${_len} )
```

###### Remove passphrase from private key

```bash
( _fd="private.key" ; _fd_unp="private_unp.key" ; \
openssl rsa -in ${_fd} -out ${_fd_unp} )
```

###### Encrypt existing private key with a passphrase

```bash
# _ciph: aes128, aes256
( _ciph="aes128" ; _fd="private.key" ; _fd_pass="private_pass.key" ; \
openssl rsa -${_ciph} -in ${_fd} -out ${_fd_pass}
```

###### Check private key

```bash
( _fd="private.key" ; \
openssl rsa -check -in ${_fd} )
```

###### Get public key from private key

```bash
( _fd="private.key" ; _fd_pub="public.key" ; \
openssl rsa -pubout -in ${_fd} -out ${_fd_pub} )
```

###### Generate private key and CSR

```bash
( _fd="private.key" ; _fd_csr="request.csr" ; _len="2048" ; \
openssl req -out ${_fd_csr} -new -newkey rsa:${_len} -nodes -keyout ${_fd} )
```

###### Generate CSR

```bash
( _fd="private.key" ; _fd_csr="request.csr" ; \
openssl req -out ${_fd_csr} -new -key ${_fd} )
```

###### Generate CSR (metadata from existing certificate)

  > Where `private.key` is the existing private key. As you can see you do not generate this CSR from your certificate (public key). Also you do not generate the "same" CSR, just a new one to request a new certificate.

```bash
( _fd="private.key" ; _fd_csr="request.csr" ; _fd_crt="cert.crt" ; \
openssl x509 -x509toreq -in ${_fd_crt} -out ${_fd_csr} -signkey ${_fd} )
```

###### Generate CSR with -config param

```bash
( _fd="private.key" ; _fd_csr="request.csr" ; \
openssl req -new -sha256 -key ${_fd} -out ${_fd_csr} \
-config <(
cat << __EOF__
[req]
default_bits        = 2048
default_md          = sha256
prompt              = no
distinguished_name  = dn
req_extensions      = req_ext

[ dn ]
C   = "<two-letter ISO abbreviation for your country>"
ST  = "<state or province where your organisation is legally located>"
L   = "<city where your organisation is legally located>"
O   = "<legal name of your organisation>"
OU  = "<section of the organisation>"
CN  = "<fully qualified domain name>"

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = <fully qualified domain name>
DNS.2 = <next domain>
DNS.3 = <next domain>
__EOF__
))
```

Other values in `[ dn ]`:

```
countryName            = "DE"                     # C=
stateOrProvinceName    = "Hessen"                 # ST=
localityName           = "Keller"                 # L=
postalCode             = "424242"                 # L/postalcode=
postalAddress          = "Keller"                 # L/postaladdress=
streetAddress          = "Crater 1621"            # L/street=
organizationName       = "apfelboymschule"        # O=
organizationalUnitName = "IT Department"          # OU=
commonName             = "example.com"            # CN=
emailAddress           = "webmaster@example.com"  # CN/emailAddress=
```

Example of `oids` (you'll probably also have to make OpenSSL know about the new fields required for EV by adding the following under `[new_oids]`):

```
[req]
...
oid_section         = new_oids

[ new_oids ]
postalCode = 2.5.4.17
streetAddress = 2.5.4.9
```

Full example:

```bash
( _fd="private.key" ; _fd_csr="request.csr" ; \
openssl req -new -sha256 -key ${_fd} -out ${_fd_csr} \
-config <(
cat << __EOF__
[req]
default_bits        = 2048
default_md          = sha256
prompt              = no
distinguished_name  = dn
req_extensions      = req_ext
oid_section         = new_oids

[ new_oids ]
serialNumber = 2.5.4.5
streetAddress = 2.5.4.9
postalCode = 2.5.4.17
businessCategory = 2.5.4.15

[ dn ]
serialNumber=00001111
businessCategory=Private Organization
jurisdictionC=DE
C=DE
ST=Hessen
L=Keller
postalCode=424242
streetAddress=Crater 1621
O=AV Company
OU=IT
CN=example.com

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = example.com
__EOF__
))
```

For more information please look at these great explanations:

- [RFC 5280](https://tools.ietf.org/html/rfc5280)
- [How to create multidomain certificates using config files](https://apfelboymchen.net/gnu/notes/openssl%20multidomain%20with%20config%20files.html)
- [Generate a multi domains certificate using config files](https://gist.github.com/romainnorberg/464758a6620228b977212a3cf20c3e08)
- [Your OpenSSL CSR command is out of date](https://expeditedsecurity.com/blog/openssl-csr-command/)
- [OpenSSL example configuration file](https://www.tbs-certificats.com/openssl-dem-server-cert.cnf)
- [Object Identifiers (OIDs)](https://www.alvestrand.no/objectid/)
- [openssl objects.txt](https://github.com/openssl/openssl/blob/master/crypto/objects/objects.txt)

###### List available EC curves

```bash
openssl ecparam -list_curves
```

###### Print ECDSA private and public keys

```bash
( _fd="private.key" ; \
openssl ec -in ${_fd} -noout -text )

# For x25519 only extracting public key
( _fd="private.key" ; _fd_pub="public.key" ; \
openssl pkey -in ${_fd} -pubout -out ${_fd_pub} )
```

###### Generate ECDSA private key

```bash
# _curve: prime256v1, secp521r1, secp384r1
( _fd="private.key" ; _curve="prime256v1" ; \
openssl ecparam -out ${_fd} -name ${_curve} -genkey )

# _curve: X25519
( _fd="private.key" ; _curve="x25519" ; \
openssl genpkey -algorithm ${_curve} -out ${_fd} )
```

###### Generate private key and CSR (ECC)

```bash
# _curve: prime256v1, secp521r1, secp384r1
( _fd="domain.com.key" ; _fd_csr="domain.com.csr" ; _curve="prime256v1" ; \
openssl ecparam -out ${_fd} -name ${_curve} -genkey ; \
openssl req -new -key ${_fd} -out ${_fd_csr} -sha256 )
```

###### Generate self-signed certificate

```bash
# _len: 2048, 4096
( _fd="domain.key" ; _fd_out="domain.crt" ; _len="2048" ; _days="365" ; \
openssl req -newkey rsa:${_len} -nodes \
-keyout ${_fd} -x509 -days ${_days} -out ${_fd_out} )
```

###### Generate self-signed certificate from existing private key

```bash
# _len: 2048, 4096
( _fd="domain.key" ; _fd_out="domain.crt" ; _days="365" ; \
openssl req -key ${_fd} -nodes \
-x509 -days ${_days} -out ${_fd_out} )
```

###### Generate self-signed certificate from existing private key and csr

```bash
# _len: 2048, 4096
( _fd="domain.key" ; _fd_csr="domain.csr" ; _fd_out="domain.crt" ; _days="365" ; \
openssl x509 -signkey ${_fd} -nodes \
-in ${_fd_csr} -req -days ${_days} -out ${_fd_out} )
```

###### Generate DH public parameters

```bash
( _dh_size="2048" ; \
openssl dhparam -out /etc/nginx/ssl/dhparam_${_dh_size}.pem "$_dh_size" )
```

###### Display DH public parameters

```bash
openssl pkeyparam -in dhparam.pem -text
```

###### Extract private key from pfx

```bash
( _fd_pfx="cert.pfx" ; _fd_key="key.pem" ; \
openssl pkcs12 -in ${_fd_pfx} -nocerts -nodes -out ${_fd_key} )
```

###### Extract private key and certs from pfx

```bash
( _fd_pfx="cert.pfx" ; _fd_pem="key_certs.pem" ; \
openssl pkcs12 -in ${_fd_pfx} -nodes -out ${_fd_pem} )
```

###### Extract certs from p7b

```bash
# PKCS#7 file doesn't include private keys.
( _fd_p7b="cert.p7b" ; _fd_pem="cert.pem" ; \
openssl pkcs7 -inform DER -outform PEM -in ${_fd_p7b} -print_certs > ${_fd_pem})
# or:
openssl pkcs7 -print_certs -in -in ${_fd_p7b} -out ${_fd_pem})
```

###### Convert DER to PEM

```bash
( _fd_der="cert.crt" ; _fd_pem="cert.pem" ; \
openssl x509 -in ${_fd_der} -inform der -outform pem -out ${_fd_pem} )
```

###### Convert PEM to DER

```bash
( _fd_der="cert.crt" ; _fd_pem="cert.pem" ; \
openssl x509 -in ${_fd_pem} -outform der -out ${_fd_der} )
```

###### Verification of the private key

```bash
( _fd="private.key" ; \
openssl rsa -noout -text -in ${_fd} )
```

###### Verification of the public key

```bash
# 1)
( _fd="public.key" ; \
openssl pkey -noout -text -pubin -in ${_fd} )

# 2)
( _fd="private.key" ; \
openssl rsa -inform PEM -noout -in ${_fd} &> /dev/null ; \
if [ $? = 0 ] ; then echo -en "OK\n" ; fi )
```

###### Verification of the certificate

```bash
( _fd="certificate.crt" ; # format: pem, cer, crt \
openssl x509 -noout -text -in ${_fd} )
```

###### Verification of the CSR

```bash
( _fd_csr="request.csr" ; \
openssl req -text -noout -in ${_fd_csr} )
```

###### Check the private key and the certificate are match

```bash
(openssl rsa -noout -modulus -in private.key | openssl md5 ; \
openssl x509 -noout -modulus -in certificate.crt | openssl md5) | uniq
```

###### Check the private key and the CSR are match

```bash
(openssl rsa -noout -modulus -in private.key | openssl md5 ; \
openssl req -noout -modulus -in request.csr | openssl md5) | uniq
```

___

##### Tool: [secure-delete](https://wiki.archlinux.org/index.php/Securely_wipe_disk)

###### Secure delete with shred

```bash
shred -vfuz -n 10 file
shred --verbose --random-source=/dev/urandom -n 1 /dev/sda
```

###### Secure delete with scrub

```bash
scrub -p dod /dev/sda
scrub -p dod -r file
```

###### Secure delete with badblocks

```bash
badblocks -s -w -t random -v /dev/sda
badblocks -c 10240 -s -w -t random -v /dev/sda
```

###### Secure delete with secure-delete

```bash
srm -vz /tmp/file
sfill -vz /local
sdmem -v
swapoff /dev/sda5 && sswap -vz /dev/sda5
```

___

##### Tool: [dd](https://en.wikipedia.org/wiki/Dd_(Unix))

###### Show dd status every so often

```bash
dd <dd_params> status=progress
watch --interval 5 killall -USR1 dd
```

###### Redirect output to a file with dd

```bash
echo "string" | dd of=filename
```

___

##### Tool: [gpg](https://www.gnupg.org/)

###### Export public key

```bash
gpg --export --armor "<username>" > username.pkey
```

  * `--export` - export all keys from all keyrings or specific key
  * `-a|--armor` - create ASCII armored output

###### Encrypt file

```bash
gpg -e -r "<username>" dump.sql
```

  * `-e|--encrypt` - encrypt data
  * `-r|--recipient` - encrypt for specific <username>

###### Decrypt file

```bash
gpg -o dump.sql -d dump.sql.gpg
```

  * `-o|--output` - use as output file
  * `-d|--decrypt` - decrypt data (default)

###### Search recipient

```bash
gpg --keyserver hkp://keyserver.ubuntu.com --search-keys "<username>"
```

  * `--keyserver` - set specific key server
  * `--search-keys` - search for keys on a key server

###### List all of the packets in an encrypted file

```bash
gpg --batch --list-packets archive.gpg
gpg2 --batch --list-packets archive.gpg
```

___

##### Tool: [system-other](https://github.com/trimstray/the-book-of-secret-knowledge#tool-system-other)

###### Reboot system from init

```bash
exec /sbin/init 6
```

###### Init system from single user mode

```bash
exec /sbin/init
```

###### Show current working directory of a process

```bash
readlink -f /proc/<PID>/cwd
```

###### Show actual pathname of the executed command

```bash
readlink -f /proc/<PID>/exe
```

