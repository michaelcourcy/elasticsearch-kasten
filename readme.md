# Elastic search blueprint 

## Goal

Demonstrate how to make a Kasten Blueprint for elasticsearch. Today ES only support backup through it's snapshot api. 

Any attempt to use storage snapshot is not supported by ES and can lead to data loss. 

Hence we're not going to use Kasten storage management for handling the backup of ES but rather integrate ES snapshot API in Kasten with the use of a blueprint.

For the user the experience is transparent he continues to use and leverage the kasten UI and API as usual : 
- Define policy, frequency and retention policy
- Restoration/Migration 
- Multitenancy
- Complete capture of the namespace and the other applications within it
- ... 

It let also the elasticsearch cluster being backup with other workloads like postgres/mysql/mongo/user-defined-persitent-wl : polyglot backup.

## Sources

Backup 
- https://www.elastic.co/guide/en/elasticsearch/reference/current/snapshot-restore.html
- https://www.elastic.co/guide/en/elasticsearch/reference/current/snapshots-register-repository.html
- https://www.elastic.co/guide/en/elasticsearch/reference/current/repository-s3.html
- https://www.elastic.co/guide/en/elasticsearch/reference/current/repo-analysis-api.html
- https://www.elastic.co/guide/en/elasticsearch/reference/current/snapshots-restore-snapshot.html#restore-different-cluster
- https://www.elastic.co/guide/en/elasticsearch/reference/current/snapshots-restore-snapshot.html#restore-different-cluster

Secure setting 
- https://www.elastic.co/guide/en/elasticsearch/reference/current/elasticsearch-keystore.html
- https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-es-secure-settings.html
- https://www.elastic.co/guide/en/elasticsearch/reference/current/secure-settings.html#reloadable-secure-settings

ES on kubernetes
- https://www.elastic.co/guide/en/cloud-on-k8s/current/index.html
- https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-operator-config.html
- https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-snapshots.html

ES CRUD 
- https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-create-index.html
- https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-index_.html
- https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-get.html



## Terminology

Snapshots is widely used and could be a storage snapshot, a kasten snapshot or in this context an ES snapshot.

We're speakig of the former ES snapshot as snapshots. For the other kind of snaphot we'll explicitly prefix the type ie: kopia snapshot.

## How it works 

Elasticsearch stores snapshots in an off-cluster storage location called a snapshot repository. Before you can take or restore snapshots, you must register a snapshot repository on the cluster. Elasticsearch supports several repository types with cloud storage options.

Snapshots are automatically deduplicated to save storage space and reduce network transfer costs. To back up an index, a snapshot makes a copy of the index’s segments and stores them in the snapshot repository. Since segments are immutable, the snapshot only needs to copy any new segments created since the repository’s last snapshot.

Each snapshot is also logically independent. When you delete a snapshot, Elasticsearch only deletes the segments used exclusively by that snapshot. Elasticsearch doesn’t delete segments used by other snapshots in the repository.

In this matter ES Snaphots really look like Kopia Snapshots, they are incremental but an intermediate snapshot can be safely deleted, garbage collection only happen if the segment is no more referenced by any snapshots. As a conscequence you may be in situation were deleting a snapshot won't delete any segment.

## Don't try to backup the filesystem

It is explicitly said by elastic that ES snapshot is the only reliable way to backup elasticsearch.

https://www.elastic.co/guide/en/elasticsearch/reference/current/snapshot-restore.html#other-backup-methods

Any attempt to make storage/kopia snapshot of the PVC could lead to unconsitencies or silent data loss.

```
A copy of the data directories of a cluster’s nodes does not work as a backup because it is not a consistent representation of their contents at a single point in time. You cannot fix this by shutting down nodes while making the copies, nor by taking atomic filesystem-level snapshots, because Elasticsearch has consistency requirements that span the whole cluster. You must use the built-in snapshot functionality for cluster backups.
```

# Install elastic search 

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

Create secure config for the s3 repository
```
kubectl create secret generic s3-repository-config \
  --from-literal=s3.client.default.access_key=${AWS_S3_ACCESS_KEY_ID} \
  --from-literal=s3.client.default.secret_key=${AWS_S3_SECRET_ACCESS_KEY} 
```

