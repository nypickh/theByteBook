# 10.5 使用 Argo CD 进行持续交付

Argo CD 是一个基于 Kubernetes 的声明式 GitOps 持续交付工具，主要用于 Kubernetes 集群中的应用管理和部署。它能够从 Git 仓库中获取应用的定义，并自动同步到集群中。

如图 10-10 所示，Argo CD 是通过 Kubernetes 控制器来实现的，它持续 watch 正在运行的应用程序，并将应用程序的实际状态与所需的目标状态（ Git 存储库中指定的）进行比较。如果应用程序的实际状态与目标状态有差异，则被认为是 OutOfSync 状态，Argo CD 会报告这些差异，同时提供工具来自动或手动将状态同步到期望的目标状态。

:::center
  ![](../assets/argocd_architecture.png)<br/>
  图 10-10 Argo CD 如何工作 [图片来源](https://argo-cd.readthedocs.io/en/stable/)
:::

接下来，笔者将演示在集群内安装 Argo CD 以及部署一个应用示例，介绍 Argo CD 的使用概况。

## 10.5.1 安装 Argo CD

先创建一个命名空间，然后通过 kubectl apply 安装 Argo CD 提供的 yaml 文件即可。

```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
安装之后，可以使用自带的 WebUI 进行交互操作，也可以额外安装 CLI ，出于演示的目的，这里使用自带 WebUI 进行交互。

默认情况下，Argo CD 的 WebUI 服务在集群内部并没有暴露出来，可以通过 LoadBalancer 或者 NodePort 类型的 Service、Ingress、Kubectl 端口转发等方式将 Argo CD 服务发布到 Kubernetes 集群外部。

如下，通过 NodePort 服务的方式暴露 Argo CD 到集群外部。

```
$ kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'
```

查找到 argocd-server 关联的 NodePort 端口，通过浏览器打开：https://localhost:35123/ 控制台。

初次访问需要登录，Argo CD 默认账户是 admin，帐号的初始密码是自动生成，并以明文的形式存储在 Argo CD 安装的命名空间中 argocd-initial-admin-secret 的 Secret 对象下的 password。

通过下面命令获取初始密码。
```
$ kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

## 10.5.2 部署应用

部署应用之前，我们先了解 Argo CD 定义的 Application 资源，它通过下面两个关键的属性将目标 Kubernetes 集群中的 namespace 与 Git 仓库中声明的期望状态连接起来：

- **Source**：指的是 Git 仓库中 Kubernetes 资源配置清单所在的位置，可以是原生的 Kubernetes 配置清单，也可以是 Helm Chart 或者 Kustomize 部署清单。
- **Destination**：通过 Server 指定 Kubernetes 集群以及相关的 namespace，这样 Argo CD 就知道将应用部署到 Kubernetes 集群中的哪个位置。

通过如下 yaml 文件，了解 Argo CD 对 Application 资源的定义。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: HEAD
    path: helm-guestbook
  destination:
    server: https://kubernetes.default.svc
    namespace: default
```

除了 Application 资源，Argo CD 也定义了 Project 资源，用来对 Application 分组，设置更细粒度的访问权限控制，实现多租户环境。

创建 Application 之后，应用状态为初始 OutOfSync 状态，此时也尚未创建任何 Kubernetes 资源，在控制台点击 “SYNC” 按钮进行同步部署，部署之后的状态如下图所示。

:::center
  ![](../assets/argocd-demo.png)<br/>
  图 10-11 Argo CD 应用部署示例
:::

如此，后续无论是通过 CI 触发更新 git 仓库中的编排文件，还是工程师直接修改，Argo CD 都会自动拉取最新的配置并应用到 Kubernetes 集群中。