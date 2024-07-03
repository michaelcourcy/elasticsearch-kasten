# Elastic search blueprint 

## Disclaimer 

This blueprint is not a Kasten supported Blueprint. It is proposed as an example that you can derive but you are responsible for its maintainance and good functionning. 

Most data service are deployed with specificities that are impossible to guess. And most likely deploying this blueprint as in would not work.

For instance we assume in this blueprint that the elasticsearch-keystore is not using passphrase which may not be true in your situation and you would have to adapt the blueprint. 

Hence you need to read, understand and *adapt* this blueprint before deploying it.

## Limitations 

- backup only work if the test of the elasticsearch cluster is green
- It only works with s3 compliant profile 
- To delete a restore point and the corresponding elasticsearch snapshot the namespace with the cluster must exist
- When restoring in another namespace only restore the elasticsearches.elasticsearch.k8s.elastic.co object the rest of the artifact that starts with the name of the cluster (for instance the secrets or the configmaps) should be excluded from the restore. 
- If you backup the pvc do not restore them, the elasticsearch will repopulate from the 


## How it works 

This blueprint and the blueprintbinding will create a backup using the elasticsearch snapshot api. 

Elasticsearch stores snapshots in an off-cluster storage location called a snapshot repository. Before you can take or restore snapshots, you must register a snapshot repository on the cluster. Elasticsearch supports several repository types with cloud storage options. **The blueprint create this repository using your kasten profile.**

Snapshots are automatically deduplicated to save storage space and reduce network transfer costs. To back up an index, a snapshot makes a copy of the index’s segments and stores them in the snapshot repository. Since segments are immutable, the snapshot only needs to copy any new segments created since the repository’s last snapshot.

Each snapshot is also logically independent. When you delete a snapshot, Elasticsearch only deletes the segments used exclusively by that snapshot. Elasticsearch doesn’t delete segments used by other snapshots in the repository.


## Backup of the PVC is not a reliable backup just an extra security

It is explicitly said by elastic that ES snapshot is the only reliable way to backup elasticsearch.

https://www.elastic.co/guide/en/elasticsearch/reference/current/snapshot-restore.html#other-backup-methods

Any attempt to make storage/kopia snapshot of the PVC could lead to unconsitencies or silent data loss.


> A copy of the data directories of a cluster’s nodes does not work as a backup 
because it is not a consistent representation of their contents at a single point 
in time. You cannot fix this by shutting down nodes while making the copies, nor 
by taking atomic filesystem-level snapshots, because Elasticsearch has consistency
requirements that span the whole cluster. You must use the built-in snapshot 
functionality for cluster backups.


It's why with this blueprint you can exclude the elastic pvc from the kasten backup. 
that will accelerate your backup.

However as a double security you can also backup the filesystem but when restoring 
you can exclude the PVC from the restore, because the data will be repopulated from
the elastic snapshot.

# Test it 

## operator


Install the operator
```
kubectl create -f https://download.elastic.co/downloads/eck/2.4.0/crds.yaml
kubectl apply -f https://download.elastic.co/downloads/eck/2.4.0/operator.yaml
```

Monitor the logs 
```
kubectl -n elastic-system logs -f statefulset.apps/elastic-operator
```

## Create a cluster 

Create a cluster with 2 nodesets 
```
kubectl create ns test-es1
cat <<EOF | kubectl apply -n test-es1 -f -
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: quickstart-small
spec:
  version: 8.4.1
  nodeSets:
  - name: default
    count: 2          
    config:
      node.store.allow_mmap: false
      node.roles: ["master", "data", "ingest"]
    volumeClaimTemplates:
    - metadata:
        name: elasticsearch-data # Do not change this name unless you set up a volume mount for the data path.
      spec:
        accessModes: ["ReadWriteOnce"]        
        resources:
          requests:
            storage: 5Gi
  - name: data-only
    count: 2          
    config:
      node.store.allow_mmap: false
      node.roles: ["data"]
    volumeClaimTemplates:
    - metadata:
        name: elasticsearch-data # Do not change this name unless you set up a volume mount for the data path.
      spec:
        accessModes: ["ReadWriteOnce"]        
        resources:
          requests:
            storage: 10Gi
EOF
```

