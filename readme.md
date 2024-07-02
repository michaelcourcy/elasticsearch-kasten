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


## Don't try to backup the filesystem

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

# TODO 

How to manage endpoint with custom certificate. 