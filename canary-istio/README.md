# Canary：Istio

本示例使用 Argo Rollouts 编排金丝雀发布，通过 Istio VirtualService 将入口流量按权重分配给稳定版本和新版本。

## 资源与流量路径

| 文件 | 作用 |
| --- | --- |
| `namespace.yaml` | 创建 `canary-lab` 命名空间 |
| `rollouts.yaml` | 定义 `rollouts-demo`，默认 4 个副本，初始使用私有 ECR 中的 `rollouts-demo:green` 镜像 |
| `service.yaml` | 定义 stable、canary 两个 Service，服务端口为 80，容器端口为 8080 |
| `gateway.yaml` | 定义 `rollouts-demo-gateway`，选择 `istio: ingressgateway` 网关工作负载，监听 HTTP **89** 端口 |
| `virtualservice.yaml` | 定义 `rollouts-demo-vsvc` 的 `primary` 路由，初始 stable/canary 权重为 100/0 |
| `application.yaml` | Argo CD Application 模板，包含 VirtualService 权重差异忽略配置 |

请求路径为：客户端 → Istio Ingress Gateway → VirtualService `primary` → stable/canary Service:80 → 对应版本的 Pod:8080。

Argo Rollouts 更新 Service 的选择器和 VirtualService 路由权重。本示例按两个 Service 分流，无需通过 DestinationRule 定义版本 subset。详见 [Argo Rollouts Istio 集成文档](https://argo-rollouts.readthedocs.io/en/stable/features/traffic-management/istio/)。

## 前置条件

- Kubernetes 集群已安装 Argo Rollouts Controller 和 CRD，本机已安装 `kubectl` 及 `kubectl argo rollouts` 插件。
- 已安装 Istio 及 Ingress Gateway，支持清单使用的 `networking.istio.io/v1` API，网关 Pod 标签匹配 `istio: ingressgateway`。
- Gateway 资源配置了监听端口，但网关的 Kubernetes Service 还需单独暴露对应端口。本例可将 Service 的 `port: 89` 映射到 `targetPort: 89`，并配置外部负载均衡器和网络访问规则。参见 [Istio 入口网关文档](https://istio.io/latest/docs/tasks/traffic-management/ingress/ingress-control/)。
- 默认镜像位于 `073070604328.dkr.ecr.cn-north-1.amazonaws.com.cn/argoproj/rollouts-demo:green`，需要镜像拉取权限和网络连通性；也可以先将 `rollouts.yaml` 中的镜像改为可访问的仓库地址。
- `namespace.yaml` 未启用 Sidecar 自动注入。若环境要求应用接入网格，请在创建 Pod 前配置相应注入标签，并确保网关到后端的 mTLS 策略匹配。
- 本目录与 `canary-lbc` 使用相同的命名空间、Rollout 和 Service 名称。同一命名空间中一次部署一个示例；并行实验时需为其中一个示例统一修改命名空间，包括 Namespace 资源名称。

## 通过 kubectl 部署

以下命令均在仓库根目录执行。分别应用工作负载清单，`application.yaml` 用于后面的 Argo CD 部署方式。

```bash
kubectl apply -f canary-istio/namespace.yaml
kubectl apply -f canary-istio/service.yaml -f canary-istio/gateway.yaml -f canary-istio/virtualservice.yaml -f canary-istio/rollouts.yaml
kubectl argo rollouts get rollout rollouts-demo -n canary-lab --watch
```

首次部署会直接建立稳定版本，等待其健康后再触发更新。

查看网关 Service 地址和端口；以下示例假设网关安装在 `istio-system`，Service 名为 `istio-ingressgateway`，请按实际安装调整：

```bash
kubectl get svc istio-ingressgateway -n istio-system
```

若 Service 已对外暴露 89 端口，访问 `http://<网关外部地址>:89/`。也可以通过端口转发验证路由，保持下面的命令运行：

```bash
kubectl port-forward -n istio-system svc/istio-ingressgateway 8089:89
```

在另一个终端执行 `curl http://localhost:8089/`，或用浏览器打开该地址。端口转发同样要求网关 Service 已定义 89 端口。

## 触发发布与验证

将下面的镜像地址替换为仓库中已存在、集群可拉取的新版本：

```bash
NEW_IMAGE='<可拉取的新版本镜像完整地址>'
kubectl argo rollouts set image rollouts-demo "rollouts-demo=${NEW_IMAGE}" -n canary-lab
kubectl argo rollouts get rollout rollouts-demo -n canary-lab --watch
```

当前清单中的发布步骤如下，权重表示新版本接收的流量比例：

| 新版本流量 | 后续动作 |
| --- | --- |
| 10% | 暂停 60 秒 |
| 25% | 等待手动推进 |
| 50% | 暂停 60 秒 |
| 100% | 暂停 30 秒，然后完成发布 |

在 25% 阶段确认新版本正常后推进：

```bash
kubectl argo rollouts promote rollouts-demo -n canary-lab
```

查看实际路由权重，并通过网关持续访问示例页面观察版本变化：

```bash
kubectl get virtualservice rollouts-demo-vsvc -n canary-lab -o yaml
kubectl get pods,svc -n canary-lab
```

权重作用于经过该 VirtualService 的请求；直接访问某个后端 Service 无法验证入口流量分配，少量请求的实际分布也可能偏离配置权重。

## 使用 Argo CD

使用 `application.yaml` 前需按环境调整：

1. 将 `spec.source.repoURL` 的 `<repo url>` 替换为 Argo CD 能访问的仓库地址，并配置仓库凭据。
2. `targetRevision` 当前为 `master`，确保该分支包含本目录；如果使用 `main`，相应修改此字段。
3. `metadata.namespace` 当前为 `default`，应改为 Argo CD 实际管理 Application 的命名空间，常见为 `argocd`；保留 `default` 时需确认已配置跨命名空间 Application 支持及项目授权。
4. 将 `spec.destination.namespace` 设为 `canary-lab`，与工作负载清单一致，并确认项目允许部署到该集群、命名空间及创建 Namespace。
5. 在 `spec.source` 下添加以下配置，使该 Application 只同步业务清单，避免将同目录的 Application 模板纳入自身同步。参见 [Argo CD 文件排除配置](https://argo-cd.readthedocs.io/en/stable/user-guide/directory/#excluding-files)。

```yaml
directory:
  exclude: application.yaml
```

将调整后的文件提交、推送到所选分支，再创建并同步 Application：

```bash
kubectl apply -f canary-istio/application.yaml
argocd app sync canary-istio
```

模板没有启用自动同步。发布时修改 Git 中 `rollouts.yaml` 的镜像，提交、推送后再次同步。直接执行 `set image` 会使集群与 Git 配置产生差异。

模板中的 `ignoreDifferences` 忽略 `.spec.http[].route[].weight`，让 Argo Rollouts 管理发布过程中的权重；`RespectIgnoreDifferences=true` 使同步时也遵守这一规则，`ApplyOutOfSyncOnly=true` 仅应用不同步的资源。首次创建 VirtualService 时仍使用清单中的初始权重。参见 [Argo CD 同步选项](https://argo-cd.readthedocs.io/en/stable/user-guide/sync-options/#respect-ignore-differences-configs)。

## 中止与回滚

发布过程中可中止，将流量切回稳定版本：

```bash
kubectl argo rollouts abort rollouts-demo -n canary-lab
```

中止不会恢复期望的镜像配置。直接使用 kubectl 管理时，可执行以下命令恢复上一个修订；使用 Argo CD 时，应恢复 Git 中的镜像配置并同步。

```bash
kubectl argo rollouts undo rollouts-demo -n canary-lab
```

## 常见问题

- **网关无法访问**：检查 Gateway 的 selector、网关 Service 的 89 端口映射、外部地址及网络规则。
- **返回 404**：检查请求是否进入正确的网关，以及 VirtualService 的 Gateway 引用、hosts 和 `primary` 路由。
- **返回 503**：检查 Pod 就绪状态、Service 端点及网关到后端的 mTLS 策略。
- **ImagePullBackOff**：检查私有 ECR 镜像权限、标签和网络，或改用可访问的镜像。
- **停在 25%**：这是手动暂停点，确认新版本正常后执行 `promote`。
- **Argo CD 同步后权重被恢复**：检查 `ignoreDifferences` 和 `RespectIgnoreDifferences=true` 是否已应用到实际管理该 VirtualService 的 Application。