monitor 
```
kubectl get elasticsearch
```

## Deploy the blueprint and the blueprint binding

```
kubectl create -f elastic-bp-s3.yaml -n kasten-io
kubectl create -f elastic-bp-binding.yaml -n kasten-io
```

The binding will apply to any `elasticsearches.elasticsearch.k8s.elastic.co` resource on your kubernetes cluster.

## Create a client 

```
PASSWORD=$(kubectl get secret quickstart-small-es-elastic-user -o go-template='{{.data.elastic | base64decode}}')
kubectl run curl -it --restart=Never --rm --image ghcr.io/kanisterio/kanister-kubectl-1.18:0.109.0 --env="PASSWORD=$PASSWORD" --command bash 
```

In the curl pod shell let's check some command 
```
ES_URL="https://quickstart-small-es-http:9200"
curl -u "elastic:$PASSWORD" -k "${ES_URL}"
```


## Create some sample data 

```
# List the index 
curl  -k -u "elastic:$PASSWORD" -X GET "${ES_URL}/*?pretty"
# Create an index 
curl  -k -u "elastic:$PASSWORD" -X PUT "${ES_URL}/my-index-000001?pretty"
#  add an index and a document 
curl  -k -u "elastic:$PASSWORD" -X PUT "${ES_URL}/my-index-000002/_doc/1?timeout=5m&pretty" -H 'Content-Type: application/json' -d'
{
  "@timestamp": "2099-11-15T13:12:00",
  "message": "GET /search HTTP/1.1 200 1070000",
  "user": {
    "id": "kimchy-2"
  }
}
'
# retreive document 
curl -k -u "elastic:$PASSWORD" -X GET "${ES_URL}/my-index-000002/_doc/1?pretty"
# Get the health 
curl  -k -u "elastic:$PASSWORD" -X GET "${ES_URL}/_cluster/health?pretty"
```

## test 

Backup and restore in another namespace by selecting only the elasticsearch resource.

> [!NOTE]
> When restoring in another namespace only restore the elasticsearches.elasticsearch.k8s.elastic.co object the rest of the artifact that starts with the name of the cluster (for instance the secrets or the configmaps) should be excluded from the restore. 

Check you get back your data in this new cluster.

# Debug kanister kubetask


Sometime you may need to follow up the execution of the kubetask pod in kasten-io 

```
while true; do kubectl logs -n kasten-io  -l createdBy=kanister -f; sleep 1; done
```

or in the user namespace 

```
while true; do kubectl logs -n test-es1  -l createdBy=kanister -f; sleep 1; done
```

But the best is to follow all the operations in the kansister-svc pods 

