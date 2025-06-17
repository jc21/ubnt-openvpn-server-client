# Setup a OpenVPN Server

> [!WARNING]
> Follow these steps in order!

Log in to the router and get a root shell:

```bash
ssh ubnt@10.0.0.1
sudo su
```

Just in case you've tried this before, it's best
to clean some things up so you can start fresh:

```bash
rm -rf /config/auth
mkdir -p /config/auth
```

> [!TIP]
> The router comes with a perl script to help create certificates and stuff.
> However, this script is a little wrong; it doesn't handle custom expiry
> dates on certificates properly. I recommend downloading my edited script
> [here](src/CA.pl) and uploading it to your router first.
> ```bash
> scp CA.pl ubnt@10.0.0.1:~
> ```
> then move it to the correct place and making a backup of the original:
> ```bash
> ssh ubnt@10.0.0.1
> sudo su
> mv /usr/lib/ssl/misc/CA.pl /usr/lib/ssl/misc/CA.pl.original
> mv ~ubnt/CA.pl /usr/lib/ssl/misc/CA.pl
> chown root:root /usr/lib/ssl/misc/CA.pl
> chmod +x /usr/lib/ssl/misc/CA.pl
> ```

Change to the working directory and do more cleanup:

```bash
cd /usr/lib/ssl/misc
rm -rf demoCA *.key *.pem
```

Edit the `CA.pl` helper script and modify the expiry days number to suit
your needs:

```bash
vi CA.pl
```


### Generate a Diffie-Hellman (DH) file

You need to do this to ensure your router has unique encryption. It will take a LONG time.
On some routers it has taken more than 45 minutes! But it _needs_ to happen.

```bash
openssl dhparam -out /config/auth/dh.pem -2 2048
```


### Generate a ROOT certificate

You're about to run a command and it will ask you for some info. Here's my advice:

- You must enter and remember a passphrase. You'll need this to generate new SERVER and CLIENT certs.
- The `Common Name` is required and should be unique across all your certificates. I use `ubnt` for this value.
- All other values can be default or emptied or filled in if you care to do so.

```bash
./CA.pl -newca
```

Copy the certs to their final destination:

```bash
cp demoCA/cacert.pem /config/auth/
cp demoCA/private/cakey.pem /config/auth/
```

Optional step: Check the certificate and ensure the details are correct:

```bash
openssl x509 -in /config/auth/cacert.pem -text -noout
```


### Generate a SERVER certificate

You're about to run more commands. Here's my advice for this one:

- You must enter and remember a passphrase. It's temporary though, we'll remove the passphrase later.
- The `Common Name` is required and should be unique across all your certificates. I use `jc21.com` for this value.
- All other values can be default or emptied or filled in if you care to do so.

```bash
./CA.pl -newreq
./CA.pl -sign
```

Move the certs to their final destination:

```bash
mv newcert.pem /config/auth/server.pem
mv newkey.pem /config/auth/server.key
```

Optional step: Check the certificate and ensure the details are correct:

```bash
openssl x509 -in /config/auth/server.pem -text -noout
```


### Generate a CLIENT certificate

This is the same as the SERVER commands, except:
- The `Common Name` is required and should be unique across all your certificates. I use my full name
  or the name of the end user or machine for this one.

```bash
./CA.pl -newreq
./CA.pl -sign
```

Move the certs to their final destination:

```bash
mv newcert.pem /config/auth/client1.pem
mv newkey.pem /config/auth/client1.key
```

Optional step: Check the certificate and ensure the details are correct:

```bash
openssl x509 -in /config/auth/client1.pem -text -noout
```

**Repeat this step for as many clients as you want, save them as `client2`, `client3` etc**


### Remove passphrases

```bash
openssl rsa -in /config/auth/server.key -out /config/auth/server-no-pass.key
openssl rsa -in /config/auth/client1.key -out /config/auth/client1-no-pass.key
```

Repeat this last line for all the other client certificates you may have created.

Overwrite the protected files with the passphrase removed ones:

```bash
mv /config/auth/server-no-pass.key /config/auth/server.key
mv /config/auth/client1-no-pass.key /config/auth/client1.key
```

## Configure the Edgerouter Software

Now that the certificates are ready, it's time to tell the router to use them
and create a OpenVPN Server.

> [!WARNING]
> Read these commands carefully and make sure you understand them. I recommend
> backing up your configuration using the web interface first!

> [!IMPORTANT]
> If your router's IP is not `10.0.0.1` then pay attention to replace the ips
> and ranges in the commands below. If your IP is `192.168.0.1` then the
> `push-route` would be `192.168.0.0/24` and the `name-server` would be
> `192.168.0.1`

> [!IMPORTANT]
> If you already have a `vtun0` interface and you don't want to replace it,
> consider using `vtun1`

```bash
ssh ubnt@10.0.0.1
configure

set firewall name WAN_LOCAL rule 30 action accept
set firewall name WAN_LOCAL rule 30 description openvpn
set firewall name WAN_LOCAL rule 30 destination port 1194
set firewall name WAN_LOCAL rule 30 protocol udp

set interfaces openvpn vtun0 mode server
set interfaces openvpn vtun0 server subnet 172.16.1.0/24
set interfaces openvpn vtun0 server push-route 10.0.0.0/24
set interfaces openvpn vtun0 server name-server 10.0.0.1
set interfaces openvpn vtun0 tls ca-cert-file /config/auth/cacert.pem
set interfaces openvpn vtun0 tls cert-file /config/auth/server.pem
set interfaces openvpn vtun0 tls key-file /config/auth/server.key
set interfaces openvpn vtun0 tls dh-file /config/auth/dh.pem

set service dns forwarding listen-on vtun0

commit
save
exit
reboot
```

## Configure your OpenVPN Client

Grab the client files from the router to your computer:

```bash
scp ubnt@10.0.0.1:/config/auth/cacert.pem .
scp ubnt@10.0.0.1:/config/auth/client1.pem .
scp ubnt@10.0.0.1:/config/auth/client1.key .
```

Create a `.ovpn` file with the following content:

```
client
dev tun
proto udp
remote vpn.yourpublicdomain-or-ipaddress 1194
float
resolv-retry infinite
nobind
persist-key
persist-tun
verb 3
ca cacert.pem
cert client1.pem
key client1.key
# optional: set this to send all traffic over vpn
# redirect-gateway def1
```

Import that file and accompanying certificates to your favourite OpenVPN client.
