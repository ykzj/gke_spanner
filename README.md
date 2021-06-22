#Google Kubernetes Engine (GKE) 和 Cloud Spanner 实践

## 谷歌云项目选择

创建一个动手实验的 Google Cloud 项目，选择您的 Google Cloud 项目并点击 **Start**。
**请尽量新建一个项目。**

<walkthrough-project-setup>
</walkthrough-project-setup>

##【解说】实操内容

### ** 内容和目的 **

在本次动手实践中，对于那些从未接触过Google Kubernetes Engine的人，我们将从创建 Kubernetes 集群开始，构建、部署和访问容器。

使用以主题访问 Cloud Spanner 的 Web 应用程序，尝试使用 Workload Identity 在没有服务帐户密钥的情况下访问 Cloud Spanner。

通过此动手操作，目的是了解使用 Google Kubernetes Engine 进行应用程序开发的第一步。

下图显示了动手系统配置（最终配置）。

![](https://storage.googleapis.com/egg-resources/egg3-2/public/gke/0-1.png)

## 启用 Google API 并创建 Kubernetes 集群

我认为 Cloud Shell 和编辑器屏幕当前已打开，但如果您尚未打开 Google Cloud Console (https://console.cloud.google.com/)，请打开控制台屏幕。请打开它。

### ** 设置项目使用**
```bash
gcloud config set project {{project-id}}
```

将 GOOGLE_CLOUD_PROJECT 设置为项目ID。

```bash
export GOOGLE_CLOUD_PROJECT=$(gcloud config list project --format "value (core.project)")
```

### ** API 激活 **

使用以下命令启用动手 Google API：

```bash
gcloud services enable cloudbuild.googleapis.com \
  sourcerepo.googleapis.com \
  containerregistry.googleapis.com \
  cloudresourcemanager.googleapis.com \
  container.googleapis.com \
  stackdriver.googleapis.com \
  cloudtrace.googleapis.com \
  cloudprofiler.googleapis.com \
  logging.googleapis.com
```

### ** 创建 Kubernetes 集群 **

启用 API 后，创建 Kubernetes 集群。
执行以下命令。

```bash
gcloud container --project "$GOOGLE_CLOUD_PROJECT" clusters create "cluster-1" \
  --zone "asia-northeast1-a" \
  --enable-autoupgrade \
  --image-type "COS" \
  --enable-ip-alias \
  --workload-pool=$GOOGLE_CLOUD_PROJECT.svc.id.goog
```

创建 Kubernetes 集群需要几分钟时间。

## 用于动手操作的架构描述

在这个动手实践中，我们将使用三个表，如下所示。这里假设 Cloud Spanner 在游戏开发中被用作后端数据库，并且表示相当于管理游戏玩家信息和物品信息的表。

![Schema](https://storage.googleapis.com/egg-resources/egg3/public/1-1.png "这次使用的Schema")

这个表的DDL如下：等我实际建表的时候再贴一下这个DDL。

```sql
CREATE TABLE players（
player_id STRING (36) NOT NULL，
name STRING(MAX) NOT NULL，
level INT64 NOT NULL，
money INT64 NOT NULL，
) PRIMARY KEY(player_id);
```

```sql
CREATE TABLE items (
item_id INT64 NOT NULL,
name STRING(MAX) NOT NULL,
price INT64 NOT NULL,
) PRIMARY KEY(item_id);
```

```sql
CREATE TABLE player_items (
player_id STRING(36) NOT NULL,
item_id INT64 NOT NULL,
quantity INT64 NOT NULL,
FOREIGN KEY(item_id) REFERENCES items(item_id)
) PRIMARY KEY(player_id, item_id),
INTERLEAVE IN PARENT players ON DELETE CASCADE;
```


## 创建一个 Cloud Spanner 实例

我认为 Cloud Shell 和编辑器屏幕当前已打开，但如果您尚未打开 Google Cloud Console (https://console.cloud.google.com/)，请打开控制台屏幕。请打开它。


### ** 创建 Cloud Spanner 实例 **

![](https://storage.googleapis.com/egg-resources/egg3/public/2-1.png)

1. 从导航菜单中选择“Spanner”

![](https://storage.cloud.google.com/egg-resources/egg3-2/public/2-2.png)

2.选择“创建实例”

### ** 输入信息**

![](https://storage.googleapis.com/egg-resources/egg3/public/2-3.png)

设置以下内容并选择“创建”。
1. 实例名称：dev-instance
2. 实例ID：dev-instance
3. 选择“地区”
4. 选择“asia-northeast1（东京）”
5. 节点分配：1
6. 选择“创建”

### ** 实例创建完成**
将显示以下屏幕并完成创建。
让我们看看你能看到什么样的信息。

![](https://storage.googleapis.com/egg-resources/egg3-2/public/2-4.png)

## 创建一个表

### ** 创建数据库 **

由于我们只创建了 Cloud Spanner 的一个实例，因此我们将创建一个数据库和表。

您可以在一个 Cloud Spanner 实例中创建多个数据库。

![](https://storage.googleapis.com/egg-resources/egg3-2/public/5-1.png)

![](https://storage.googleapis.com/egg-resources/egg3-2/public/5-2.png)

1. 选择dev-instnace换屏
2. 选择创建数据库

### ** 输入数据库名称 **

![](https://storage.googleapis.com/egg-resources/egg3-2/public/5-3.png)
输入“player-db”作为名称。


### ** 数据库架构定义 **

![](https://storage.googleapis.com/egg-resources/egg3-2/public/5-4.png)
转至定义架构的屏幕。

将以下 DDL 直接粘贴到区域 1 中。

```sql
CREATE TABLE players (
player_id STRING(36) NOT NULL,
name STRING(MAX) NOT NULL,
level INT64 NOT NULL,
money INT64 NOT NULL,
) PRIMARY KEY(player_id);

CREATE TABLE items (
item_id INT64 NOT NULL,
name STRING(MAX) NOT NULL,
price INT64 NOT NULL,
) PRIMARY KEY(item_id);

CREATE TABLE player_items (
player_id STRING(36) NOT NULL,
item_id INT64 NOT NULL,
quantity INT64 NOT NULL,
FOREIGN KEY(item_id) REFERENCES items(item_id)
) PRIMARY KEY(player_id, item_id),
INTERLEAVE IN PARENT players ON DELETE CASCADE;
```

2. 选择创建开始创建表。

### ** 数据库创建完成**

![](https://storage.googleapis.com/egg-resources/egg3-2/public/5-5.png)

如果一切顺利，将创建数据库并同时生成三个表。

## 准备 Cloud Spanner 连接客户端 

### ** 构建一个写入 Cloud Spanner 的应用程序 **

首先，让我们创建一个使用客户端库的 Web 应用程序。

在 Cloud Shell 中，我认为您在这次使用的 `gke_spanner` 目录中。
有一个名为 spanner 的目录，因此请移至该目录。

```bash
cd spanner
```

让我们检查一下目录的内容。

```bash
ls -la
```

您将找到名为“main.go”和“pkg/”的文件和目录。
您还可以在 Cloud Shell 编辑器中看到这一点。

让我们从编辑器中打开 `spanner/main.go` 并检查内容。

```bash
cloudshell edit main.go
```

![](https://storage.googleapis.com/egg-resources/egg3-2/public/4-1.png)

此应用程序是用于在我们这次创建的游戏中注册新用户的应用程序。
执行后，Web 服务器将启动。
当您向 Web 服务器发送 HTTP 请求时，用户 ID 会自动编号，并将新用户信息写入 Cloud Spanner 播放器表。

下面的代码是实际执行此操作的部分。

```go
func (h *spanHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
        ...
    p := NewPlayers()
    // get player infor from POST request
    err := GetPlayerBody(r, p)
    if err != nil {
      LogErrorResponse(err, w)
      return
    }
    // use UUID for primary-key value
    randomId, _ := uuid.NewRandom()
    // insert a recode using mutation API
    m := []*spanner.Mutation{
      spanner.InsertOrUpdate("players", tblColumns, []interface{}{randomId.String(), p.Name, p.Level, p.Money}),
    }
    // apply mutation to cloud spanner instance
    _, err = h.client.Apply(r.Context(), m)
    if err != nil {
      LogErrorResponse(err, w)
      return
    }
    LogSuccessResponse(w, "A new Player with the ID %s has been added!\n", randomId.String())}
        ...

```

接下来，让我们构建用这种 Go 语言编写的源代码。

然后使用以下命令构建。在第一次构建时，会下载依赖库，因此需要一些时间。
下载和构建将在大约 1 分钟内完成。

```bash
go build -o player
```

让我们检查是否有构建的二进制文件。
您应该已经创建了一个名为“player”的二进制文件。您现在有一个连接到 Cloud Spanner 并写入的应用程序。

```bash
ls -la
```

** 附录）如何在不构建的情况下运行二进制文件 **

您还可以使用以下命令在不构建二进制文件的情况下运行应用程序：

```bash
go run *.go
```



## 写入数据：应用程序

### ** 从 Web 应用程序添加玩家数据 **

如果未设置，请将 GOOGLE_CLOUD_PROJECT 设置为项目ID。

```bash
export GOOGLE_CLOUD_PROJECT=$(gcloud config list project --format "value (core.project)")
```
然后运行你刚刚构建的 `player` 命令。

```bash
./player
```

如果输出如下日志，则表示Web服务器正在运行。

```bash
2021/06/14 01:14:25 Defaulting to port 8080
2021/06/14 01:14:25 Listening on port 8080
```

如果输出如下日志，说明没有设置`GOOGLE_CLOUD_PROJECT`的环境变量。

```bash
2021/06/14 18:05:47 'GOOGLE_CLOUD_PROJECT' is empty. Set 'GOOGLE_CLOUD_PROJECT' env by 'export GOOGLE_CLOUD_PROJECT=<gcp project id>'
```

设置环境变量并重试。
```bash
export GOOGLE_CLOUD_PROJECT=$(gcloud config list project --format "value (core.project)")
```

或者
```bash
GOOGLE_CLOUD_PROJECT={{project-id}} ./player
```

此 Web 服务器在接受针对特定路径的 HTTP 请求时会注册、更新和删除新的玩家信息。
现在让我们向 Web 服务器发送创建新玩家的请求。
在与运行“player”的控制台不同的选项卡中，使用以下命令发送 HTTP POST 请求。

```bash
curl -X POST -d '{"name": "testPlayer1", "level": 1, "money": 100}' localhost:8080/players
```

发送 `curl` 命令应该返回类似于以下的结果：

```bash
A new Player with the ID 78120943-5b8e-4049-acf3-b6e070d017ea has been added!
```

如果你得到错误 **`invalid character'\\' looking for opening of value`**，尝试删除反斜杠 (\\) 字符并在运行 curl 命令时不换行运行它。
这个ID (`78120943-5b8e-4049-acf3-b6e070d017ea`) 是应用程序自动生成的用户 ID，从数据库的角度来看，它是玩家表的主键。
记下手头生成的 ID，因为它将在后续练习中使用。

##检查GKE集群创建是否完成

### 检查图形用户界面

创建 Kubernetes 集群后，您可以在 Management Console-Kubernetes Engine-Clusters 中看到它。

![](https://storage.googleapis.com/egg-resources/egg3-2/public/gke/1-1.png)

通过创建 Kubernetes 集群，当前系统配置如下所示：

![](https://storage.googleapis.com/egg-resources/egg3-2/public/gke/1-2.png)

### ** 命令设置 **

执行以下命令，设置命令执行环境。

```bash
gcloud config set project {{project-id}}
gcloud config set compute/zone asia-northeast1-a
gcloud config set container/cluster cluster-1
gcloud container clusters get-credentials cluster-1
```

上述命令中的以下命令带来了本地操作 Kubernetes 集群（Cloud Shell）所需的凭据。

```bash
gcloud container clusters get-credentials cluster-1
```

执行以下命令检查与 Kubernetes Cluster 的通信并检查版本。

```bash
kubectl version
```

如果得到如下结果，就可以和Kubernetes Cluster进行通信了（每个Client和Server都输出version）。

```
Client Version: version.Info{Major:"1", Minor:"21", GitVersion:"v1.21.1", GitCommit:"5e58841cce77d4bc13713ad2b91fa0d961e69192", GitTreeState:"clean", BuildDate:"2021-05-12T14:18:45Z", GoVersion:"go1.16.4", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"19+", GitVersion:"v1.19.9-gke.1900", GitCommit:"008fd38bf3dc201bebdd4fe26edf9bf87478309a", GitTreeState:"clean", BuildDate:"2021-04-14T09:22:08Z", GoVersion:"go1.15.8b5", Compiler:"gc", Platform:"linux/amd64"}
```

通过设置命令，当前系统配置如下。

![](https://storage.googleapis.com/egg-resources/egg3-2/public/gke/1-3.png)

## 创建一个 Docker 容器镜像

### **构建 Docker 容器镜像**

创建 Docker 容器镜像时，准备一个 Dockerfile。
Dockerfile 存储在 spanner 目录中。

```bash
cd ~/gke_spanner
```

您可以使用以下命令在 Cloud Shell 编辑器中检查文件的内容。

```bash
cloudshell edit spanner/Dockerfile
```

返回终端并使用以下命令开始构建 Docker 容器映像：

```bash
docker build -t asia.gcr.io/${GOOGLE_CLOUD_PROJECT}/spanner-app:v1 ./spanner
```

![](https://storage.googleapis.com/egg-resources/egg3-2/public/gke/2-1.png)

构建 Docker 容器映像需要几分钟时间。

您可以使用以下命令在本地（Cloud Shell）查看评论图像：

```bash
docker image ls
```

您应该会看到如下所示的容器图像。

```
REPOSITORY                                       TAG  IMAGE ID       CREATED         SIZE
asia.gcr.io/<GOOGLE_CLOUD_PROJECT>/spanner-app   v1   8952a9a242f5   5 minutes ago   23.2MB
```

### **推送 Docker 容器镜像**

构建容器镜像后，将构建的镜像上传到 Google Container Registry。

以下命令将 `docker push` 设置为使用 `gcloud` 凭据。

```bash
gcloud auth configure-docker
```

使用以下命令推送到 Google Container Registry：
（第一次需要时间）

```bash
docker push asia.gcr.io/${GOOGLE_CLOUD_PROJECT}/spanner-app:v1
```

使用以下命令验证它是否已推送到 Google Container Registry：

```bash
gcloud container images list-tags asia.gcr.io/${GOOGLE_CLOUD_PROJECT}/spanner-app
```

您还可以从管理控制台 > 工具 > Container Registry 检查它。

![](https://storage.googleapis.com/egg-resources/egg3-2/public/gke/2-2.png)

随着容器镜像加入Google Container Registry，目前系统配置如下：

![](https://storage.googleapis.com/egg-resources/egg3-2/public/gke/2-3.png)

## 工作负载身份设置

### **确保启用工作负载标识**

确保在 GKE 集群上启用了 Workload Identity。
执行以下命令。

```bash
gcloud container clusters describe cluster-1 \
  --format="value(workloadIdentityConfig.workloadPool)" --zone asia-northeast1-a
```

如果输出是`<GOOGLE_CLOUD_PROJECT> .svc.id.goog`，则设置正确。

此外，请确保在 NodePool 上启用了 Workload Identity，与集群分开。
执行以下命令。

```bash
gcloud container node-pools describe default-pool --cluster=cluster-1 \
  --format="value(config.workloadMetadataConfig.mode)" --zone asia-northeast1-a
```

如果它输出`GKE_METADATA`，则设置正确。

### **创建服务帐户**

创建一个使用 Workload Identity 的服务帐户。
创建 Kubernetes 服务帐户 (KSA) 和 Google 服务帐户 (GSA)。

使用以下命令在 Kubernetes 集群上为 Kubernetes 创建服务帐户 (KSA)：

```bash
kubectl create serviceaccount spanner-app
```

使用以下命令创建 Google 服务帐户 (GSA)：

```bash
gcloud iam service-accounts create spanner-app
```

这次使用的 Spanner App 是访问 Cloud Spanner 的 Web 应用程序。
您必须授予 `roles/spanner.databaseUser` 权限，以便您可以操作数据（添加、更新、删除等）到 Cloud Spanner 数据库。
将以下命令添加到您的服务帐户以允许对 Cloud Spanner 数据库进行数据操作。

```bash
gcloud projects add-iam-policy-binding $GOOGLE_CLOUD_PROJECT \
  --role "roles/spanner.databaseUser" \
  --member serviceAccount:spanner-app@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com
```

此时，我们只会将 KSA 和 GSA 创建为单独的。链接 KSA 和 GSA 的工作如下。

### **链接服务帐户**

链接以便Kubernetes Service Account可以借用Google Service Account的权限。
使用以下命令进行链接。

```bash
gcloud iam service-accounts add-iam-policy-binding \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:$GOOGLE_CLOUD_PROJECT.svc.id.goog[default/spanner-app]" \
  spanner-app@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com
```

使用以下命令将 Google 服务帐户信息添加到 Kubernetes 服务帐户注释中：

```bash
kubectl annotate serviceaccount spanner-app \
  iam.gke.io/gcp-service-account=spanner-app@${GOOGLE_CLOUD_PROJECT}.iam.gserviceaccount.com
```

## 创建 Kubernetes 部署

从这里，我们将创建资源以在 Kubernetes 集群上运行。

### **编辑部署清单文件** 

清单文件存储在 k8s 目录中。
您可以使用以下命令在 Cloud Shell 编辑器中检查文件的内容。

```bash
cloudshell edit k8s/spanner-app-deployment.yaml
```

**将文件中的项目id改为自己的{{project-id}}**

![](https://storage.googleapis.com/egg-resources/egg3-2/public/gke/3-1.png)


### **创建部署**

使用以下命令在 Kubernetes 集群上创建部署：

```bash
kubectl apply -f k8s/spanner-app-deployment.yaml
```

使用以下命令检查创建的 Deployment 和 Pod。

```bash
kubectl get deployments
```

```bash
kubectl get pods -o wide
```

通过创建部署，当前系统配置如下所示：

![](https://storage.googleapis.com/egg-resources/egg3-2/public/gke/4-1.png)

**附录）kubectl 命令参考第 1 部分**

 * 当您想检查列表中的 pod 时
```bash
kubectl get pods
```

 * 当你想知道一个 Pod 的 IP 地址和执行节点时
```bash
kubectl get pods -o wide
```

 * 当您想查看 Pod 的详细信息时
```bash
kubectl describe pods <pod name>
```

 * 当你想在 YAML 中获取 Pod 的定义时
```bash
kubectl get pods <pod name> -o yaml
```


## 创建 Kubernetes 服务（发现）

### **创建服务**

清单文件存储在 k8s 目录中。
您可以使用以下命令在 Cloud Shell 编辑器中检查文件的内容。

```bash
cloudshell edit k8s/spanner-app-service.yaml
```

使用以下命令在 Kubernetes 集群上创建服务：

```bash
kubectl apply -f k8s/spanner-app-service.yaml
```

使用以下命令检查创建的服务：

```bash
kubectl get svc
```

由于正在创建网络负载平衡器 (TCP)，分配外部 IP 地址 (EXTERNAL-IP) 需要一些时间（分钟）。
以下命令始终使用 -w 标志监视更改（Ctrl + C 中断）

```bash
kubectl get svc -w
```


**附录）kubectl 命令参考第 2 部分**

 * 当您想在列表中检查服务时
```bash
kubectl get services
```

 * 当您想知道 Service 将通信路由到的 pod (IP) 时
 * 端点资源由服务资源自动管理
```bash
kubectl get endpoints
```

 * 当您想查看服务的详细信息时
```bash
kubectl describe svc <service name>
```

 * 当你想在 YAML 中获取 Service 的定义时
```bash
kubectl get svc <service name> -o yaml
```

### **访问服务**

创建一个带有 `Type: LoadBalancer` 的服务将发出一个网络负载均衡器。
您可以从【管理控制台】-【网络】-【网络服务】-【负载均衡】中查看。

![](https://storage.googleapis.com/egg-resources/egg3-2/public/gke/5-1.png)

让我们指定负载均衡器的外部 IP 地址并实际访问它。
使用以下命令检查服务的外部 IP 地址：

```bash
kubectl get svc
```

在以下输出中，“EXTERNAL-IP”是外部 IP 地址。
请在您自己的环境中阅读它，因为它因环境而异。

```
NAME         TYPE           CLUSTER-IP    EXTERNAL-IP    PORT(S)          AGE
spanner-app  LoadBalancer   10.0.12.198   203.0.113.245  8080:30189/TCP   58s
kubernetes   ClusterIP      10.56.0.1     <none>         443/TCP          47m
```

使用以下 curl 命令访问以获取玩家信息。
将 <EXTERNAL IP> 部分更改为上面确认的 IP 地址。

```bash
curl <EXTERNAL IP>:8080/players
```

如果您可以通过 curl 命令访问它，那么服务资源和部署（和 Pod）工作正常。

我们还确认工作负载身份功能允许我们在没有服务帐户密钥的情况下访问 Cloud Spanner。

![](https://storage.googleapis.com/egg-resources/egg3-2/public/gke/5-2.png)

### **操作 Spanner 应用程序**

 * 添加了新玩家（玩家 ID 将在此之后自动编号）
```bash
curl -X POST -d '{"name": "testPlayer99", "level": 9, "money": 10000}' <EXTERNAL IP>:8080/players
```

如果你得到错误 **`invalid character'\\' looking for opening of value`**，尝试删除反斜杠 (\\) 字符并在运行 curl 命令时不换行运行它。

 * 获取玩家列表
```bash
curl <EXTERNAL IP>:8080/players
```

 * 播放器更新（playerId 应相应更改）
```bash
curl -X PUT -d '{"playerId":"afceaaab-54b3-4546-baba-319fc7b2b5b0","name": "testPlayer1", "level": 2, "money": 200}' <EXTERNAL IP>:8080/players
```

 * 删除玩家（根据需要更改玩家 ID）
```bash
curl -X DELETE http://<EXTERNAL IP>:8080/players/afceaaab-54b3-4546-baba-319fc7b2b5b0
```

**附录）使用 kubectl 进行故障排除**

如果您在 Kubernetes 上遇到资源问题，请使用 `kubectl describe` 命令进行检查。
当您创建、更新或删除资源时，Kubernetes 会发出“事件”资源。您可以使用 kubectl describe 命令在一小时内看到“事件”。

```bash
kubectl describe <resource name> <object name>
kubectl describe deployments hello-node
```

此外，请检查应用程序日志以查看应用程序中是否存在任何错误。
使用 `kubectl logs` 命令查看应用程序日志。

```bash
kubectl logs -f <pod name>
```

＃ 清理

做完所有的练习，删除资源。

 1. 删除负载均衡器。如果不先删除Disk和LB，即使删除了整个Kubernetes集群它们也会保留，所以先删除它们。
```bash
kubectl delete svc spanner-app
```

 2. 删除Container Registry中存储的Docker镜像
```bash
gcloud container images delete asia.gcr.io/$PROJECT_ID/spanner-app:v1 --quiet
```

 3. 删除Kubernetes集群
```bash
gcloud container clusters delete "cluster-1" --zone "asia-northeast1-a"
```

对于 `Do you want to continue (Y/n)?`，输入 `y`。

 4. 删除您的 Google 服务帐户
```bash
gcloud iam service-accounts delete spanner-app@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com
```

对于 `Do you want to continue (Y/n)?`，输入 `y`。

 5. 删除 Cloud Spanner 实例。
```bash
gcloud spanner instances delete dev-instance
```

对于 `Do you want to continue (Y/n)?`，输入 `y`。

＃**谢谢你！**

这样就完成了 Google Kubernetes Engine 和 Cloud Spanner 的动手操作。