Mac, first [install ggrep](https://stackoverflow.com/questions/59232089/how-to-install-gnu-grep-on-mac-os) 
```
kubectl logs -n kasten-io -l component=kanister --tail=10000 -f | ggrep '"LogKind":"datapath"' | ggrep -o -P '(?<="Pod_Out":").*?(?=",)'
```

Linux 
```
kubectl logs -n kasten-io -l component=kanister --tail=10000 -f | grep '"LogKind":"datapath"' | grep -o -P '(?<="Pod_Out":").*?(?=",)'
```

# Working with a custom certificate authority for your S3 repository

## Docs
https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-transport-settings.html#k8s-transport-third-party-tools
https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-transport-settings.html#k8s-transport-third-party-tools
https://www.elastic.co/guide/en/elasticsearch/reference/current/update-node-certs-different.html


It is explicitly said [in this documenation](https://www.elastic.co/guide/en/elasticsearch/reference/current/repository-s3.html) that for the s3 protocol 
```
- protocol :
The protocol to use to connect to S3. Valid values are either http or https. 
Defaults to https. When using HTTPS, this repository type validates the 
repository’s certificate chain using the JVM-wide truststore. 
Ensure that the root certificate authority is in this truststore using the 
JVM’s keytool tool. If you have a custom certificate authority for your S3 
repository and you use the Elasticsearch bundled JDK, then you will need to 
reinstall your CA certificate every time you upgrade Elasticsearch.
```

Then we're going to add our ca to the /usr/share/elasticsearch/jdk/lib/security/cacerts files in the elasticsearch pods.

## Obtaining the CA Chain of your S3 endpoint

To obtain the CA (Certificate Authority) chain of your S3 endpoint, you can use the `openssl` command-line tool. This is particularly useful when you need to verify the trust chain of a certificate or when you need to provide the full chain to a service or application.

## Steps

1. **Download the Certificate**: Use the `openssl` command to connect to your server and download the certificate. For example, to download the certificate from `minio.minio-mic.mcourcy.customers.kastenevents.com`, you can use:

   ```bash
   openssl s_client -showcerts -connect minio.minio-mic.mcourcy.customers.kastenevents.com:443 </dev/null
   ```

2. **Extract the CA Chain**: The output will include the server certificate followed by any intermediate certificates and the root CA certificate. You need to look for the lines between -----BEGIN CERTIFICATE----- and -----END CERTIFICATE----- for each certificate in the chain, except for the first one, which is the server's certificate.

3. **Save the CA Chain**: Copy the intermediate and root CA certificates (everything except the first certificate) to a new file. This file represents your CA chain. Ensure to include the BEGIN CERTIFICATE and END CERTIFICATE lines for each certificate. 

For instance If I obtain this output 
```
Connecting to 15.237.4.98
CONNECTED(00000006)
depth=2 C=US, O=Internet Security Research Group, CN=ISRG Root X1
verify return:1
depth=1 C=US, O=Let's Encrypt, CN=R10
verify return:1
depth=0 CN=minio.minio-mic.mcourcy.customers.kastenevents.com
verify return:1
---
Certificate chain
 0 s:CN=minio.minio-mic.mcourcy.customers.kastenevents.com
   i:C=US, O=Let's Encrypt, CN=R10
   a:PKEY: rsaEncryption, 2048 (bit); sigalg: RSA-SHA256
   v:NotBefore: Jul  1 19:55:18 2024 GMT; NotAfter: Sep 29 19:55:17 2024 GMT
-----BEGIN CERTIFICATE-----
MIIFNTCCBB2gAwIBAgISBKZRqqXrPaIrQfnWESIQZIFoMA0GCSqGSIb3DQEBCwUA
MDMxCzAJBgNVBAYTAlVTMRYwFAYDVQQKEw1MZXQncyBFbmNyeXB0MQwwCgYDVQQD
EwNSMTAwHhcNMjQwNzAxMTk1NTE4WhcNMjQwOTI5MTk1NTE3WjA9MTswOQYDVQQD
EzJtaW5pby5taW5pby1taWMubWNvdXJjeS5jdXN0b21lcnMua2FzdGVuZXZlbnRz
LmNvbTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAK8wABXsxchCEpYn
tkkACGsfb8u2bGFtDv+ro7oTdhaMq0n4qyzyDGvQMxhPiIVbepqvr7a/HZin8FtF
ikXKj0rQFV0soX0TjigCx4ob1ezXCKK3sdudsEu6r/jxGLuAwOv2zgUTfPeyFMBL
9FkXR9esNITEueWB8w1vzOZ1exX0q16aW9HRayS9kNM5ZUB6IqWWtYTra6jKr+/0
/Ljv28VTEMq6xb0qwMxMerCxoFhR8vVIbl+6IA7Dc2/ksN5oFqzt7t/7hfM/oflk
jF+8TilmCmpYJeazbEpACaCj2QZj1Y3r/+TBevyPajrmooLjqxMzVnLwyBJlB5De
iYzzlAkCAwEAAaOCAjcwggIzMA4GA1UdDwEB/wQEAwIFoDAdBgNVHSUEFjAUBggr
BgEFBQcDAQYIKwYBBQUHAwIwDAYDVR0TAQH/BAIwADAdBgNVHQ4EFgQUhiN8YvUQ
IEru9jVv6cUOyv8nZ34wHwYDVR0jBBgwFoAUu7zDR6XkvKnGw6RyDBCNojXhyOgw
VwYIKwYBBQUHAQEESzBJMCIGCCsGAQUFBzABhhZodHRwOi8vcjEwLm8ubGVuY3Iu
b3JnMCMGCCsGAQUFBzAChhdodHRwOi8vcjEwLmkubGVuY3Iub3JnLzA9BgNVHREE
NjA0gjJtaW5pby5taW5pby1taWMubWNvdXJjeS5jdXN0b21lcnMua2FzdGVuZXZl
bnRzLmNvbTATBgNVHSAEDDAKMAgGBmeBDAECATCCAQUGCisGAQQB1nkCBAIEgfYE
gfMA8QB2AD8XS0/XIkdYlB1lHIS+DRLtkDd/H4Vq68G/KIXs+GRuAAABkHAUoTAA
AAQDAEcwRQIgONN1Wh9v06h3HYMHYMOJaKbUuUewTppV/G9yMbOkzjsCIQDtCQaM
043+/fhY/FQ76bLMNuaN7/doTJmDq5eU7GjkYwB3AO7N0GTV2xrOxVy3nbTNE6Iy
h0Z8vOzew1FIWUZxH7WbAAABkHAUoSgAAAQDAEgwRgIhAJSdPY0XsLNHAikAKoqR
2Ef2mEp2mks1zJxfLidsO0zrAiEAzJkUxaBZZAURUEK1HE9GRfyML8JkoaapCIth
6VuCQsQwDQYJKoZIhvcNAQELBQADggEBAKxDLj4SfDNK4AZ9lhZ4Fdo3m4RXuYvP
/fD0b2XAyB6a/LCIIbJqy8Lg32sVMcRVN0hez9dhEJCNSQ+KV6fYzGzep6KB1g0i
8DKiGGirU5WnbvC60J94fXC2Lo7mqWcJrYtgKnwddKUSFBr+PN7oJ+4eKJmveHOA
Bckw63+o5YMPDlreLKKi7yl1mM7iZeeDckcIUKOEmDYNQHCWAfnX9N38DV2PVBF0
YQIa65k4FQ+sOWx3uRkvsf48O65eLJl6c46V6o0vvl2mlsgPMh1kSp1ILVsci9NT
LmUV+AJTEH/Nw2OAlL337odKNLJQ5ZRGdSBgkNX+uRsw/FNLF39o5Bg=
-----END CERTIFICATE-----
 1 s:C=US, O=Let's Encrypt, CN=R10
   i:C=US, O=Internet Security Research Group, CN=ISRG Root X1
   a:PKEY: rsaEncryption, 2048 (bit); sigalg: RSA-SHA256
   v:NotBefore: Mar 13 00:00:00 2024 GMT; NotAfter: Mar 12 23:59:59 2027 GMT
-----BEGIN CERTIFICATE-----
MIIFBTCCAu2gAwIBAgIQS6hSk/eaL6JzBkuoBI110DANBgkqhkiG9w0BAQsFADBP
MQswCQYDVQQGEwJVUzEpMCcGA1UEChMgSW50ZXJuZXQgU2VjdXJpdHkgUmVzZWFy
Y2ggR3JvdXAxFTATBgNVBAMTDElTUkcgUm9vdCBYMTAeFw0yNDAzMTMwMDAwMDBa
Fw0yNzAzMTIyMzU5NTlaMDMxCzAJBgNVBAYTAlVTMRYwFAYDVQQKEw1MZXQncyBF
bmNyeXB0MQwwCgYDVQQDEwNSMTAwggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEK
AoIBAQDPV+XmxFQS7bRH/sknWHZGUCiMHT6I3wWd1bUYKb3dtVq/+vbOo76vACFL
YlpaPAEvxVgD9on/jhFD68G14BQHlo9vH9fnuoE5CXVlt8KvGFs3Jijno/QHK20a
/6tYvJWuQP/py1fEtVt/eA0YYbwX51TGu0mRzW4Y0YCF7qZlNrx06rxQTOr8IfM4
FpOUurDTazgGzRYSespSdcitdrLCnF2YRVxvYXvGLe48E1KGAdlX5jgc3421H5KR
mudKHMxFqHJV8LDmowfs/acbZp4/SItxhHFYyTr6717yW0QrPHTnj7JHwQdqzZq3
DZb3EoEmUVQK7GH29/Xi8orIlQ2NAgMBAAGjgfgwgfUwDgYDVR0PAQH/BAQDAgGG
MB0GA1UdJQQWMBQGCCsGAQUFBwMCBggrBgEFBQcDATASBgNVHRMBAf8ECDAGAQH/
AgEAMB0GA1UdDgQWBBS7vMNHpeS8qcbDpHIMEI2iNeHI6DAfBgNVHSMEGDAWgBR5
tFnme7bl5AFzgAiIyBpY9umbbjAyBggrBgEFBQcBAQQmMCQwIgYIKwYBBQUHMAKG
Fmh0dHA6Ly94MS5pLmxlbmNyLm9yZy8wEwYDVR0gBAwwCjAIBgZngQwBAgEwJwYD
VR0fBCAwHjAcoBqgGIYWaHR0cDovL3gxLmMubGVuY3Iub3JnLzANBgkqhkiG9w0B
AQsFAAOCAgEAkrHnQTfreZ2B5s3iJeE6IOmQRJWjgVzPw139vaBw1bGWKCIL0vIo
zwzn1OZDjCQiHcFCktEJr59L9MhwTyAWsVrdAfYf+B9haxQnsHKNY67u4s5Lzzfd
u6PUzeetUK29v+PsPmI2cJkxp+iN3epi4hKu9ZzUPSwMqtCceb7qPVxEbpYxY1p9
1n5PJKBLBX9eb9LU6l8zSxPWV7bK3lG4XaMJgnT9x3ies7msFtpKK5bDtotij/l0
GaKeA97pb5uwD9KgWvaFXMIEt8jVTjLEvwRdvCn294GPDF08U8lAkIv7tghluaQh
1QnlE4SEN4LOECj8dsIGJXpGUk3aU3KkJz9icKy+aUgA+2cP21uh6NcDIS3XyfaZ
QjmDQ993ChII8SXWupQZVBiIpcWO4RqZk3lr7Bz5MUCwzDIA359e57SSq5CCkY0N
4B6Vulk7LktfwrdGNVI5BsC9qqxSwSKgRJeZ9wygIaehbHFHFhcBaMDKpiZlBHyz
rsnnlFXCb5s8HKn5LsUgGvB24L7sGNZP2CX7dhHov+YhD+jozLW2p9W4959Bz2Ei
RmqDtmiXLnzqTpXbI+suyCsohKRg6Un0RC47+cpiVwHiXZAW+cn8eiNIjqbVgXLx
KPpdzvvtTnOPlC7SQZSYmdunr3Bf9b77AiC/ZidstK36dRILKz7OA54=
-----END CERTIFICATE-----
---
Server certificate
subject=CN=minio.minio-mic.mcourcy.customers.kastenevents.com
issuer=C=US, O=Let's Encrypt, CN=R10
---
No client certificate CA names sent
Peer signing digest: SHA256
Peer signature type: RSA-PSS
Server Temp Key: X25519, 253 bits
---
SSL handshake has read 3191 bytes and written 438 bytes
Verification: OK
---
New, TLSv1.3, Cipher is TLS_AES_256_GCM_SHA384
Server public key is 2048 bit
This TLS version forbids renegotiation.
Compression: NONE
Expansion: NONE
No ALPN negotiated
Early data was not sent
Verify return code: 0 (ok)
---
DONE
```

Keep this part and to proceed the operations with the keytool
do it in **any** pod of the cluster 
```
kubectl exec -it quickstart-small-es-data-only-0 -- bash
cat <<EOF > s3-ca.crt
-----BEGIN CERTIFICATE-----
MIIFBTCCAu2gAwIBAgIQS6hSk/eaL6JzBkuoBI110DANBgkqhkiG9w0BAQsFADBP
MQswCQYDVQQGEwJVUzEpMCcGA1UEChMgSW50ZXJuZXQgU2VjdXJpdHkgUmVzZWFy
Y2ggR3JvdXAxFTATBgNVBAMTDElTUkcgUm9vdCBYMTAeFw0yNDAzMTMwMDAwMDBa
Fw0yNzAzMTIyMzU5NTlaMDMxCzAJBgNVBAYTAlVTMRYwFAYDVQQKEw1MZXQncyBF
bmNyeXB0MQwwCgYDVQQDEwNSMTAwggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEK
AoIBAQDPV+XmxFQS7bRH/sknWHZGUCiMHT6I3wWd1bUYKb3dtVq/+vbOo76vACFL
YlpaPAEvxVgD9on/jhFD68G14BQHlo9vH9fnuoE5CXVlt8KvGFs3Jijno/QHK20a
/6tYvJWuQP/py1fEtVt/eA0YYbwX51TGu0mRzW4Y0YCF7qZlNrx06rxQTOr8IfM4
FpOUurDTazgGzRYSespSdcitdrLCnF2YRVxvYXvGLe48E1KGAdlX5jgc3421H5KR
mudKHMxFqHJV8LDmowfs/acbZp4/SItxhHFYyTr6717yW0QrPHTnj7JHwQdqzZq3
DZb3EoEmUVQK7GH29/Xi8orIlQ2NAgMBAAGjgfgwgfUwDgYDVR0PAQH/BAQDAgGG
MB0GA1UdJQQWMBQGCCsGAQUFBwMCBggrBgEFBQcDATASBgNVHRMBAf8ECDAGAQH/
AgEAMB0GA1UdDgQWBBS7vMNHpeS8qcbDpHIMEI2iNeHI6DAfBgNVHSMEGDAWgBR5
tFnme7bl5AFzgAiIyBpY9umbbjAyBggrBgEFBQcBAQQmMCQwIgYIKwYBBQUHMAKG
Fmh0dHA6Ly94MS5pLmxlbmNyLm9yZy8wEwYDVR0gBAwwCjAIBgZngQwBAgEwJwYD
VR0fBCAwHjAcoBqgGIYWaHR0cDovL3gxLmMubGVuY3Iub3JnLzANBgkqhkiG9w0B
AQsFAAOCAgEAkrHnQTfreZ2B5s3iJeE6IOmQRJWjgVzPw139vaBw1bGWKCIL0vIo
zwzn1OZDjCQiHcFCktEJr59L9MhwTyAWsVrdAfYf+B9haxQnsHKNY67u4s5Lzzfd
u6PUzeetUK29v+PsPmI2cJkxp+iN3epi4hKu9ZzUPSwMqtCceb7qPVxEbpYxY1p9
1n5PJKBLBX9eb9LU6l8zSxPWV7bK3lG4XaMJgnT9x3ies7msFtpKK5bDtotij/l0
GaKeA97pb5uwD9KgWvaFXMIEt8jVTjLEvwRdvCn294GPDF08U8lAkIv7tghluaQh
1QnlE4SEN4LOECj8dsIGJXpGUk3aU3KkJz9icKy+aUgA+2cP21uh6NcDIS3XyfaZ
QjmDQ993ChII8SXWupQZVBiIpcWO4RqZk3lr7Bz5MUCwzDIA359e57SSq5CCkY0N
4B6Vulk7LktfwrdGNVI5BsC9qqxSwSKgRJeZ9wygIaehbHFHFhcBaMDKpiZlBHyz
rsnnlFXCb5s8HKn5LsUgGvB24L7sGNZP2CX7dhHov+YhD+jozLW2p9W4959Bz2Ei
RmqDtmiXLnzqTpXbI+suyCsohKRg6Un0RC47+cpiVwHiXZAW+cn8eiNIjqbVgXLx
KPpdzvvtTnOPlC7SQZSYmdunr3Bf9b77AiC/ZidstK36dRILKz7OA54=
-----END CERTIFICATE-----
EOF
```

Still in the pod now copy the cacerts file in the current dyrectory and change its authorization 
for being able to alter it. 
```
cp /usr/share/elasticsearch/jdk/lib/security/cacerts cacerts
chmod 777 cacerts
/usr/share/elasticsearch/jdk/bin/keytool -import \
      -trustcacerts \
      -keystore /usr/share/elasticsearch/jdk/lib/security/cacerts \
      -storepass changeit \
      -noprompt -alias s3-ca
exit
```

And copy it in your laptop, we're going to convert it in a secret 
```
kubectl cp quickstart-small-es-data-only-0:/usr/share/elasticsearch/cacerts cacerts
```

Now you can create a secret containing the cacerts that will be mounted later 
in the elasticsearch container.
```
kubectl create secret generic quickstart-small-es-cacerts --from-file=cacerts=cacerts
```

Recreate the elasticsearch cluster with this secret mounted in  /usr/share/elasticsearch/jdk/lib/security/cacerts
```
k delete elasticsearches.elasticsearch.k8s.elastic.co --all
cat <<EOF | kubectl create -n test-es1 -f -
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: quickstart-small
spec:
  version: 8.4.1
  nodeSets:
  - name: default
    count: 2          
    config:
      node.store.allow_mmap: false
      node.roles: ["master", "data", "ingest"]      
    podTemplate:
      spec:
        containers:
        - name: elasticsearch
          volumeMounts:
          - name: cacerts
            mountPath: /usr/share/elasticsearch/jdk/lib/security/cacerts
            subPath: cacerts         
        volumes:
        - name: cacerts
          secret:
            defaultMode: 420
            secretName: quickstart-small-es-cacerts
            items:
            - key: cacerts
              path: cacerts    
    volumeClaimTemplates:
    - metadata:
        name: elasticsearch-data # Do not change this name unless you set up a volume mount for the data path.
      spec:
        accessModes: ["ReadWriteOnce"]        
        resources:
          requests:
            storage: 5Gi
  - name: data-only
    count: 2          
    config:
      node.store.allow_mmap: false
      node.roles: ["data"]
    podTemplate:
      spec:
        containers:
        - name: elasticsearch
          volumeMounts:
          - name: cacerts
            mountPath: /usr/share/elasticsearch/jdk/lib/security/cacerts
            subPath: cacerts         
        volumes:
        - name: cacerts
          secret:
            defaultMode: 420
            secretName: quickstart-small-es-cacerts
            items:
            - key: cacerts
              path: cacerts
    volumeClaimTemplates:
    - metadata:
        name: elasticsearch-data # Do not change this name unless you set up a volume mount for the data path.
      spec:
        accessModes: ["ReadWriteOnce"]        
        resources:
          requests:
            storage: 10Gi
EOF
```


check on the pod that the keystore now contains the s3-ca alias 

```
kubectl exec -it quickstart-small-es-data-only-0 -- bash
/usr/share/elasticsearch/jdk/bin/keytool -list \
    -keystore /usr/share/elasticsearch/jdk/lib/security/cacerts \
    -storepass changeit | grep s3-ca
```

You should see the alias that was created 
```
s3-ca, Jul 2, 2024, trustedCertEntry,
```



