
## Install QUAY registry in RHEL 9

Set IP address and Hostname
```bash
# nmcli conn add con-name test ifname ens3 type ethernet ipv4.address 192.168.1.170 ipv4.dns 8.8.8.8 ipv4.method manual autoconnect yes ipv4.gateway 192.168.1.1
# hostnamectl set-hostname quay-server.example.com
```
Register the system, Install podman and login to registry.redhat.io

```bash
  # subscription-manager register
  # yum install -y podman
  # podman login registry.redhat.io
```

Add ports to firewall


```bash
# firewall-cmd --permanent --add-port=80/tcp \
&& firewall-cmd --permanent --add-port=443/tcp \
&& firewall-cmd --permanent --add-port=5432/tcp \
&& firewall-cmd --permanent --add-port=5433/tcp \
&& firewall-cmd --permanent --add-port=6379/tcp \
&& firewall-cmd --reload
```

Make an /etc/hosts entry
```bash
# echo 192.168.1.170 quay-server.example.com >> /etc/hosts
```


Configure Databse


```bash
# mkdir -p $QUAY/postgres-quay
# setfacl -m u:26:-wx $QUAY/postgres-quay

# podman run -d --rm --name postgresql-quay \
  -e POSTGRESQL_USER=quayuser \
  -e POSTGRESQL_PASSWORD=quaypass \
  -e POSTGRESQL_DATABASE=quay \
  -e POSTGRESQL_ADMIN_PASSWORD=adminpass \
  -p 5432:5432 \
  -v $QUAY/postgres-quay:/var/lib/pgsql/data:Z \
  registry.redhat.io/rhel8/postgresql-13:1-109

# podman exec -it postgresql-quay /bin/bash -c 'echo "CREATE EXTENSION IF NOT EXISTS pg_trgm" | psql -d quay -U postgres'
```

Configure Redis

```bash
# podman run -d --rm --name redis \
  -p 6379:6379 \
  -e REDIS_PASSWORD=strongpassword \
  registry.redhat.io/rhel8/redis-6:1-110
```

Run the quay config container

```bash
# podman run --rm -it --name quay -p 80:8080 -p 443:8443 registry.redhat.io/quay/quay-rhel8:v3.10.3 config secret
```

Now open the browser with http://quay-server.example.com and login with `quayconfig` user and `secret` password. Provide all the necessary details and then download the quay-config.tar.gz file

Stop the quay config container

```bash
# podman stop quay
```
Create the quay container configuration.

```bash
# mkdir $QUAY/config
# cp quay-config.tar.gz $QUAY/config
# cd $QUAY/config
# tar -xvf quay-config.tar.gz
# mkdir $QUAY/storage
# setfacl -m u:1001:-wx $QUAY/storage

```

Now start the actual QUAY container.

```bash
# podman run -d --rm -p 80:8080 -p 443:8443  \
   --name=quay \
   -v $QUAY/config:/conf/stack:Z \
   -v $QUAY/storage:/datastorage:Z \
   registry.redhat.io/quay/quay-rhel8:v3.10.5
```

To configure TLS in quay registry follow below steps.

=> First create the key and cert file. Once ssl.crt and ssl.key are created then follow below.

```bash
# cp ssl.* $QUAY/config
# cd $QUAY/config
# find . -type f -exec setfacl -m user:1001:rw {} \;
# find . -type d -exec setfacl -m user:1001:rwx {} \;

```

Edit the config.yaml file and change following line

```bash
# cd $QUAY/config
# vim config.yaml
PREFERRED_URL_SCHEME: https
```

Restart the quay container to reflect the config changes.


```bash
# podman stop quay
# podman run -d --rm -p 80:8080 -p 443:8443 \
  --name=quay \
  -v $QUAY/config:/conf/stack:Z \
  -v $QUAY/storage:/datastorage:Z \
  registry.redhat.io/quay/quay-rhel8:v3.10.5

```

Try to access the registry, it will be failing with TLS error as we have self signed certs configured.

```bash
# podman login quay-server.example.com
Username: testuser
Password: *******

tls: failed to verify certificate: x509: certificate signed by unknown authority
```

Try login with --tls-verify=false, it should work

```bash
# podman login quay-server.example.com --tls-verify=false
Username: testuser
Password: *******
Login Succeeded!

# podman logout quay-server.example.com
```
Now add the root-ca bundle to the certs so that we can login to registry with TLS.

```bash
# cp rootCA.pem /etc/containers/certs.d/quay-server.example.com/ca.crt

# podman login quay-server.example.com 
Username: testuser
Password: *******
Login Succeeded!

```


## Official OCP Documentation Links

[Proof of Concept - Deploying Red Hat Quay](https://access.redhat.com/documentation/en-us/red_hat_quay/3.10/html-single/proof_of_concept_-_deploying_red_hat_quay/index)

## 🛠 Skills
QUAY, Linux, Openssl


## Authors

- [@kaybee-singh](https://www.github.com/kaybee-singh)
