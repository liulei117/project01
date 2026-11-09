```markdown
# 基于K3s三节点集群的离线可观测平台项目文档
> 系统环境：Ubuntu 24.04
> 集群架构：1 control-plane(master) + 2 worker 三节点K3s集群
> 项目目标：离线环境搭建完整云原生可观测平台，实现集群指标监控、容器日志采集、自定义告警并推送钉钉
> 技术栈：K3s、Helm3、kube-prometheus-stack、Loki+Promtail、Grafana、Alertmanager、prometheus-webhook-dingtalk

## 一、集群节点规划
| 主机名 | IP地址 | 角色 |
| ---- | ---- | ---- |
| k3s-master01 | 192.168.194.138 | control-plane（Server） |
| k3s-worker01 | 192.168.194.139 | worker（Agent） |
| k3s-worker02 | 192.168.194.140 | worker（Agent） |

> 执行标记说明
> ✅【三台全部执行】：master、worker01、worker02
> ✅【仅master执行】：只在控制面节点
> ✅【仅worker执行】：两台工作节点

## 二、阶段一：三台机器初始化（三台全部执行）
> 除主机名、静态IP每台单独修改，其余三台配置保持一致
1. **修改主机名**
```bash
# master节点
sudo hostnamectl set-hostname k3s-master01
# worker01节点
sudo hostnamectl set-hostname k3s-worker01
# worker02节点
sudo hostnamectl set-hostname k3s-worker02
```
> 说明：`hostnamectl`永久修改主机名，修改完成断开SSH重连，提示符主机名刷新。

2. **设置root密码**
```bash
sudo passwd root
```
> 说明：Ubuntu默认禁止root登录，该命令给root设置密码，方便切换root用户。

3. **更新系统 + 安装基础工具**
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget vim chrony bash-completion
```
> 说明
> - `apt update`：刷新软件源索引
> - `apt upgrade -y`：升级已安装包
> - `chrony`：时间同步服务；`bash-completion`：kubectl命令补全依赖包

4. **配置静态IP（netplan）**
```bash
sudo nano /etc/netplan/00-installer-config.yaml
```
master配置示例：
```yaml
network:
  ethernets:
    ens33:
      addresses: [192.168.194.138/24]
      gateway4: 192.168.194.2
      nameservers:
        addresses: [223.5.5.5,114.114.114.114]
  version: 2
```
> worker01地址改为`192.168.194.139/24`；worker02地址改为`192.168.194.140/24`
> 保存：`Ctrl+O`；退出：`Ctrl+X`
> 先测试配置（写错自动回滚，不会断网）
```bash
sudo netplan try
```
确认无误再正式应用
```bash
sudo netplan apply
```
校验
```bash
ip a
ping 223.5.5.5
```

5. **清理克隆虚拟机machine-id**
```bash
sudo rm /etc/machine-id
sudo systemd-machine-id-setup
```
> 说明：克隆虚拟机所有机器ID相同，会造成网络、证书、节点识别错乱，重建唯一machine-id。

6. **永久关闭swap（K8s硬性要求）**
```bash
sudo swapoff -a
sudo sed -i '/swap/s/^/#/' /etc/fstab
```
> 说明：立刻关闭交换分区，同时注释fstab里swap配置，重启不会自动开启；swap开启会干扰K8s内存调度。

7. **chrony时间同步 + 配置东八区**
```bash
# 设置系统时区为东八区
sudo timedatectl set-timezone Asia/Shanghai
# 修改chrony配置，使用国内时间源
sudo nano /etc/chrony/chrony.conf
```
将内置默认`pool xxx`替换为国内NTP源：
```
pool ntp.aliyun.com iburst
pool ntp.sjtu.edu.cn iburst
```
保存退出，重启并开机自启
```bash
sudo systemctl enable --now chronyd
sudo systemctl restart chronyd
```
校验
```bash
timedatectl
chronyc sources
```
确认时区：`Asia/Shanghai`，同步源正常。

