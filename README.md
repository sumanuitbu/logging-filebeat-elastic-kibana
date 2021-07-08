
### Logging with Filebeat-Elasticsearch-Kibana

Components

- **filebeat** - log collector

- **elasticserach** - log aggregator

  - 1 client pod
  - 2 master pods
  - 3 data pods
  
- **kibana** - log viewer

Implementation
- secure with user password
- enable secure tranport over https

### Official elasticsearch helm release 

> **https://github.com/elastic/helm-charts/tree/7.8/elasticsearch**

```
helm repo add elastic https://helm.elastic.co

helm repo list
```


### Generate elastic certificates

Elscticsearch support certificates in both formats .p12 and .crt.
We will test oout both ways - 

**PKCS#12 format**

The PKCS#12 or PFX format is a binary format for storing the server certificate, 
any intermediate certificates, and the private key into a single encryptable file. 
PFX files are usually found with the extensions .pfx and .p12. PFX files are 
typically used on Windows machines to import and export certificates and private keys. 

```
docker run -it --rm `
 -v ${PWD}/tls:/tls `
 -w /usr/share/elasticsearch/ `
 docker.elastic.co/elasticsearch/elasticsearch:7.8.1 bash

./bin/elasticsearch-certutil ca --out /tls/elastic-stack-ca.p12 --pass ''

./bin/elasticsearch-certutil cert  \
  --ca /tls/elastic-stack-ca.p12  --ca-pass '' \
  --dns "elasticsearch-client,elasticsearch-client.logging.svc,elasticsearch-client.logging.svc,cluster.local,localhost" \
  --pass '' --name elastic --out /tls/elastic-client-certificates.p12

./bin/elasticsearch-certutil cert  \
  --ca /tls/elastic-stack-ca.p12  --ca-pass '' \
  --dns "elasticsearch-data,elasticsearch-data.logging.svc,elasticsearch-data.logging.svc,cluster.local,localhost" \
  --pass '' --name elastic --out /tls/elastic-data-certificates.p12

./bin/elasticsearch-certutil cert  \
  --ca /tls/elastic-stack-ca.p12  --ca-pass '' \
  --dns "elasticsearch-master,elasticsearch-master.logging.svc,elasticsearch-master.logging.svc,cluster.local,localhost" \
  --pass '' --name elastic --out /tls/elastic-master-certificates.p12

openssl pkcs12 -in /tls/elastic-stack-ca.p12 -out /tls/ca.pem -clcerts -nokeys

openssl pkcs12 -in /tls/elastic-stack-ca.p12 -out /tls/ca-key.pem -nocerts -nodes

### cleanup

rm -rf /tls/certs.zip \
  /tls/ca  /tls/kibana \
  /tls/elastic-client /tls/elastic-data /tls/elastic-master

```

```
kubectl create ns logging 

kubectl -n logging create secret generic elastic-client-certificates --from-file=tls/elastic-client-certificates.p12 
kubectl -n logging create secret generic elastic-data-certificates --from-file=tls/elastic-data-certificates.p12 
kubectl -n logging create secret generic elastic-master-certificates --from-file=tls/elastic-master-certificates.p12 

kubectl -n logging create secret generic elastic-ca --from-file=tls/ca/ca.pem

### cleanup

kubectl -n logging delete secret `
  elastic-client-certificates `
  elastic-data-certificates `
  elastic-ca 

```
-----------------------------------------------------------------------

**CRT format**

CRT is a file extension for a digital certificate file used with a web browser. 
CRT files are used to verify a secure website’s authenticity, distributed by 
certificate authority (CA) companies such as GlobalSign, VeriSign and Thawte.

```
docker run -it --rm `
 -v ${PWD}/tls:/tls `
 -w /usr/share/elasticsearch/ `
 docker.elastic.co/elasticsearch/elasticsearch:7.8.1 bash

./bin/elasticsearch-certutil cert --keep-ca-key --pem \
  --in /tls/instance.yml --out /tls/certs.zip

unzip /tls/certs.zip  -d /tls/

openssl x509 -in /tls/ca/ca.crt -out /tls/ca/ca.pem -outform PEM

### cleanup

rm -rf /tls/certs.zip \
  /tls/ca  /tls/kibana \
  /tls/elastic-client /tls/elastic-data /tls/elastic-master

```

```
kubectl create ns logging 

kubectl -n logging create secret generic elastic-client-certificates `
  --from-file=tls/elastic-client/elastic-client.crt `
  --from-file=tls/elastic-client/elastic-client.key `
  --from-file=tls/ca/ca.crt

kubectl -n logging create secret generic elastic-data-certificates `
  --from-file=tls/elastic-data/elastic-data.crt `
  --from-file=tls/elastic-data/elastic-data.key `
  --from-file=tls/ca/ca.crt


kubectl -n logging create secret generic elastic-master-certificates `
  --from-file=tls/elastic-master/elastic-master.crt `
  --from-file=tls/elastic-master/elastic-master.key `
  --from-file=tls/ca/ca.crt

kubectl -n logging create secret generic elastic-ca `
  --from-file=tls/ca/ca.pem

### cleanup

kubectl -n logging delete secret `
  elastic-client-certificates `
  elastic-data-certificates `
  elastic-ca 

```

-----------------------------------------------------------------

### Setup elasticserach

**Install file beat**
```
helm install es-filebeat elastic/filebeat `
 --version 7.8.1 `
 --values es-7.8.1-values-filebeat.yaml `
 --namespace logging
```

**Install elasticsearch**
```
helm install es-master elastic/elasticsearch `
 --version 7.8.1 `
 --values es-7.8.1-values-master.yaml `
 --namespace logging


helm install es-data  elastic/elasticsearch `
 --version 7.8.1 `
 --values es-7.8.1-values-data.yaml `
 --namespace logging


helm install es-client  elastic/elasticsearch `
 --version 7.8.1 `
 --values es-7.8.1-values-client.yaml `
 --namespace logging

```

**Run alpine pod  and test api**
```
kubectl run alpine --image=openkubeio/alpine-openldap-client  --restart=Always

kubectl exec alpine -- curl -sL -u elastic:elasticelastic http://elasticsearch-master.logging.svc.cluster.local:9200/_cluster/state?pretty   

kubectl exec alpine -- curl -sL -u elastic:elasticelastic http://elasticsearch-client.logging.svc.cluster.local:9200/_cluster/state?pretty  

kubectl exec alpine -- curl -sL -u elastic:elasticelastic http://elasticsearch-client.logging.svc.cluster.local:9200/_cat/nodes?v
```

**Install Kibana**
```
helm install es-kibana elastic/kibana `
 --version 7.8.1 `
 --values es-7.8.1-values-kibana.yaml `
 --namespace logging

```

**Test Kibana**
```
kubectl exec alpine -- curl -sL -u elastic:elasticelastic http://es-kibana-kibana:5601 

kubectl port-forward -n logging svc/es-kibana-kibana 5601:5601  
```


CRT format Example
> https://www.elastic.co/blog/configuring-ssl-tls-and-https-to-secure-elasticsearch-kibana-beats-and-logstash

Cert convertion
> https://stackoverflow.com/questions/15144046/converting-pkcs12-certificate-into-pem-using-openssl
