
# elasticsearch certificate update 

Updating the certificates for your Elasticsearch cluster is a critical maintenance task that ensures the continued security and integrity of your data. This guide will walk you through the essential steps involved in this process, aiming to minimize disruption and maintain the stability of your Elasticsearch environment. We'll cover everything from preparing your new certificates and securely distributing them across your nodes to updating the Elasticsearch configuration and performing a rolling restart. By following these instructions carefully, you can confidently update your Elasticsearch certificates and keep your cluster secure. Let's get started!

Befor Start please read full documentation of [Scuring Elasticserch Cluster](https://www.elastic.co/docs/deploy-manage/security/secure-your-cluster-deployment).

## Checking the expiration date of the current certificate.

In the first step, we need to check how much validity remains on our current certificate. We will use the following command for this purpose. Please note that all our processes are based on certificates in the `.p12` format. At this stage, all we need is the current SSL certificate file and its password.

#

> :warning: NOTE
> 
> Before starting, please review the documentation related to securing your Elasticsearch cluster and create a backup of your ELK stack configurations

```shell
openssl pkcs12 -in /path/to/elasticsearch/elastic-certificates.p12 -nodes  | openssl x509 -noout -enddate
```

> You can also use the following command in Kibana's Dev Tools to obtain the expiration time of your certificate.

```http request
GET _ssl/certificates
```

You can also use the following commands for Kibana to find the certificate's expiration time.
 
```shell
openssl x509 -in /path/to/kibana/certs/certificate.cer.pem -noout -enddate
openssl x509 -in /path/to/kibana/certs/rootCA-certificate.cer.pem -noout -enddate
```
this certificates used for transport layer between nodes and http layer 

If your Certificate Authority (CA) is about to expire, you can still use its private key file to generate a new CA certificate. In this case, this approach will be used. (Keeping Same)

#

:toolbox: Step-by-Step Guide (Using `openssl` and `elasticsearch-certutil`)

**🔓 1.Extract CA key and Cert From the `.p12`**

```shell
#  Extract the CA private key
openssl pkcs12 -in /path/to/elastic-stack-ca.p12 -nocerts -out rootCA.key -nodes

# Extract the CA certificate
openssl pkcs12 -in /path/to/elastic-stack-ca.p12 -clcerts -nokeys -out ca.crt
```
You'll be prompted for the .p12 password.

#

**🔄 2. Re-issue the CA Certificate with Extended Validity**

Now re-use the extracted `ca.key` to issue a new CA cert (valid for e.g., 5 more years):

```shell
openssl req -x509 -new -nodes -key /path/to/rootCA.key -sha256 -days 1825 -out /path/to/new/rootCA.crt -subj "/CN=Elastic Stack CA"
```
You can use -days as needed (e.g., 3650 = 10 years).

> ⚠️ Note
> 
> also you can verify expireation date of new root Ca Certificate File

```shell
openssl x509 -in /path/to/new/rootCA.crt -noout -dates
```

#

**🏗 3. (Optional) Rebuild a New .p12 if You Need to Repackage It**

If you want to repackage the new cert + existing key into a new `.p12`:

```shell
openssl pkcs12 -export -inkey /path/to/rootCA.key -in /path/to/new/rootCA.crt -out /path/to/new/elastic-stack-ca.p12 -name "elastic-stack-ca"
```

> ⚠️ Note
>
> Similar to the previous case, we can also check the validity period of the new SSL certificate here as follows:

```shell
openssl pkcs12 -in /path/to/new/elastic-stack-ca.p12 -nokeys | openssl x509 -noout -enddate
```

## Reissue Node/HTTP/Client Certs

🛠️ Step-by-Step: Reissue Certs Using `instance.yml` + `New CA` + `elasticsearch-certutil` 

📁 Example instance.yml
Here’s what your instance.yml might look like:

> ⚠️ NOTE 
> 
> Do this method for create both http and transport layer cert bases of this example

```yaml
instances:
  - name: es-node-1
    dns:
      - es-node-1.example.com
    ip:
      - 192.168.1.10
```

#

**✅ Step 1: Place your CA cert & key (or .p12) in one folder**
You must have either:
* `ca-cert.pem` and `ca.key` (PEM format), or
* `elastic-stack-ca.p12` (PKCS#12 bundle)

#

**✅ Step 2: Run the command to generate signed certs**

```
# If you use .p12 bundle:

elasticsearch-certutil cert --silent -in /path/to/instances.yml --out /path/to/output/elastic-certificates.zip --pass `password` ./path/to/elastic-stack-ca.p12  --ca-pass `password`

# If you use PEM files:

elasticsearch-certutil cert \
  --ca-cert ca-cert.pem \
  --ca-key ca.key \
  --in instance.yml \
  --out certs.zip \
  --pem

```
After this step you have two file `http-cert.zip` and `transport-cert.zip`

#

**✅ Step 3: Unzip the output and distribute the certs**

```shell
unzip http-cert.zip -d ./
unzip transport-cert.zip -d ./
```

also we can check Certificates  expire date

```shell
openssl pkcs12 -in /path/to/transport-cert.p12 -nokeys | openssl x509 -noout -enddate
openssl pkcs12 -in /path/to/http-cert.p12 -nokeys | openssl x509 -noout -enddate
```

Then Extract certificate & Key & chain file of http cert file for kibana 

```shell

openssl pkcs12 -in /path/to/http-cert.p12 -password pass:`password` -nocerts -nodes > /path/to/ext-cert/client.key
openssl pkcs12 -in /path/to/http-cert.p12 -password pass:`password` -clcerts -nokeys > /path/to/ext-cert/client.cer
openssl pkcs12 -in /path/to/http-cert.p12 -password pass:`password` -clcerts -nokeys -chain > /path/to/ext-cert/client-ca.cer
```

after done process we talk about this commands 

**✅ Step 4: replace new certificate files into elasticsearch | Kibana | logstash**

```shell
cd /path/to/elasticsearch/
ls -ltrh
cp /path/to/new/transport-cert/transport-cert.p12 /path/to/etc/elasticsearch/
cp /path/to/http-cert/http-cert.p12 /path/to/elasticsearch/http-cert.p12
chown elasticsearch:elasticsearch ./transport-cert.p12
chown elasticsearch:elasticsearch ./http-cert.p12
chmod 400 ./transport-cert.p12
chmod 400 ./http-cert.p12
cd /etc/kibana/
ls -ltrh
cp ./rootCA.crt /etc/kibana/
ls -ltrh
openssl x509 -in rootCA.crt -noout -dates
chmod 755 /etc/kibana/rootCA.crt
ls -ltrh
cd /etc/kibana/certs
rm *
cp output/node1/ext-cert/client-ca.cer /etc/kibana/certs/
cp output/node1/ext-cert/client.cer /etc/kibana/certs/
cp output/node1/ext-cert/client.key /etc/kibana/certs/
cp ./rootCA.crt ./output/logstash/client-ca.cer
scp client-ca.cer root@logstash-machine-ip:/etc/logstash/certs/
scp client-ca.cer root@logstash-machine-ip:/etc/logstash/certs/
openssl x509 -in /etc/logstash/certs/client-ca.cer -noout -dates


service elasticsearch restart
service cerebro restart
service kibana restart
service logstash restart

curl -XGET --insecure --user es-user 'https://es-machine-ip-or-domain:9200/_cat/indices' | grep red | wc -l

```