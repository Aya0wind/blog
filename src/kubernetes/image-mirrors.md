## 镜像源

+ 默认的k8s镜像源国内很多无法访问，例如k8s.gcr.io等，可以换成国内的源

| 镜像源                                                       | 更新速度 | 替换      |
| ------------------------------------------------------------ | -------- | --------- |
| http://hub-mirror.c.163.com                                  | 一般     | docker.io |
| http://registry.cn-hangzhou.aliyuncs.com/google_containers   |          |           |
| [使用阿里云代理k8s镜像](https://zhuanlan.zhihu.com/p/429685829) | 快       | 任意      |



### K3s(使用containerd)换源

```bash
echo "mirrors:
  "docker.io":
    endpoint:
      - \"http://hub-mirror.c.163.com\"">/etc/rancher/k3s/registries.yaml
```



### Helm源

| 名称                            | URL                                                          | 备注                                                       |
| :------------------------------ | ------------------------------------------------------------ | ---------------------------------------------------------- |
| bitnami                         | https://charts.bitnami.com/bitnami                           | bitnami官方仓库，东西比较全                                |
| gitlab                          | https://charts.gitlab.io                                     | gitlab官方仓库，包含gitlab-agent和gitlab等平台helm包。     |
| traefik                         | https://traefik.github.io/charts                             | traefik网关                                                |
| ali                             | https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts       | 阿里官方仓库，主要是阿里的工具                             |
| prometheus-community            | https://prometheus-community.github.io/helm-charts           | prometheus官方仓库，包含prometheus监控三件套打包helm和单件 |
| stable                          | http://mirror.azure.cn/kubernetes/charts/                    | 微软仓库，也比较全                                         |
| emqx                            | https://repos.emqx.io/charts                                 | 分布式mqtt集群                                             |
| jetstack                        | https://charts.jetstack.io                                   | 主要是有cert-manager                                       |
| nvdp                            | https://nvidia.github.io/k8s-device-plugin                   | Nvidia插件仓库，需要调度GPU安装                            |
| nfs-subdir-external-provisioner | https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/ | nfs卷分配器，需要基于nfs的持久卷声明使用                   |
| elastic                         | https://helm.elastic.co                                      | es全家桶仓库                                               |

### 常见问题

+ 镜像拉取失败

> 尝试给K3s换源，如果换源后还是拉取失败，检查对应源是否有对应镜像（可以crictl pull一下尝试看能否下载）。如果都失败，就拉取其他镜像仓库的镜像至私有仓库，然后重新打tag，修改deployment的镜像后重新部署拉取。