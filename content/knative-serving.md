---
title: "knative-serving初探"
date: 2020-12-28T08:47:11+08:00
description: knative是流行的serverless框架，目前参与的公司主要是 Google、Pivotal、IBM、Red Hat，目前迭代很快
draft: false
---

- 介绍
1. knative是流行的serverless框架，目前参与的公司主要是 Google、Pivotal、IBM、Red Hat，目前迭代很快
1. knative 是建立在 kubernetes 和 istio 平台之上的，使用 kubernetes 提供的容器管理能力（deployment、replicaset、和 pods等），以及 istio 提供的网络管理功能（ingress、LB、dynamic route等）
1. 对比kubeless，knative发展更快，大公司支持多
1. knative可方便快速部署你的服务（docker镜像+一条命令），不用理解写k8s的yaml文件，自动扩容，灰度(金丝雀)发布...
- knative serving官方原版说明
1. Rapid deployment of serverless containers
1. Automatic scaling up and down to zero
1. Routing and network programming
1. Point-in-time snapshots of deployed code and configurations

-  安装环境 `kubernetes-1.19.*` `knative-0.19` `istio-1.7.5`
- 安装istio参考[https://knative.dev/docs/install/installing-istio/#installing-istio-without-sidecar-injection](https://knative.dev/docs/install/installing-istio/#installing-istio-without-sidecar-injection)
- 下载knative对应的istio版本，看官网knative-0.19对应istio-1.7.x。
```
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.7.5 TARGET_ARCH=x86_64 sh -
cd istio-1.7.5
export PATH=$PWD/bin:$PATH
```
- 首先通过Istio Operator来安装Istio
```
cat << EOF > ./istio-minimal-operator.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  values:
    global:
      proxy:
        autoInject: disabled
      useMCP: false
      # The third-party-jwt is not enabled on all k8s.
      # See: https://istio.io/docs/ops/best-practices/security/#configure-third-party-service-account-tokens
      jwtPolicy: first-party-jwt

  addonComponents:
    pilot:
      enabled: true
    prometheus:
      enabled: false

  components:
    ingressGateways:
      - name: istio-ingressgateway
        enabled: true
      - name: cluster-local-gateway
        enabled: true
        label:
          istio: cluster-local-gateway
          app: cluster-local-gateway
        k8s:
          service:
            type: ClusterIP
            ports:
            - port: 15020
              name: status-port
            - port: 80
              targetPort: 8080
              name: http2
            - port: 443
              targetPort: 8443
              name: https
EOF

istioctl install -f istio-minimal-operator.yaml
```
- 导出Istio的入口网关（可理解为spring cloud gateway），NodePort端口
```
kubectl patch svc -n istio-system istio-ingressgateway -p '{"spec": {"type": "NodePort"}}'
```

- Using Istio mTLS feature

```
kubectl create ns knative-serving
kubectl label namespace knative-serving istio-injection=enabled
```

- Set PeerAuthentication to PERMISSIVE on knative-serving system namespace.

```
cat <<EOF | kubectl apply -f -
apiVersion: "security.istio.io/v1beta1"
kind: "PeerAuthentication"
metadata:
  name: "default"
  namespace: "knative-serving"
spec:
  mtls:
    mode: PERMISSIVE
EOF
```

- Install the Knative Istio controller
```
kubectl apply --filename https://github.com/knative/net-istio/releases/download/v0.19.0/release.yaml
```

- 安装knative
https://knative.dev/docs/install/any-kubernetes-cluster/
```
kubectl apply --filename https://github.com/knative/serving/releases/download/v0.19.0/serving-crds.yaml
# 如报错执行多次
kubectl apply --filename https://github.com/knative/serving/releases/download/v0.19.0/serving-core.yaml
```
- 卸载knative不能直接删除命名空间，要通过`kubectl delete --filename https://github.com/knative/serving/releases/download/v0.19.0/serving-core.yaml`

- 检测安装后的版本
```
kubectl get namespace knative-serving -o 'go-template={{index .metadata.labels "serving.knative.dev/release"}}'
```
- 检查knative组件状态
```
$ kubectl get pods --namespace knative-serving

NAME                                READY   STATUS    RESTARTS   AGE
activator-845748699b-cf7xc          1/1     Running   2          53m
autoscaler-696d8868f4-f79zv         1/1     Running   3          53m
controller-7f7566fbcb-pqd7w         1/1     Running   3          53m
istio-webhook-7dd7c94c7b-kz5k7      1/1     Running   0          53m
networking-istio-85f6b5c894-fhcrz   1/1     Running   3          53m
webhook-54d6f984b4-nx8ll            1/1     Running   3          53m
```

- 下载对应的knative client [https://github.com/knative/client/releases](https://github.com/knative/client/releases)

- 拷贝k8s服务器的.kube目录mac系统根目录，在k8s服务器上无需拷贝
- 创建helloworld-go
```
kn service create helloworld-go --image gcr.io/knative-samples/helloworld-go --env TARGET="Go Sample v2"
```

- 把默认域名改成自己的 [https://knative.dev/docs/serving/using-a-custom-domain/](https://knative.dev/docs/serving/using-a-custom-domain/)
```
kubectl patch configmap/config-domain \
  --namespace knative-serving \
  --type merge \
  --patch '{"data":{"example.com":""}}'
  ```

- 解析域名，添加`*.default`到A记录中
```
*.default.example.com                   59     IN     A   35.237.28.44
```

- 查看istio入口网关端口
```
kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}'
```
 
- 访问

![](https://oscimg.oschina.net/oscnet/up-ad66608f9eba6200e3a91fa58a5fabdd79d.png)

- 如果自动获取证书，访问https。参考另一文
[https://my.oschina.net/u/160697/blog/4784648](https://my.oschina.net/u/160697/blog/4784648)

- autoscale/追踪参考[https://my.oschina.net/u/160697/blog/4791170](https://my.oschina.net/u/160697/blog/4791170)

- 实际例子[https://my.oschina.net/u/160697/blog/4792546](https://my.oschina.net/u/160697/blog/4792546)