8. **配置 /etc/hosts**
```bash
sudo nano /etc/hosts
```
追加内容：
```
192.168.194.138 k3s-master01
192.168.194.139 k3s-worker01
192.168.194.140 k3s-worker02
```
测试互通
```bash
ping k3s-worker01
ping k3s-worker02
```

## 三、阶段二：部署K3s集群
### 2.1 安装K3s Server（仅master）
> 使用国内Rancher镜像源，解决墙外脚本拉取超时；增加`--disable traefik`不部署自带Ingress，减少资源占用，后续按需单独部署
```bash
curl -sfL [https://rancher-mirror.rancher.cn/k3s/k3s-install.sh](https://rancher-mirror.rancher.cn/k3s/k3s-install.sh) | \
INSTALL_K3S_MIRROR=cn sh -s - server --disable traefik
```
等待执行完成，查看节点
```bash
kubectl get nodes
```

### 2.2 获取集群加入Token（仅master）
```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```
复制输出的token，不要带空格换行。

### 2.3 Worker节点加入集群（仅worker，两台都执行）
> worker同样使用国内源，将`K3S_TOKEN`替换上面拿到的值
```bash
curl -sfL [https://rancher-mirror.rancher.cn/k3s/k3s-install.sh](https://rancher-mirror.rancher.cn/k3s/k3s-install.sh) | \
INSTALL_K3S_MIRROR=cn K3S_URL=[https://192.168.194.138:6443](https://192.168.194.138:6443) \
K3S_TOKEN="这里替换master获取的token" sh -
```

### 2.4 集群校验（仅master）
```bash
kubectl get nodes
```
预期三台节点全部Ready。

### 2.5 手动添加master污点（仅master）
> 新版K3s默认不带污点，业务Pod会被调度到master，不符合控制面隔离规范，手动添加污点
```bash
kubectl taint node k3s-master01 node-role.kubernetes.io/control-plane:NoSchedule
```
查看污点确认
```bash
kubectl describe node k3s-master01 | grep Taints
```
> 如果提示`already exists`，说明已经有污点，无需操作。

### 2.6 kubectl bash自动补全（仅master）
> Ubuntu24.04默认不会自动加载补全脚本，写入`.bashrc`增加判断，避免重复写入
```bash
cat >> ~/.bashrc <<'EOF'
if command -v kubectl &> /dev/null && [ -f /usr/share/bash-completion/bash_completion ]; then
  source <(kubectl completion bash)
fi
EOF
```
生效
```bash
source ~/.bashrc
```
> 说明：先判断kubectl是否存在、bash-completion文件是否存在，再加载补全脚本；新开终端自动生效，支持Tab补全资源、命令。

### 2.7 集群基础验证（仅master）
```bash
# 查看集群全部Pod
kubectl get pods -A
# 查看集群版本
kubectl version
# 查看集群事件
kubectl get events
```

## 四、阶段三：离线资源前置准备（外网机器操作）
> 离线部署和在线helm最大区别：在线直接helm repo add，离线需要提前缓存所有Chart包、容器镜像，传到集群节点导入，全程不依赖外网
1. **下载helm3二进制包**
```bash
# 下载helm
wget [https://get.helm.sh/helm-v3.x.x-linux-amd64.tar.gz](https://get.helm.sh/helm-v3.x.x-linux-amd64.tar.gz)
# 解压，传到k3s-master节点
```
2. **下载Chart离线包**
```bash
# 外网添加仓库
helm repo add prometheus-community [https://prometheus-community.github.io/helm-charts](https://prometheus-community.github.io/helm-charts)
helm repo add grafana [https://grafana.github.io/helm-charts](https://grafana.github.io/helm-charts)
helm repo update
# 下载kube-prometheus-stack chart（压缩包）
helm pull prometheus-community/kube-prometheus-stack --untar=false
# 下载loki-stack chart（包含loki + promtail）
helm pull grafana/loki-stack --untar=false
```
下载后得到`.tgz`chart包，通过sftp上传到k3s-master节点`/obs-project/`目录。

