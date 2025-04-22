
## elasticsearch certificate update 

Updating the certificates for your Elasticsearch cluster is a critical maintenance task that ensures the continued security and integrity of your data. This guide will walk you through the essential steps involved in this process, aiming to minimize disruption and maintain the stability of your Elasticsearch environment. We'll cover everything from preparing your new certificates and securely distributing them across your nodes to updating the Elasticsearch configuration and performing a rolling restart. By following these instructions carefully, you can confidently update your Elasticsearch certificates and keep your cluster secure. Let's get started!

## Checking the expiration date of the current certificate.

In the first step, we need to check how much validity remains on our current certificate. We will use the following command for this purpose. Please note that all our processes are based on certificates in the `.p12` format. At this stage, all we need is the current SSL certificate file and its password.

> :warning: NOTE
> 
> Before starting, please review the documentation related to securing your Elasticsearch cluster and create a backup of your ELK stack configurations

```shell
openssl pkcs12 -in /etc/elasticsearch/elastic-certificates.p12 -nodes  | openssl x509 -noout -enddate
```