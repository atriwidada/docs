## Optional Configurations

The Greenbone Community Edition on Kali Linux installation relies on the same sub-system components as the [source code installation](/22.4/source-build/index.md) and all configuration options are available. Let's cover some common custom configurations.

### Configure Remote Access To The Web Interface

By default Greenbone Community Edition is installed with only `localhost` access to the {term}`GSA` web interface. This means Greenbone Community Edition can only be accessed via the IP address `127.0.0.1`. To enable remote access to the web interface, the {term}`gsad` systemd service file must be modified and the gsad service must be restarted.


Edit the contents of the `gsad.service` systemd service file:

```{code-block}
:caption: Use vi to edit the gsad.sevice file
vi /usr/lib/systemd/system/gsad.service
```

Change the value of the `--listen` argument to `0.0.0.0` and optionally change the value of `--port` to the standard SSL/TLS port 443:

```diff
-ExecStart=/usr/sbin/gsad --foreground --listen=127.0.0.1 --port=9392
+ExecStart=/usr/sbin/gsad --foreground --listen=0.0.0.0 --port=443
```

To listen on both IPv4 and IPv6, change the `--listen` argument to `::`

```diff
-ExecStart=/usr/sbin/gsad --foreground --listen=127.0.0.1 --port=9392
+ExecStart=/usr/sbin/gsad --foreground --listen :: --port 443
```

Restart the `gsad` service:
```{code-block}
sudo systemctl daemon-reload
sudo systemctl restart gsad
```

### Setting A Password Policy

The password policy configuration file defines the rules for user passwords such as minimum length, complexity, and expiration period, ensuring that all user passwords adhere to the desired security standards.

```{code-block}
:caption: Edit the Greenbone Community Edition password policy configuration
vi /etc/gvm/pwpolicy.conf
```

### Use Let's Encrypt TLS for Greenbone Security Assistant
1. Install `certbot` package
```{code-block}
sudo apt install certbot -y
```
2. Ensure your server has a public IP and its entry on your DNS was setup properly. Let's assume we will use greenbone.mydomain.tld as our server's FQDN.
3. Setup TLS certificate using `certbot`. You need to shutdown `gsad` daemon first if it already listening on port 80 or 443.
```{code-block}
# optional:
# sudo systemctl stop gsad
sudo certtbot certonly --standalone -d greenbone.mydomain.tld --verbose
```
4. Copy generated private key and certificate to an accessible directory.
```{code-block}
sudo cp /etc/letsencrypt/live/greenbone.mydomain.tld/privkey.pem /var/lib/gvm/private/CA/le-privkey.pem
sudo cp /etc/letsencrypt/live/greenbone.mydomain.tld/fullchain.pem /var/lib/gvm/private/CA/le-fullchain.pem
sudo chown _gvm:_gvm /var/lib/gvm/private/CA/le-{fullchain,privkey}.pem
```
5. Change systemd service file to accomodate this.
```diff
-ExecStart=/usr/sbin/gsad --foreground --listen=127.0.0.1 --port=9392
+ExecStart=/usr/sbin/gsad --foreground --listen :: --port 443 --ssl-private-key=/var/lib/gvm/private/CA/le-privkey.pem --ssl-certificate=/var/lib/gvm/private/CA/le-fullchain.pem 
```
6. Test access, browse to [greenbone.mydomain.tld](https://greenbone.mydomain.tld/)
7. Periodic TLS certificate renewal. Need to be done every 90 days. Check /var/log/gvm/gsad.log if you encounter any problem.
```{code-block}
# shutdown gsad
sudo systemctl stop gsad
# run certbot renewal
sudo certtbot renew --standalone -d greenbone.mydomain.tld --verbose
# copy new certificate
sudo cp /etc/letsencrypt/live/greenbone.mydomain.tld/fullchain.pem /var/lib/gvm/private/CA/le-fullchain.pem
sudo chown _gvm:_gvm /var/lib/gvm/private/CA/le-fullchain.pem
# start gsad again
sudo systemctl start gsad
```
