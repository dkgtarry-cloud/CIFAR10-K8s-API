本项目是一个基于 Kubernetes 的 CIFAR-10 图像分类推理平台。
使用 Flask 构建推理服务，调用 PyTorch 训练的卷积神经网络（CNN）模型完成图像分类任务，
并通过 Docker + Kubernetes 实现容器化与集群级部署，支持自动化调度与服务访问。

本项目进一步引入 Kubernetes 容器编排体系，完成了从单容器到多节点集群的演进，
实践了 Pod 管理、Deployment 滚动更新、Service 暴露与 Ingress 路由 的完整流程,实现模型推理服务在集群环境下的可扩展、可维护部署，

通过本项目，我系统掌握了从 镜像制作 → 集群部署 → 网络暴露 → 故障排查与验证测试 的端到端流程，加深了对 Kubernetes 架构、服务暴露机制与 AI 平台基础运维思路 的理解。

## 🚀 环境说明
- Python  + PyTorch 
- Docker Desktop (K8s Enabled)
- kubectl CLI 工具
- 镜像仓库：本地 registry


## 🧱 架构设计
<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/ab9e5453-0baf-4a57-a682-bb618f25a568" />


## 部署步骤

构建 Docker 镜像(Docker Desktop自带的单节点Kubernetes集群当前不支持GPU调度，本次镜像构建采用CPU版本）
```bash
docker build -t cifar-api-cpu:v1 .
```

<img width="865" height="384" alt="image" src="https://github.com/user-attachments/assets/eef833f1-5525-4e3f-8f9b-0a2a49f8ad09" />

创建 Deployment

```bash
kubectl apply -f cifar10-deployment.yaml
```
验证部署状态：
```bash
kubectl get pods -o wide
```

<img width="865" height="52" alt="image" src="https://github.com/user-attachments/assets/ed01ffef-496f-4307-bb35-9e693bfeebce" />

<img width="865" height="95" alt="image" src="https://github.com/user-attachments/assets/d10df6b8-761b-4957-b181-b99e76ff4d42" />

暴露 Service

```bash
kubectl apply -f cifar10-service.yaml
```

<img width="865" height="48" alt="image" src="https://github.com/user-attachments/assets/6618ddff-e4b3-40c0-bd37-443913d04edd" />

<img width="865" height="72" alt="image" src="https://github.com/user-attachments/assets/99448cb8-e255-4857-8a85-e24d1ca47c2d" />

测试服务：
```bash
curl -X POST -F "file=@test_image.png" http://localhost:30080/predict
```
返回：
```bash
{"predict":"bird"}
```
<img width="865" height="46" alt="image" src="https://github.com/user-attachments/assets/e34637b4-7928-4a99-8d96-37f9f46fa640" />

配置 Ingress

```bash
kubectl apply -f cifar10-ingress.yaml
```
提示：annotation "kubernetes.io/ingress.class" 已弃用，应改为
spec.ingressClassName: nginx

<img width="865" height="86" alt="image" src="https://github.com/user-attachments/assets/f3a331c5-5db9-4317-befb-44f632a118f9" />

验证 Ingress 状态：
```bash
kubectl get ingress
```
验证推理接口（通过域名访问）
```bash
curl -v -X POST -F "file=@test_image.png" http://tarry.cifar10.local/predict
```
预期结果：（若无法解析域名，请在本地 hosts 文件中添加 127.0.0.1  tarry.cifar10.local ）
```bash
{"predict":"bird"}
```

<img width="865" height="448" alt="image" src="https://github.com/user-attachments/assets/ee7fc3af-26fa-42c2-997f-500cfb2520b4" />


## 故障排查与性能监控

一、镜像拉取失败（ImagePullBackOff）
手动修改Deployment.yaml为错误镜像名进行测试

<img width="573" height="680" alt="image" src="https://github.com/user-attachments/assets/3092dc0b-b35a-40b5-abe5-b2345a133148" />


```bash
kubectl get pods
```

问题表现：

<img width="865" height="143" alt="image" src="https://github.com/user-attachments/assets/fbb3a3fa-3edc-48c2-81b6-437cc682b951" />

排查命令：
```bash
kubectl describe pod <pod-name>
```
日志示例：
```bash
Failed to pull image "cifar-api:wrong-v1": failed to resolve reference ...
```
<img width="865" height="205" alt="image" src="https://github.com/user-attachments/assets/41a633e2-6d9e-4c4c-b65f-0df109ff8911" />

修正镜像名为正确的本地镜像（如 cifar-api:cpu-v1）
重新部署：
```bash
kubectl delete deployment cifar10-deployment
kubectl apply -f cifar10-deployment.yaml
```
修复后验证：
```bash
kubectl get pods
```
<img width="865" height="135" alt="image" src="https://github.com/user-attachments/assets/1243d635-5519-48f2-985f-13e83043fcc1" />


二、容器启动失败（CrashLoopBackOff）

Deployment 中强制执行错误命令：

<img width="550" height="615" alt="image" src="https://github.com/user-attachments/assets/891a316d-fabb-4b3a-8d76-354f59477735" />


容器尝试运行不存在的脚本，导致启动失败并不断重启。
```bash
kubectl get pods 
```

查看容器日志：
```bash
kubectl logs <pod-name>
```
输出：
```bash
python: can't open file '/app/wrong_entry.py': [Errno 2] No such file or directory
```

解决方案：
将命令改回正确启动脚本：
```bash
command: ["python", "app.py"]
```

重新部署后容器恢复正常运行。


三、实时监控与资源分析

实时监听 Pod 状态
```bash
kubectl get pods -w
```
-w 参数表示 watch，会实时输出 Pod 状态变化过程。

查看资源使用情况
```bash
kubectl top pods
kubectl top nodes
```
1m = 1 millicore CPU（约 0.1% 核心）

237Mi ≈ 237 MB 内存

可用于判断 Pod 或节点是否资源紧张。

查看系统事件时间线
```bash
kubectl get events --sort-by=.metadata.creationTimestamp | tail -10
```
显示最新系统事件（Pod 创建、重启、失败原因等）。

<img width="865" height="433" alt="image" src="https://github.com/user-attachments/assets/d040f5e5-2b3a-4412-b22f-d59c06272945" />



