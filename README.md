# 原README

https://github.com/moule3053/mck8s/blob/main/README.md

# 离线部署

## 部署K8S
```sh
# 安装3个单节点集群
# node0 - 管理集群 cluster0 <node0_ip_address>
# node1 - 工作集群 cluster1 <node1_ip_address>
# node2 - 工作集群 cluster2 <node2_ip_address>
```

## 生成kubeconfig
```sh
# 在node0上拷贝所有集群的配置文件
cd /root/.kube
cp config cluster0
scp node1:/root/.kube/config cluster1
scp node2:/root/.kube/config cluster2

# 替换集群名、用户名、证书校验
for f in cluster*;do
  sed -i "s/kubernetes/$f/g" $f
  #sed -i 's/certificate-authority-data.*$/insecure-skip-tls-verify: true/g' $f
done

# 修改cluster0/cluster1/cluster2，设置IP地址
    server: https://<node0_ip_address>:6443
    server: https://<node1_ip_address>:6443
    server: https://<node2_ip_address>:6443

# 设置kubectl环境变量
export KUBECONFIG="/root/.kube/cluster0:/root/.kube/cluster1:/root/.kube/cluster2"

# 重命名contexts
for i in {0..2}; do
  kubectl config rename-context cluster$i-admin@cluster$i cluster$i
done

# 查看配置
kubectl config get-contexts

# 整合配置文件
cp config config.local
kubectl config view --raw=true > config
unset KUBECONFIG
```

## 安装Prometheus

```sh
# 在每个节点上导入images
docker load < images/prom-alertmanager-amd64-v0.21.0-image.tar
docker load < images/prom-node-exporter-amd64-v1.0.1-image.tar
docker load < images/prom-prometheus-amd64-v2.19.2-image.tar

# 修改node0的配置文件，聚合其他节点数据
echo <<EOF >> config/prometheus/configmap.yaml
    - job_name: 'agg-prom'
      scrape_interval: 15s
      metrics_path: /federate
      honor_labels: true
      scheme: http
      params:
        match[]:
          - '{__name__=~"job:.*"}'
          - '{job="prometheus"}'
          - '{job="kubernetes-nodes"}'
          - '{job="kubernetes-cadvisor"}'
          - '{name=~".+"}'
          - '{job="kubernetes-service-endpoints"}'
          - '{job="kubernetes-pods"}'
          - '{job="kubernetes-apiservers"}'
          - '{pod_name=".+"}'
          - '{namespace="global"}'
          - '{job="node-exporter"}'
      static_configs:
        - targets: [<node1_ip_address>:30003]
          labels:
            cluster_name: cluster1
        - targets: [<node2_ip_address>:30003]
          labels:
            cluster_name: cluster2
      tls_config:
        insecure_skip_verify: true
EOF


# 在所有节点上安装prometheus
kubectl apply -f config/node_exporter/node_exporter.yaml
kubectl apply -f config/prometheus/configmap.yaml
kubectl apply -f config/prometheus/rbac-setup.yaml
kubectl apply -f config/prometheus/prometheus.yaml
kubectl apply -f config/alertmanager/alertmanager.yaml
```

## 安装metrics-server
```sh
# 在所有节点上安装metrics-server
docker load < images/k8s.gcr.io-metrics-server-metrics-server-amd64-v0.3.7-image.tar
kubectl apply -f config/metrics_server.yaml
```

## 安装helm
```sh
tar -zxf helm-v3.6.3-linux-amd64.tar.gz
mv linux-amd64/helm /usr/local/bin/
```

## 安装kubefed
```sh
# 安装kubefedctl
tar -zxf kubefedctl-0.8.1-linux-amd64.tgz
mv kubefedctl /usr/local/bin/

# 加载镜像
docker load < images/kubefed-v0.8.1.tar
docker load < images/bitnami-kubectl-1.17.16.tar

# 安装kubefed
helm --namespace kube-federation-system upgrade -i kubefed kubefed-0.8.1.tgz --create-namespace

# 加入kubefed集群
for i in {1..2}; do
  kubefedctl join cluster$i --cluster-context cluster$i --host-cluster-context cluster0 --v=2
done
```

## 安装Cilium
```sh
# 在所有节点上安装Cilium CLI，加载镜像
tar -zxf cilium-linux-amd64.tar.gz
mv cilium /usr/local/bin/
for f in images/*;do docker load < $f;done

# 在node0上安装
cilium install --cluster-id 1 --cluster-name cluster0
cilium clustermesh enable --service-type NodePort

# 在node1上安装
cilium install --cluster-id 2 --cluster-name cluster1
cilium clustermesh enable --service-type NodePort

# 在node2上安装
cilium install --cluster-id 3 --cluster-name cluster2
cilium clustermesh enable --service-type NodePort

# 在node0上建立连接
cilium clustermesh connect --destination-context cluster1
cilium clustermesh connect --destination-context cluster2

# 启动hubble（可观察性）
cilium hubble enable --ui

# 查看多集群的节点
kubectl exec -n kube-system -t ds/cilium -- cilium node list

# 启用hubble界面
cilium hubble ui

# 访问 http://<node0_ip_address>:12000/kube-system

```

## Build mck8s 
```sh
git clone https://github.com/00ahui/mck8s.git

cd mck8s

mkdir images

cd multi-cluster-horizontal-pod-autoscaler && docker build -t mck8s-autoscaler .
cd .. && docker save -o images/mck8s-autoscaler.tar mck8s-autoscaler

cd cloud-cluster-provisioner-autoscaler && docker build -t mck8s-provisioner .
cd .. && docker save -o images/mck8s-provisioner.tar mck8s-provisioner

cd multi-cluster-scheduler && docker build -t mck8s-scheduler .
cd .. && docker save -o images/mck8s-scheduler.tar mck8s-scheduler

cd multi-cluster-rescheduler && docker build -t mck8s-rescheduler .
cd .. && docker save -o images/mck8s-rescheduler.tar mck8s-rescheduler
```

## Intall mck8s 

```sh
# 在node0上加载镜像
for f in images/*.tar; do docker load < $f;done

# 加载CRDs
kubectl apply -f manifests/crds/01_rbac_mck8s.yaml
kubectl apply -f manifests/crds/

# 加载POD
kubectl apply -f manifests/controllers/01_deployment_multi_cluster_scheduler.yaml
kubectl apply -f manifests/controllers/02_deployment_multi_cluster_hpa.yaml
kubectl apply -f manifests/controllers/03_deployment_cloud_provisioner_cluster_autoscaler.yaml

# 测试
kubectl apply -f example/nginx.yaml
kubectl get mcd

```