3. **导出所有需要的容器镜像（外网导出tar包）**
需要提前梳理chart里面所有镜像，使用docker pull拉取后导出为tar包
- prometheus-stack相关：prometheus-operator、prometheus、alertmanager、node-exporter、kube-state-metrics、grafana
- loki-stack相关：loki、promtail
- 钉钉转发：timonwong/prometheus-webhook-dingtalk:v2.1.0

> 所有镜像包，全部传到k3s-master、k3s-worker01、k3s-worker02三个节点，**每个节点都要导入镜像**
```bash
# 节点导入镜像命令
ctr image import xxx.tar
# 校验导入完成
ctr images ls
```
> ⚠️重点：所有节点都必须导入镜像，否则Pod调度到worker节点会镜像拉取失败（离线环境无外网）

## 五、阶段四：Master节点离线安装Helm3
1. 将离线helm二进制包上传到k3s-master01
2. 解压、移动到系统PATH
```bash
tar -zxvf helm-v3.x.x-linux-amd64.tar.gz
chmod +x linux-amd64/helm
mv linux-amd64/helm /usr/local/bin/helm
helm version
```
> 离线环境不需要执行`helm repo add / helm repo update`，后续直接使用本地tgz包安装

## 六、阶段五：部署kube-prometheus-stack（指标监控栈）
### 4.1 目录准备
```bash
mkdir -p ~/obs-project/prometheus
cd ~/obs-project/prometheus
```
把离线下载好的`kube-prometheus-stack-xxx.tgz`放到这个目录，编写自定义`values.yaml`
values.yaml核心配置说明：
- prometheus：数据保留7天，资源限制，`serviceMonitor`选择器匹配prometheus标签
- alertmanager：配置`alertmanagerConfigSelector`，匹配label `alertmanager: loki`
- grafana：开启持久化，storageClass使用`local-path`；NodePort暴露30903；默认密码Admin@123；预置Prometheus、Alertmanager数据源，关闭自带默认数据源
- kube-state-metrics：开启采集所有K8s资源（pod、deployment、svc、secret等）
- 关闭kube-proxy、scheduler采集（本集群不需要）

### 4.2 离线helm安装命令
```bash
helm install kube-prometheus-stack ./kube-prometheus-stack-xxx.tgz -n monitoring --create-namespace -f values.yaml
```

### 4.3 查看Pod状态
```bash
kubectl get pods -n monitoring -w
```
等待prometheus、alertmanager、grafana、node-exporter、kube-state-metrics全部running

### 4.4 查看Service，获取Grafana访问端口
```bash
kubectl get svc -n monitoring
```
Grafana为NodePort，访问地址：`http://节点IP:30903`，账号：admin，密码：Admin@123

## 七、阶段六：部署Loki + Promtail日志栈（离线部署）
### 5.1 目录准备
```bash
mkdir -p ~/obs-project/loki
cd ~/obs-project/loki
```
放入离线loki-stack tgz包，编写values.yaml
values.yaml说明：
- loki：开启持久化，storageClass=local-path，10Gi存储
- promtail：DaemonSet，在所有节点上运行，采集容器标准输出日志，推送地址`http://loki:3100/loki/api/v1/push`

### 5.2 离线helm安装loki-stack
```bash
helm install loki ./loki-stack-xxx.tgz -n monitoring -f values.yaml
```

### 5.3 验证
```bash
kubectl get pods -n monitoring | grep loki
kubectl get pods -n monitoring | grep promtail
```

### 5.4 Grafana添加Loki数据源
地址：`[http://loki.monitoring.svc.cluster.local:3100](http://loki.monitoring.svc.cluster.local:3100)`
在Explore中查询日志，示例查询语句
```
{namespace="monitoring"}
```
> 截图验证：可以正常采集、查看监控命名空间日志

