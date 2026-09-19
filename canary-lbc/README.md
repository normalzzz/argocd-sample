# Canary：AWS Load Balancer Controller

本示例使用 Argo Rollouts 编排金丝雀发布，通过 AWS Load Balancer Controller（LBC）配置 ALB，将流量按权重分配给稳定版本和新版本。

## 资源与流量路径

| 文件 | 作用 |
| --- | --- |
| `namespace.yaml` | 创建 `canary-lab` 命名空间 |
| `rollout.yaml` | 定义 `rollouts-demo`，默认 4 个副本，初始镜像为 `argoproj/rollouts-demo:yellow` |
| `services.yaml` | 定义 root、stable、canary 三个 Service，服务端口为 80，容器端口为 8080 |
| `ingress.yaml` | 创建公网 ALB 入口，监听 HTTP **81** 端口，使用 IP 类型的 Target Group |

请求路径为：客户端 → ALB:81 → stable/canary Target Group → 对应版本的 Pod:8080。

Ingress 的 `rollouts-demo-root` 后端使用 `use-annotation`。Argo Rollouts 自动维护 `alb.ingress.kubernetes.io/actions.rollouts-demo-root` 注解，LBC 根据其中的权重更新 ALB 转发规则。直接访问 root Service 无法验证 ALB 的流量比例。详见 [Argo Rollouts ALB 集成文档](https://argo-rollouts.readthedocs.io/en/stable/features/traffic-management/alb/)。

## 前置条件

- Kubernetes 集群已安装 Argo Rollouts Controller 和 CRD，本机已安装 `kubectl` 及 `kubectl argo rollouts` 插件。
- 已安装 AWS Load Balancer Controller，配置好所需 IAM 权限、子网发现和 `alb` IngressClass；网络允许访问 ALB 的 81 端口及后端 Pod。
- 集群能够拉取示例镜像。
- 本目录与 `canary-istio` 使用相同的命名空间、Rollout 和 Service 名称。同一命名空间中一次部署一个示例；并行实验时需为其中一个示例统一修改命名空间，包括 Namespace 资源名称。

## 部署与访问

以下命令均在仓库根目录执行，适用于直接通过 kubectl 管理的实验环境。

```bash
kubectl apply -f canary-lbc/namespace.yaml
kubectl apply -f canary-lbc/services.yaml -f canary-lbc/ingress.yaml -f canary-lbc/rollout.yaml
kubectl argo rollouts get rollout rollouts-demo -n canary-lab --watch
```

首次部署会直接建立稳定版本；等待 Rollout 健康后再更新镜像，才能观察金丝雀步骤。查看 ALB 地址：

```bash
kubectl get ingress rollouts-demo -n canary-lab
```

等待 `ADDRESS` 出现且 Target Group 健康后访问：

```bash
ALB_HOST=$(kubectl get ingress rollouts-demo -n canary-lab -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
curl "http://${ALB_HOST}:81/"
```

也可以在浏览器打开同一地址，观察示例页面中的版本颜色。

## 触发发布

将镜像从 `yellow` 更新为可拉取的 `blue` 版本：

```bash
kubectl argo rollouts set image rollouts-demo rollouts-demo=argoproj/rollouts-demo:blue -n canary-lab
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

观察转发注解、Pod 和 Service，配合连续访问 ALB 验证发布；少量请求的实际分布可能偏离配置权重。

```bash
kubectl get ingress rollouts-demo -n canary-lab -o yaml
kubectl get pods,svc -n canary-lab
```

## 中止与回滚

发布过程中可中止，将流量切回稳定版本：

```bash
kubectl argo rollouts abort rollouts-demo -n canary-lab
```

中止不会恢复期望的镜像配置。直接使用 kubectl 管理时，可恢复上一个修订：

```bash
kubectl argo rollouts undo rollouts-demo -n canary-lab
```

有关首次部署、推进与中止行为，参见 [Argo Rollouts 基础操作](https://argo-rollouts.readthedocs.io/en/stable/getting-started/)。

## 使用 Argo CD

在 Argo CD 中创建 Application，仓库路径设为 `canary-lbc`，目标集群按环境选择，目标命名空间设为 `canary-lab`，同步需要的 Git 分支。仓库中的清单必须已推送到该分支。

通过 GitOps 管理时，修改 `rollout.yaml` 中的镜像，提交、推送并同步 Application 来发布；回滚时恢复 Git 中的镜像配置并同步。直接执行 `set image` 或 `undo` 会与 Git 中的配置产生差异，启用自动自愈时可能被恢复。

## 常见问题

- **Ingress 没有地址**：查看 `kubectl describe ingress rollouts-demo -n canary-lab`，检查 LBC 日志、IAM 权限、子网与 IngressClass 配置。
- **访问失败或返回 503**：确认使用 81 端口，检查安全组、Target Group 健康状态，以及 Pod 的 readiness probe 和 Service 端点。
- **停在 25%**：这是 `pause: {}` 的预期行为，需要执行 `promote`。
- **没有出现逐步放量**：确认已完成首次部署，并且本次修改改变了 Pod 模板，例如镜像标签。
- **ImagePullBackOff**：检查镜像地址、标签及集群到镜像仓库的网络连通性。