Create a cluster 
```
cat <<EOF | kubectl apply -f -
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: quickstart
spec:
  version: 8.4.1
  # secureSettings:
  # - secretName: s3-repository-config
  nodeSets:
  - name: default
    count: 1
    config:
      node.store.allow_mmap: false
EOF
```

monitor 
```
kubectl get elasticsearch
```

You can check that the secret entries has been added to the elastic search keystore

```
k logs -c elastic-internal-init-keystore quickstart-es-default-0
```

## Create a client 

```
PASSWORD=$(kubectl get secret quickstart-es-elastic-user -o go-template='{{.data.elastic | base64decode}}')
kubectl run curl -it --restart=Never --rm --image curlimages/curl --env="PASSWORD=$PASSWORD" --command sh 
```

In the curl pod shell let's check some command 
```
curl -u "elastic:$PASSWORD" -k "https://quickstart-es-http:9200"
```


## Create some sample data 

```
# Create an index 
curl  -k -u "elastic:$PASSWORD" -X PUT "https://quickstart-es-http:9200/my-index-000001?pretty"
#  add a document 
curl  -k -u "elastic:$PASSWORD" -X PUT "https://quickstart-es-http:9200/my-index-000001/_doc/1?timeout=5m&pretty" -H 'Content-Type: application/json' -d'
{
  "@timestamp": "2099-11-15T13:12:00",
  "message": "GET /search HTTP/1.1 200 1070000",
  "user": {
    "id": "kimchy"
  }
}
'
# retreive document 
curl -k -u "elastic:$PASSWORD" -X GET "https://quickstart-es-http:9200/my-index-000001/_doc/1?pretty"
```

# Register a repository 

## Secure clien setting 

Let's add s3 credentials in the keystore, this command need to be executed on each elastic node. 

```
kubectl exec -it quickstart-es-default-0 -c elasticsearch -- bash -c "echo ${AWS_S3_ACCESS_KEY_ID} | /usr/share/elasticsearch/bin/elasticsearch-keystore add --stdin -f s3.client.default.access_key" 
kubectl exec -it quickstart-es-default-0 -c elasticsearch -- bash -c "echo ${AWS_S3_SECRET_ACCESS_KEY} | /usr/share/elasticsearch/bin/elasticsearch-keystore add --stdin -f s3.client.default.secret_key"
```

reload the secure settings, secure settings are reloadable without restart of the nodes.

```
curl -k -u "elastic:$PASSWORD" -X POST "https://quickstart-es-http:9200/_nodes/reload_secure_settings?pretty" -H 'Content-Type: application/json' -d'
{
  
}
'
```

## Naming

Let's register a repository following the k1O good practice and assuming were working with compatible s3. Later we'll improve the blueprint to handle the registration.

```
k10/<k8s-uid>/elastic-search/<namespace-uid>/<es-cluster-uid>
```

This approach garantee the unicity of the repository accross clusters, namespaces and elastic search instances.


Register the repo, execute this command in another terminal and paste the output between ===...=== in the elasticsearch client 
```
CLUSTER_UID=$(kubectl get ns default -o jsonpath='{.metadata.uid}')
NS_UID=$(kubectl get ns default -o jsonpath='{.metadata.uid}')
ES_CLUSTER_UID=$(kubectl get elasticsearch quickstart -o jsonpath='{.metadata.uid}')
REPO_PATH=k10/$CLUSTER_UID/elastic-search/$NS_UID/$ES_CLUSTER_UID
echo "======================"
cat <<EOF
curl -k -u "elastic:\$PASSWORD" -X PUT "https://quickstart-es-http:9200/_snapshot/k10_repo?pretty" -H 'Content-Type: application/json' -d'
{
  "type": "s3",
  "settings": {    
    "bucket": "michael-couchbase",
    "endpoint": "s3.amazonaws.com",
    "region": "eu-west-3",
    "base_path": "$REPO_PATH"
  }
}
'
EOF
echo "======================"
```


## Testing the repo 

Once the repository is created we need to check that it meets the requirement using the repo ananalysis api. 

https://www.elastic.co/guide/en/elasticsearch/reference/current/repo-analysis-api.html

```
curl -k -u "elastic:$PASSWORD" -X POST "https://quickstart-es-http:9200/_snapshot/k10_repo/_analyze?blob_count=10&max_blob_size=1mb&timeout=120s&pretty"
```