## 八、阶段七：部署钉钉告警中转服务 prometheus-webhook-dingtalk
> Alertmanager不能直接调用钉钉机器人，需要webhook中转服务，把alertmanager告警格式转为钉钉markdown格式
包含三个资源的yaml文件：
1. `dingtalk-deployment.yaml`：Deployment，镜像`timonwong/prometheus-webhook-dingtalk:v2.1.0`，挂载configmap配置
2. `dingtalk-service.yaml`：ClusterIP Service，端口8060
3. `dingtalk-configmap.yaml`：存放钉钉Webhook地址、加签密钥，配置target名称`webhook1`

### 6.1 目录准备
```bash
mkdir -p ~/obs-project/prometheus-webhook-dingtalk
cd ~/obs-project/prometheus-webhook-dingtalk
# 写入yaml文件
```

### 6.2 应用资源
```bash
kubectl apply -f dingtalk-configmap.yaml
kubectl apply -f dingtalk-deployment.yaml
kubectl apply -f dingtalk-service.yaml
# 校验
kubectl get pods -n monitoring | grep dingtalk
kubectl get svc -n monitoring | grep dingtalk
```
service域名：`prometheus-webhook-dingtalk.monitoring.svc.cluster.local:8060`
告警接收地址：`[http://prometheus-webhook-dingtalk.monitoring.svc.cluster.local:8060/webhook1/send](http://prometheus-webhook-dingtalk.monitoring.svc.cluster.local:8060/webhook1/send)`
> ⚠️这个url必须和后面AlertmanagerConfig里面的webhook地址一致；同时webhook名字和configmap内target名称保持一致

## 九、阶段八：Alertmanager告警路由 + PrometheusRule告警规则
### 7.1 AlertmanagerConfig告警路由配置（alertmanagerconfig.yaml）
CRD资源：AlertmanagerConfig，命名空间monitoring
- label匹配：`alertmanager: loki`（和prometheus-stack values里面alertmanagerConfigSelector对应）
- 路由规则：匹配所有告警，转发给上面webhook中转地址；配置group_wait、group_interval、repeat_interval；告警分组等待、重复告警间隔

应用配置
```bash
kubectl apply -f alertmanagerconfig.yaml -n monitoring
```

### 7.2 PrometheusRule告警规则（alert-rules.yaml）
CRD资源：PrometheusRule，定义告警表达式，本项目4条常用告警
1. NodeHighCPU：节点CPU使用率>80%，持续2m，级别warning
2. NodeHighMemory：节点内存使用率>80%，持续2m，级别warning
3. PodRestartFrequently：Pod15分钟内重启次数>3，级别critical
4. PodOOMKilled：Pod发生OOM，被内核杀死，级别critical

> yaml标签添加`prometheus: kube-prometheus-stack`，保证可以被prometheus的ruleSelector匹配加载
```bash
kubectl apply -f alert-rules.yaml -n monitoring
```

### 7.3 告警验证
1. 查看Prometheus页面Alerts，查看告警规则是否加载成功
2. 制造故障测试（节点CPU压测、启动会OOM的Pod），触发告警
3. 等待for时间后，钉钉收到FIRING告警；故障恢复后，钉钉收到RESOLVED恢复通知，和告警预期一致

## 十、阶段九：Grafana大盘说明 & 截图说明
导入通用K8s Node监控Dashboard
✅ 正常展示指标：节点CPU使用率、内存使用率、磁盘使用率、系统负载、磁盘IO、Pod运行数量、Pod重启次数、TOP高负载容器
⚠️ 面板N/A说明：
大盘内置`Current Connections`、请求速率面板，依赖Ingress-Nginx指标（`nginx_connections_active`、`nginx_ingress_controller_requests`）；**本项目需求不包含Ingress网关，没有部署ingress-nginx，因此这两块面板无数据显示N/A，不影响项目核心监控、告警、日志功能**。
> 面试说明话术：该Grafana大盘为通用K8s模板，内置Ingress网关监控面板；当前项目未部署Ingress，对应Ingress指标缺失显示N/A，集群核心监控、日志、告警链路全部正常。

