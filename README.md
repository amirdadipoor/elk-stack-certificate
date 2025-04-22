
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

:toolbox: Step-by-Step Guide (Using `openssl` and `elasticsearch-certutil`)

**🔓 1.Extract CA key and Cert From the `.p12`**

```shell
#  Extract the CA private key
openssl pkcs12 -in /path/to/elastic-stack-ca.p12 -nocerts -out ca.key -nodes

# Extract the CA certificate
openssl pkcs12 -in /path/to/elastic-stack-ca.p12 -clcerts -nokeys -out ca.crt
```
You'll be prompted for the .p12 password.

#


