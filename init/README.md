##### 1.添加mysql chart仓库

mysql-operator https://radondb.github.io/radondb-mysql-kubernetes/ 

状态变为Active

##### 2.在chart 商店里面搜索 “mysql-operator”安装

##### 3.在Deployment中可以看到一个名为 mysql-operator

参考github项目：https://github.com/radondb/radondb-mysql-kubernetes/blob/main/docs/en-us/deploy_radondb-mysql_operator_on_rancher.md

使用一下yaml创建mysql集群(https://github.com/radondb/radondb-mysql-kubernetes/blob/main/config/samples/mysql_v1alpha1_mysqlcluster.yaml)

```shell
apiVersion: mysql.radondb.com/v1alpha1
kind: MysqlCluster
metadata:
  name: sample
spec:
  replicas: 3
  mysqlVersion: "5.7"
  
  # the backupSecretName specify the secret file name which store S3 information,
  # if you want S3 backup or restore, please create backup_secret.yaml, uncomment below and fill secret name:
  # backupSecretName: 
  
  # if you want create mysqlcluster from S3, uncomment and fill the directory in S3 bucket below:
  # such as restoreFrom: "backup_202241423817"
  # restoreFrom: 
  
  # Restore from NFS, uncomment below and set the ip of NFS server
  # such as nfsServerAddress: "10.233.55.172"
  # nfsServerAddress: 
  mysqlOpts:
    user: radondb_usr
    password: RadonDB@123
    database: radondb
    initTokuDB: false

    # A simple map between string and string.
    # Such as:
    #    mysqlConf:
    #      expire_logs_days: "7"
    mysqlConf: {}

    resources:
      requests:
        cpu: 100m
        memory: 256Mi
      limits:
        cpu: 500m
        memory: 1Gi

  xenonOpts:
    image: radondb/xenon:1.1.5-alpha
    admitDefeatHearbeatCount: 5
    electionTimeout: 10000

    resources:
      requests:
        cpu: 50m
        memory: 128Mi
      limits:
        cpu: 100m
        memory: 256Mi

  metricsOpts:
    enabled: false
    image: prom/mysqld-exporter:v0.12.1

    resources:
      requests:
        cpu: 10m
        memory: 32Mi
      limits:
        cpu: 100m
        memory: 128Mi

  podPolicy:
    imagePullPolicy: IfNotPresent
    sidecarImage: radondb/mysql57-sidecar:v2.2.0
    busyboxImage: busybox:1.32

    slowLogTail: false
    auditLogTail: false

    labels: {}
    annotations: {}
    affinity: {}
    priorityClassName: ""
    tolerations: []
    schedulerName: ""
    # extraResources defines quotas for containers other than mysql or xenon.
    extraResources:
      requests:
        cpu: 10m
        memory: 32Mi

  persistence:
    enabled: true
    accessModes:
    - ReadWriteOnce
    #storageClass: ""
    size: 20Gi
```