## 十一、阶段十：集群整体验证&验收清单
```bash
# 集群节点状态
kubectl get nodes
# 监控组件Pod
kubectl get pods -n monitoring
# 查看service
kubectl get svc -n monitoring
# 持久化存储PVC
kubectl get pvc -n monitoring
```
预期：grafana、prometheus、loki PVC都是Bound状态

✅ 功能验收清单
1. Grafana监控面板：节点CPU、内存、负载、磁盘IO、Pod数量曲线正常展示
2. Loki日志：Grafana Explore可以查询集群容器日志
3. 告警：触发故障 → 钉钉收到FIRING告警；故障恢复 → 钉钉收到RESOLVED恢复通知

## 十二、离线项目踩坑汇总
1. **镜像导入问题**：离线环境镜像tar包必须所有节点都导入；只导入master，Pod调度到worker节点会镜像拉取失败
2. **Chart包路径问题**：离线helm install不能写远程仓库地址，必须写本地tgz文件路径，否则会去外网拉取
3. **AlertmanagerConfig标签匹配**：Alertmanager的selector标签，必须和AlertmanagerConfig资源的label完全一致，否则告警路由配置不生效
4. **Webhook target名称对齐**：Configmap target名称（webhook1）和Alertmanager webhook url路径必须一致，否则钉钉收不到告警
5. **K3s local-path存储类**：默认自带，不需要额外部署；数据存储在节点本地磁盘，删除节点数据丢失；需要分布式存储可替换为Longhorn
6. **文件权限**：离线传输镜像包、yaml文件，注意文件权限；ctr导入镜像需要root权限
7. **YAML缩进问题**：yaml缩进错误导致钉钉告警转发服务配置加载失败，修正缩进后告警恢复
8. **钉钉机器人校验**：开启IP白名单时集群Pod出口IP不固定收不到消息；改为仅使用加签校验调试成功
9. **PrometheusRule告警规则不加载**：告警资源标签和Prometheus CR的ruleSelector标签不匹配，修改yaml标签匹配后，规则正常加载识别
10. **告警延迟**：规则中`for`持续等待时间、alertmanager group_wait配置会造成告警推送慢，可按需调整参数
11. **时间同步坑**：时间不同步会导致Prometheus指标时间错乱、告警判定异常，必须配置国内chrony NTP源+东八时区

## 十三、项目文件目录结构
```
monitor-obs/
├── helm-values/
│   ├── kube-prometheus-stack-values.yaml   # prometheus栈helm配置
│   └── loki-stack-values.yaml              # loki日志栈helm配置
├── alertmanager/
│   ├── prometheusrule.yaml                 # 自定义告警规则yaml
│   └── alertmanagerconfig.yaml             # alertmanager告警路由配置
├── dingtalk-webhook/
│   ├── dingtalk-webhook-all.yaml      #dingtalk服务
├── docs/
│   ├── README.md                           # 这份项目说明文档
│   └── images/                             # Grafana监控截图、钉钉告警截图
└── charts/                                  # 离线tgz chart包
```

## 十四、项目总结
本项目基于三节点K3s集群搭建一套完整离线可观测平台，包含指标监控、日志采集、钉钉告警三大核心能力；全部Chart、容器镜像提前离线缓存，不依赖外网。
使用Helm管理监控组件，所有配置YAML固化留存，可重复部署、迁移。
集群可以实时观测节点、容器资源状态，故障自动告警推送至钉钉，满足K8s集群基础运维监控需求。
扩展方向：部署Ingress-Nginx补齐Grafana大盘Ingress指标；新增磁盘告警；接入分布式持久存储Longhorn；美化钉钉告警模板。