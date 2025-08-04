# ks-4.1.2
本文大部分安装包都已经下载到/kk/下

###  拉取docker镜像

(本文安装包都已经下载到/kk/下)
(本文安装包都已经下载到/kk/下)
(本文安装包都已经下载到/kk/下)

```bash
docker pull registry.cn-hangzhou.aliyuncs.com/kubesphereio-jdjn/kubesphereio-jdjn:4.1.2
```

###  下载 KubeSphere Core Helm Chart(本文大部分安装包都已经下载到/kk/下)


1.  安装 helm。
    
    ```bash
    curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
    ```
    
2.  下载 KubeSphere Core Helm Chart。
    
    ```bash
    VERSION=1.1.3     # Chart 版本
    helm fetch https://charts.kubesphere.io/main/ks-core-1.1.2.tgz
    ```
    

## 离线部署

### 1\. 准备工作

将联网主机 node1 上的三个文件同步至离线环境的 master 节点。

+   `kk`
    
+   `kubesphere.tar.gz`
    
+   `ks-core-1.1.3.tgz`
    

### 2\. 创建配置文件

1.  创建离线集群配置文件。
    
    ```bash
    ./kk create config --with-kubernetes v1.26.12
    ```
    
2.  修改配置文件。
    
    | 说明 |
    | --- |
    | 
    +   按照离线环境的实际配置修改节点信息。
        
    +   指定 `registry` 仓库的部署节点，用于 KubeKey 部署自建 Harbor 仓库。
        
    +   `registry` 里可以指定 `type` 类型为 `harbor`，否则默认安装 docker registry。
        
    +   对于 Kubernetes v1.24+，建议将 `containerManager` 设置为 `containerd`。
        
    
    
    
     |
    
    以下为示例配置文件。如需了解各参数的配置方法，请参阅[此文档](https://kubesphere.saowu.top/zh/docs/v4.1/03-installation-and-upgrade/02-install-kubesphere/02-install-kubernetes-and-kubesphere/)。
    
    ```yaml
    apiVersion: kubekey.kubesphere.io/v1alpha2
    kind: Cluster
    metadata:
      name: sample
    spec:
      hosts:
      - {name: master, address: 192.168.0.3, internalAddress: 192.168.0.3, user: root, password: "<REPLACE_WITH_YOUR_ACTUAL_PASSWORD>"}
      - {name: node2, address: 192.168.0.4, internalAddress: 192.168.0.4, user: root, password: "<REPLACE_WITH_YOUR_ACTUAL_PASSWORD>"}
      roleGroups:
        etcd:
        - master
        control-plane:
        - master
        worker:
        - node2
        # 如需使用 kk 自动部署镜像仓库，请设置该主机组 （建议仓库与集群分离部署，减少相互影响）
        # 如果需要部署 harbor 并且 containerManager 为 containerd 时，由于部署 harbor 依赖 docker，建议单独节点部署 harbor
        registry:
        - node2
      controlPlaneEndpoint:
        ## Internal loadbalancer for apiservers
        # internalLoadbalancer: haproxy
    
        domain: lb.kubesphere.local
        address: ""
        port: 6443
      kubernetes:
        version: v1.26.12
        containerManager: containerd
      network:
        plugin: calico
        kubePodsCIDR: 10.233.64.0/18
        kubeServiceCIDR: 10.233.0.0/18
        ## multus support. https://github.com/k8snetworkplumbingwg/multus-cni
        multusCNI:
          enabled: false
      registry:
        # 如需使用 kk 部署 harbor, 可将该参数设置为 harbor，不设置该参数且需使用 kk 创建容器镜像仓库，将默认使用 docker registry。
        # type: harbor
        # 如使用 kk 部署的 harbor 或其他需要登录的仓库，需设置对应仓库的 auths，如使用 kk 创建默认的 docker registry 仓库，则无需配置 auths 参数。
        auths:
          "dockerhub.kubekey.local":
            # 部署 harbor 时需指定 harbor 帐号密码
            # username: admin
            # password: Harbor12345
            skipTLSVerify: true
        # 设置集群部署时使用的私有仓库地址。如果您已有可用的镜像仓库，请替换为您的实际镜像仓库地址。
        # 如果离线包中为原始 dockerhub 镜像（即 manifest 文件中的镜像地址为 docker.io/***），可以将该参数设置为 dockerhub.kubekey.local/ks, 表示将镜像全部推送至名为 ks 的 harbor 项目中。
        privateRegistry: "dockerhub.kubekey.local"
        # 如果构建离线包时 Kubernetes 镜像使用的是阿里云仓库镜像，需配置该参数。如果使用 dockerhub 镜像，则无需配置此参数。
        namespaceOverride: "kubesphereio"
        registryMirrors: []
        insecureRegistries: []
      addons: []
    ```
    

### 3\. 创建镜像仓库

| 说明 |
| --- |
| 
如果您已有可用的镜像仓库，可跳过此步骤。



 |

执行以下命令创建镜像仓库。

```bash
./kk init registry -f config-sample.yaml -a kubesphere.tar.gz
```

+   `config-sample.yaml` 为离线集群的配置文件。
    
+   `kubesphere.tar.gz` 为包含 ks-core 及各扩展组件镜像的离线安装包。
    

如果显示如下信息，则表明镜像仓库创建成功。

```bash
Pipeline[InitRegistryPipeline] execute successfully
```

### 4\. 创建 harbor 项目（若镜像仓库为 Harbor）

| 说明 |
| --- |
| 
由于 Harbor 项目存在访问控制（RBAC）的限制，即只有指定角色的用户才能执行某些操作。如果您未创建项目，则镜像不能被推送到 Harbor。Harbor 中有两种类型的项目：

+   公共项目（Public）：任何用户都可以从这个项目中拉取镜像。
    
+   私有项目（Private）：只有作为项目成员的用户可以拉取镜像。
    

Harbor 管理员账号：**admin**，密码：**Harbor12345**。

harbor 安装文件在 `/opt/harbor` 目录下，可在该目录下对 harbor 进行运维。



 |

执行以下命令创建 harbor 项目。

1.  创建脚本配置文件。
    
    ```bash
    vi create_project_harbor.sh
    ```
    
    ```bash
    #!/usr/bin/env bash
    
    # Copyright 2018 The KubeSphere Authors.
    #
    # Licensed under the Apache License, Version 2.0 (the "License");
    # you may not use this file except in compliance with the License.
    # You may obtain a copy of the License at
    #
    #     http://www.apache.org/licenses/LICENSE-2.0
    #
    # Unless required by applicable law or agreed to in writing, software
    # distributed under the License is distributed on an "AS IS" BASIS,
    # WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
    # See the License for the specific language governing permissions and
    # limitations under the License.
    
    url="https://dockerhub.kubekey.local"  # 或修改为实际镜像仓库地址
    user="admin"
    passwd="Harbor12345"
    
    harbor_projects=(
            ks
            kubesphere
            kubesphereio
            coredns
            calico
            flannel
            cilium
            hybridnetdev
            kubeovn
            openebs
            library
            plndr
            jenkins
            argoproj
            dexidp
            openpolicyagent
            curlimages
            grafana
            kubeedge
            nginxinc
            prom
            kiwigrid
            minio
            opensearchproject
            istio
            jaegertracing
            timberio
            prometheus-operator
            jimmidyson
            elastic
            thanosio
            brancz
            prometheus
    )
    
    for project in "${harbor_projects[@]}"; do
        echo "creating $project"
        curl -u "${user}:${passwd}" -X POST -H "Content-Type: application/json" "${url}/api/v2.0/projects" -d "{ \"project_name\": \"${project}\", \"public\": true}" -k  # 注意在 curl 命令末尾加上 -k
    done
    ```
    
2.  创建 Harbor 项目。
    
    ```bash
    chmod +x create_project_harbor.sh
    ```
    
    ```bash
    ./create_project_harbor.sh
    ```
    

### 5\. 安装 Kubernetes

执行以下命令创建 Kubernetes 集群：

```bash
./kk create cluster -f config-sample.yaml -a kubesphere.tar.gz --with-local-storage
```

| 说明 |
| --- |
| 
指定 --with-local-storage 参数会默认部署 openebs localpv，如需对接其他存储，可在 Kubernetes 集群部署完成后自行安装。



 |

如果显示如下信息，则表明 Kubernetes 集群创建成功。

```bash
Pipeline[CreateclusterPipeline] execute successfully
Installation is complete.
```

### 6\. 安装 KubeSphere

1.  安装 KubeSphere。
    
    ```bash
    helm upgrade --install -n kubesphere-system --create-namespace ks-core ks-core-1.1.3.tgz \
         --set global.imageRegistry=dockerhub.kubekey.local/ks \
         --set extension.imageRegistry=dockerhub.kubekey.local/ks \
         --set ksExtensionRepository.image.tag=v1.1.2 \
         --debug \
         --wait
    ```
    
    | 说明 |
    | --- |
    | 
    +   `ksExtensionRepository.image.tag` 为之前获取到的 Extensions Museum 版本（即 [https://get-images.kubesphere.io/](https://get-images.kubesphere.io/) 上展示的最新扩展组件仓库版本）。
        
    +   如需高可用部署 KubeSphere，可在命令中添加 `--set ha.enabled=true,redisHA.enabled=true`。
        
    
    
    
     |
    
    如果显示如下信息，则表明 KubeSphere 安装成功：
    
    ```bash
    NOTES:
    Thank you for choosing KubeSphere Helm Chart.
    
    Please be patient and wait for several seconds for the KubeSphere deployment to complete.
    
    1. Wait for Deployment Completion
    
        Confirm that all KubeSphere components are running by executing the following command:
    
        kubectl get pods -n kubesphere-system
    2. Access the KubeSphere Console
    
        Once the deployment is complete, you can access the KubeSphere console using the following URL:
    
        http://192.168.6.6:30880
    
    3. Login to KubeSphere Console
    
        Use the following credentials to log in:
    
        Account: admin
        Password: P@88w0rd
    
    NOTE: It is highly recommended to change the default password immediately after the first login.
    For additional information and details, please visit https://kubesphere.io.
    ```
    
2.  从成功信息中的 **Console**、**Account** 和 **Password** 参数分别获取 KubeSphere Web 控制台的 IP 地址、管理员用户名和管理员密码，并使用网页浏览器登录 KubeSphere Web 控制台。
    
    | 说明 |
    | --- |
    | 
    取决于您的硬件和网络环境，您可能需要配置流量转发规则并在防火墙中放行 30880 端口。
    
    
    
     |
    
