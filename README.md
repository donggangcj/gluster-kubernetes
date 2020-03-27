本代码仓从[gluster-kuberntes](https://github.com/gluster/gluster-kubernetes)克隆
出来。

源仓库master分支相关脚本无法部署成功，主要是由于kubernetes的版本发生了变化，一些beta版本
的资源升至稳定版本，所以一些模板yaml文件需要修改，这些修改很多在issus中能够发现，目前还未合并
到主分支，仍然需要手动修改。

本仓库还将镜像源改为国内镜像源，镜像源使用国内`daocloud.io/daoclou`下的镜像。同时还记录了
部署中一些需要注意的点。

1. glusterfs server和glusterfs client版本需要尽量保持一直，由于镜像中的glusterfs server
版本较低，为`7.1`，所以客户端不能够安装最新版本，centos7 yum版本回退操作如下：

```
# 获取事务ID
yum history info glusterfs-fuse 
# 删除所有依赖
yum history undo ${事务ID}
# 安装7.1版本客户端
yum install -y https://buildlogs.centos.org/centos/7/storage/x86_64/gluster-7/glusterfs-libs-7.1-1.el7.x86_64.rpm
yum install -y https://buildlogs.centos.org/centos/7/storage/x86_64/gluster-7/glusterfs-7.1-1.el7.x86_64.rpm
yum install -y https://buildlogs.centos.org/centos/7/storage/x86_64/gluster-7/glusterfs-client-xlators-7.1-1.el7.x86_64.rpm
yum install -y https://buildlogs.centos.org/centos/7/storage/x86_64/gluster-7/glusterfs-fuse-7.1-1.el7.x86_64.rpm

```

2. 国内需要注意时区问题，可能导致挂载失败。
3. 设备不能包含任何数据
4. 需要加载必要的内核模块


## 在每个节点上执行

```
fdisk -l
# 查看结果中是否有未被使用的磁盘

# 检查和加载内核模块
lsmod | grep dm_snapshot || modprobe dm_snapshot
lsmod | grep dm_mirror || modprobe dm_mirror
lsmod | grep dm_thin_pool || modprobe dm_thin_pool
# 检查加载是否成功
lsmod | egrep '^(dm_snapshot|dm_mirror|dm_thin_pool)'
# 输出内容
# dm_thin_pool           66358  0 
# dm_snapshot            39103  0 
# dm_mirror              22289  0 

# 安装mount.glusterfs命令
yum install -y https://buildlogs.centos.org/centos/7/storage/x86_64/gluster-7/glusterfs-libs-7.1-1.el7.x86_64.rpm
yum install -y https://buildlogs.centos.org/centos/7/storage/x86_64/gluster-7/glusterfs-7.1-1.el7.x86_64.rpm
yum install -y https://buildlogs.centos.org/centos/7/storage/x86_64/gluster-7/glusterfs-client-xlators-7.1-1.el7.x86_64.rpm
yum install -y https://buildlogs.centos.org/centos/7/storage/x86_64/gluster-7/glusterfs-fuse-7.1-1.el7.x86_64.rpm
# 默认安装的是glusterfs 3.12，为了和下面gk-deploy脚本里面安装的版本一致，手动安装7.1版本

# 查看glusterfs版本
glusterfs --version
glusterfs 7.1

mount.glusterfs -V
glusterfs 7.1
```

## 在主节点上执行

```
# step 1. 下载安装文件
# 以下测试使用的，最新commit是：
#   Latest commit
#   ec38822
# 请酌情来判断是否需要执行
git clone https://github.com/donggangcj/gluster-kubernetes.git
cd gluster-kubernetes/deploy

# step 2. 准备topology文件
# ***************************
# 重点关注
# - hostsnames.manage里面填写节点的hostname
# - hostnames.storage里面填写节点的ip
# - devices里面填写磁盘的名称
# ***************************
cat << EOF > topology.json
{
  "clusters": [
    {
      "nodes": [
        {
          "node": {
            "hostnames": {
              "manage": [
                "node01"
              ],
              "storage": [
                "${IP_LIST['node01']}"
              ]
            },
            "zone": 1
          },
          "devices": [
            "/dev/sdb"
          ]
        },
        {
          "node": {
            "hostnames": {
              "manage": [
                "node02"
              ],
              "storage": [
                "${IP_LIST['node02']}"
              ]
            },
            "zone": 1
          },
          "devices": [
            "/dev/sdb"
          ]
        },
        {
          "node": {
            "hostnames": {
              "manage": [
                "node03"
              ],
              "storage": [
                "${IP_LIST['node03']}"
              ]
            },
            "zone": 1
          },
          "devices": [
            "/dev/sdb"
          ]
        }
      ]
    }
  ]
}
EOF

# 安装前确认节点都ready状态
kubectl get nodes

# step 2. 部署heketi and GlusterFS
ADMIN_KEY=adminkey
USER_KEY=userkey
./gk-deploy -g -y -v --admin-key ${ADMIN_KEY} --user-key ${USER_KEY}
# 如果第一次没安装成功，需要二次安装，使用下面命令清除之前的安装资源
# 删除资源和服务
# ./gk-deploy -g --abort --admin-key adminkey --user-key userkey
# 查看lv名称
# lvs
# 删除lv
# lvremove /dev/vg
# 清除磁盘(在节点机器上执行)
# wipefs -a /dev/sdc

# step 3. 检查heketi和glusterfs运行情况
export HEKETI_CLI_SERVER=$(kubectl get svc/heketi --template 'http://{{.spec.clusterIP}}:{{(index .spec.ports 0).port}}')
echo $HEKETI_CLI_SERVER
curl $HEKETI_CLI_SERVER/hello
# Hello from Heketi
# 如果timeout的话，看看是不是master没搞成node节点，没加入kube-proxy
# 可以获取到地址之后，到node节点上执行curl操作


# step 4. 创建storageclass，来自动为pvc创建pv
SECRET_KEY=`echo -n "${ADMIN_KEY}" | base64`

cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: heketi-secret
  namespace: default
data:
  # base64 encoded password. E.g.: echo -n "mypassword" | base64
  key: ${SECRET_KEY}
type: kubernetes.io/glusterfs
EOF

cat << EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: glusterfs-storage
provisioner: kubernetes.io/glusterfs
parameters:
  resturl: "${HEKETI_CLI_SERVER}"
  restuser: "admin"
  secretNamespace: "default"
  secretName: "heketi-secret"
  volumetype: "replicate:3"
EOF
# 注意生成pvc资源时，需要指定storageclass为上面配置的"glusterfs-storage"


kubectl get nodes,pods
NAME          STATUS   ROLES    AGE    VERSION
node/node01   Ready    <none>   5d3h   v1.17.0
node/node02   Ready    <none>   5d3h   v1.17.0
node/node03   Ready    <none>   5d3h   v1.17.0

NAME                          READY   STATUS    RESTARTS   AGE
pod/glusterfs-bhprz           1/1     Running   0          45m
pod/glusterfs-jt64n           1/1     Running   0          45m
pod/glusterfs-vkfp5           1/1     Running   0          45m
pod/heketi-779bc95979-272qk   1/1     Running   0          38m

```