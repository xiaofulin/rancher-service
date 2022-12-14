##### 1.通过sc.yaml创建存储类
```shell
## 创建了一个存储类
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-storage
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: k8s-sigs.io/nfs-subdir-external-provisioner
parameters:
  archiveOnDelete: "true"  ## 删除pv的时候，pv的内容是否要备份

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-client-provisioner
  labels:
    app: nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: default
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: nfs-client-provisioner
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: registry.cn-hangzhou.aliyuncs.com/lfy_k8s_images/nfs-subdir-external-provisioner:v4.0.2
          # resources:
          #    limits:
          #      cpu: 10m
          #    requests:
          #      cpu: 10m
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: k8s-sigs.io/nfs-subdir-external-provisioner
            - name: NFS_SERVER
              value: 192.168.1.40 ## 指定自己nfs服务器地址
            - name: NFS_PATH  
              value: /mnt/k8s_nfs/dev-test  ## nfs服务器共享的目录
      volumes:
        - name: nfs-client-root
          nfs:
            server: 192.168.1.40
            path: /mnt/k8s_nfs/dev-test
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: default
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-client-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    # replace with namespace where provisioner is deployed
    namespace: default
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: default
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: default
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    # replace with namespace where provisioner is deployed
    namespace: default
roleRef:
  kind: Role
  name: leader-locking-nfs-client-provisioner
  apiGroup: rbac.authorization.k8s.io


```

##### 2.添加mysql chart仓库

mysql-operator https://radondb.github.io/radondb-mysql-kubernetes/ 

状态变为Active

##### 3.在chart 商店里面搜索 “mysql-operator”安装

##### 4.在Deployment中可以看到一个名为 mysql-operator

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