In our case we see no issue in the  issue_detected field

```
{
  "coordinating_node" : {
    "id" : "TSXhhLBRQ-mxERCAs3Dqrw",
    "name" : "quickstart-es-default-0"
  },
  "repository" : "k10_repo",
  "blob_count" : 10,
  "concurrency" : 10,
  "read_node_count" : 10,
  "early_read_node_count" : 2,
  "max_blob_size" : "1mb",
  "max_blob_size_bytes" : 1048576,
  "max_total_data_size" : "1gb",
  "max_total_data_size_bytes" : 1073741824,
  "seed" : 90130092,
  "rare_action_probability" : 0.02,
  "blob_path" : "temp-analysis-7e39YRLHTJiczmmIpGBuAQ",
  "issues_detected" : [ ],
  "summary" : {
    "write" : {
      "count" : 10,
      "total_size" : "2.9mb",
      "total_size_bytes" : 3141632,
      "total_throttled" : "23.49ms",
      "total_throttled_nanos" : 23493400,
      "total_elapsed" : "1.11s",
      "total_elapsed_nanos" : 1112300374
    },
    "read" : {
      "count" : 11,
      "total_size" : "3.6mb",
      "total_size_bytes" : 3867976,
      "total_wait" : "1.06ms",
      "total_wait_nanos" : 1064412,
      "max_wait" : "170.1micros",
      "max_wait_nanos" : 170102,
      "total_throttled" : "34.66ms",
      "total_throttled_nanos" : 34664844,
      "total_elapsed" : "97.5ms",
      "total_elapsed_nanos" : 97504450
    }
  },
  "listing_elapsed" : "47.49ms",
  "listing_elapsed_nanos" : 47494710,
  "delete_elapsed" : "406.47ms",
  "delete_elapsed_nanos" : 406478316
}
```

## Creating the first snapshot 

```
curl -k -u "elastic:$PASSWORD" -X PUT "https://quickstart-es-http:9200/_snapshot/k10_repo/my_snapshot_1?wait_for_completion=true&pretty"
```

You should get a state filed with SUCCESS 

```
{
  "snapshot" : {
    "snapshot" : "my_snapshot_1",
    "uuid" : "99Jr_QWgQQylqAN7MD3UDA",
    "repository" : "k10_repo",
    "version_id" : 8040199,
    "version" : "8.4.1",
    "indices" : [
      "my-index-000001",
      ".geoip_databases"
    ],
    "data_streams" : [ ],
    "include_global_state" : true,
    "state" : "SUCCESS",
    "start_time" : "2022-08-31T15:27:25.006Z",
    "start_time_in_millis" : 1661959645006,
    "end_time" : "2022-08-31T15:27:27.007Z",
    "end_time_in_millis" : 1661959647007,
    "duration_in_millis" : 2001,
    "failures" : [ ],
    "shards" : {
      "total" : 2,
      "failed" : 0,
      "successful" : 2
    },
    "feature_states" : [
      {
        "feature_name" : "geoip",
        "indices" : [
          ".geoip_databases"
        ]
      }
    ]
  }
}
```

You can also check the path has been created in the s3 bucket.

If the database is very big you may not use the parameter wait_for_completion=true instead you can monitor the status of the snapshot 

```
curl -k -u "elastic:$PASSWORD" -X GET "https://quickstart-es-http:9200/_snapshot/k10_repo/_current?pretty"
```

# Restore 

## check the snapshot and change the data

List the snapshot 
```
curl -k -u "elastic:$PASSWORD" -X GET "https://quickstart-es-http:9200/_snapshot/k10_repo/*?verbose=false&pretty"
```

Let's delete the document in the indice and add another one.
```
curl  -k -u "elastic:$PASSWORD" -X GET "https://quickstart-es-http:9200/my-index-000001/_search?pretty"

curl  -k -u "elastic:$PASSWORD" -X DELETE "https://quickstart-es-http:9200/my-index-000001/_doc/1?routing=shard-1&pretty"

curl  -k -u "elastic:$PASSWORD" -X GET "https://quickstart-es-http:9200/my-index-000001/_search?pretty"

curl  -k -u "elastic:$PASSWORD" -X PUT "https://quickstart-es-http:9200/my-index-000001/_doc/2?timeout=5m&pretty" -H 'Content-Type: application/json' -d'
{
  "@timestamp": "2099-11-15T13:12:00",
  "message": "GET /search HTTP/1.1 200 1070000",
  "user": {
    "id": "kimchy2"
  } 
}
'
curl  -k -u "elastic:$PASSWORD" -X GET "https://quickstart-es-http:9200/my-index-000001/_search?pretty"
```

## Restore the entire cluster 

### First stop this list of service 

```
curl  -k -u "elastic:$PASSWORD"  -X PUT "https://quickstart-es-http:9200/_cluster/settings?pretty" -H 'Content-Type: application/json' -d'
{
  "persistent": {
    "ingest.geoip.downloader.enabled": false
  }
}
'
curl  -k -u "elastic:$PASSWORD"  -X POST "https://quickstart-es-http:9200/_ilm/stop?pretty"
curl  -k -u "elastic:$PASSWORD"  -X POST "https://quickstart-es-http:9200/_ml/set_upgrade_mode?enabled=true&pretty"
curl  -k -u "elastic:$PASSWORD"  -X PUT "https://quickstart-es-http:9200/_cluster/settings?pretty" -H 'Content-Type: application/json' -d'
{
  "persistent": {
    "xpack.monitoring.collection.enabled": false
  }
}
'
curl  -k -u "elastic:$PASSWORD"  -X POST "https://quickstart-es-http:9200/_watcher/_stop?pretty"
```

### Add a restore user

this is to continue request while restoring the cluster.
```
kubectl exec -it quickstart-es-default-0 -c elasticsearch -- bash -c "/usr/share/elasticsearch/bin/elasticsearch-users useradd restore_user -p my_password -r superuser"
```

### Delete all datastreams and indices 
```
curl  -k -u "elastic:$PASSWORD" -X PUT "https://quickstart-es-http:9200/_cluster/settings?pretty" -H 'Content-Type: application/json' -d'
{
  "persistent": {
    "action.destructive_requires_name": false
  }
}
'
curl  -k -u "elastic:$PASSWORD" -X DELETE "https://quickstart-es-http:9200/_data_stream/*?expand_wildcards=all&pretty"
curl  -k -u "elastic:$PASSWORD" -X DELETE "https://quickstart-es-http:9200/*?expand_wildcards=all&pretty"
```

### Now restore 
```
curl  -k -u "elastic:$PASSWORD" -X POST "https://quickstart-es-http:9200/_snapshot/k10_repo/my_snapshot_1/_restore?pretty" -H 'Content-Type: application/json' -d'
{
  "indices": "*",
  "include_global_state": true
}
'
```

### Restart the service 
```
curl  -k -u "elastic:$PASSWORD"  -X PUT "https://quickstart-es-http:9200/_cluster/settings?pretty" -H 'Content-Type: application/json' -d'
{
  "persistent": {
    "ingest.geoip.downloader.enabled": true
  }
}
'
curl  -k -u "elastic:$PASSWORD"  -X POST "https://quickstart-es-http:9200/_ilm/start?pretty"
curl  -k -u "elastic:$PASSWORD"  -X POST "https://quickstart-es-http:9200/_ml/set_upgrade_mode?enabled=false&pretty"
curl  -k -u "elastic:$PASSWORD"  -X PUT "https://quickstart-es-http:9200/_cluster/settings?pretty" -H 'Content-Type: application/json' -d'
{
  "persistent": {
    "xpack.monitoring.collection.enabled": true
  }
}
'
curl  -k -u "elastic:$PASSWORD"  -X POST "https://quickstart-es-http:9200/_watcher/_start?pretty"
```

### Reenable destructive_requires_name

```
curl  -k -u "elastic:$PASSWORD" -X PUT "https://quickstart-es-http:9200/_cluster/settings?pretty" -H 'Content-Type: application/json' -d'
{
  "persistent": {
    "action.destructive_requires_name": null
  }
}
'
```


# Delete the snapshot 

```
curl -k -u "elastic:$PASSWORD" -X DELETE "https://quickstart-es-http:9200/_snapshot/k10_repo/my_snapshot_1?pretty"
```


# Blueprint 


## Install 

```
kubectl create -f elastic-bp.yaml
kubectl --namespace default annotate elasticsearch/quickstart \
    kanister.kasten.io/blueprint=elastic-bp