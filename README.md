# Dapr 研讨会
>译注：本人刚开始学习， 如果发现任何翻译错误欢迎发[Issues](issues/new)指正。

此存储库通过几个动手作业来向你介绍Dapr。你将从一个简单的微服务应用程序开始，它包含多服务。在每个作业中，你将修改应用程序的一部分，使其与Dapr一起工作（或者如Donovan Brown所说“在应用程序上涂抹一些Dapr”）。你将会使用的Dapr构建块包括：

- 服务调用(Service invocation)
- 状态管理(State-management)
- 发布(Publish) 和 订阅(Subscribe)
- 绑定(Bindings)
- 机密管理(Secrets management)

由于Dapr可以从多种编程语言中使用，我们在研讨会中添加了3个版本的动手作业：

- C# (.NET)
- Java
- Python

在开始研讨会之前，请选择你想要使用的语言，并遵循该语言的说明。
你将在自托管模式下使用 Dapr。

## 领域

本次研讨会中的些动手作业，将使用安装在荷兰几条高速公路上的超速摄像头设置进行工作。下图是这次虚拟的环境设备概述：

![Speeding cameras](img/speed-trap-overview.png)

每个车道都有有 1 个入口摄像头和 1 个出口摄像头。当汽车通过入口摄像头时，会将汽车的车牌和时间戳记录下来。
当汽车通过出口摄像头时，系统也会记录此时间戳。然后，系统根据进出时间戳计算汽车的平均速度。如果发现超速违章，则会向中央罚款收集机构（或荷兰的 CJIB）发送消息。他们将检索车主的信息，并送他（她）罚款。

### 体系结构

为了方便在代码中模拟，定义了以下服务:

![Services](img/services.png)

- **摄像头模拟器(Camera Simulation)** 模拟过往车辆。
- **交通管制服务(Traffic Control Service)** 提供 2 个HTTP端点：`/entrycam` 和 `/exitcam`。这些端点可用于模拟通过入口或出口摄像头的汽车。
- **罚款收取服务(Fine Collection Service)** 提供 1 个HTTP端点：`/collectfine` 用来收取罚款。
- **车辆登记服务(Vehicle Registration Service)** 提供 1 个HTTP 端点： `/getvehicleinfo/{license-number}` 用来获取车辆和车主信息。

下面的序列图中描述模拟工作的方式：

<img src="img/sequence.png" alt="Sequence diagram" style="zoom:67%;" />

1. 摄像头模拟器生成一个随机车牌，并向TrafficControlService的`/entrycam`端点发送一条*VehiclerRegistered*消息（包含此车牌、随机进入车道（1-3）和时间戳）。
2. TrafficControlService 存储 *VehicleState*（车牌和进入时间戳）。
3. 等待一段时间后，摄像头模拟器将向TrafficControlService的`/exitcam`端口发送 *VehicleRegistered* 消息（包含第一步中的车牌，随机的退出车道(1-3)和退出时间戳）。
4. TrafficControlService 从存储的车辆实体中取回 *VehicleState* 。
5. TrafficControlService 使用车辆的进出时间戳计算平均速度。 同时为了方便审计也会保存包含有退出时间戳的*VehicleState* ，但这里为了使流程图清晰没有画出。
6. 如果平均速度超过限速，则TrafficControlService会调用FineCollectionService的 `/collectfine` 端点。 请求类型是*SpeedingViolation* ，它包含车牌、道路标识、违规超速的公里数和违规时间。
7. FineCollectionService 计算超速违章的罚款。
8. FineCollectionSerivice 使用超速车辆的车牌来调用VehicleRegistrationService服务的 `/vehicleinfo/{license-number}` 端点获取车辆和车主信息。
9. FineCollectionService 通过电子邮件向车主发送罚款。

此序列中描述的所有操作在执行过程中都记录到控制台，方便跟踪调试。

### 应用 Dapr 的结束状态

完成所有动手作业后，应用程序的体系结构会变成使用Dapr工作，它看起来如下所示：

<img src="img/dapr-setup.png" alt="Dapr setup" style="zoom:67%;" />

1. 对于FineCollectionService 和 VehicleRegistrationService 服务之间的请求相应通讯使用 **服务调用(service invocation)** 构建块。
2. 对于将超速违章发送到FineCollectionService，使用**发布和订阅(publish and subscribe)**构建块。使用RabbitMQ作为消息传出中介。
3. 对于存储车辆状态，使用 **状态管理(state management)** 构建块。这里使用Redis来存储状态。
4. 罚款通过电子邮件发送给超速车辆的车主。使用 Dapr的 SMTP **输出绑定(output binding)** 来发送电子邮件。
5. 使用 Dapr的MQTT **输入绑定(input binding)** 来发送模拟的车辆信息给 TrafficControlService。 这里使用 Mosquitto 作为MQTT中介。
6. FineCollectionService 需要保存连接STMP服务器凭据和用于计算罚款组件的许可密钥。这里使用**机密管理(secrets management)**构建块通过本地文件组件来获取凭据和许可密钥。

下面的序列图显示了解决方案如何使用 Dapr ：

<img src="img/sequence-dapr.png" alt="Sequence diagram with Dapr" style="zoom:67%;" />

> 如果在研讨会期间，你不知道作业的最终结果应该是什么，请返回此 README 查看最终结果。

## 研讨会的新手入门

### 先决条件

确保你的机器上安装了以下先决条件：

- Git ([下载](https://git-scm.com/))
- Visual Studio Code ([下载](https://code.visualstudio.com/download)) 并且安装以下扩展:
  - [REST Client](https://marketplace.visualstudio.com/items?itemName=humao.rest-client)
- Docker for desktop ([下载](https://www.docker.com/products/docker-desktop))
- [安装 Dapr CLI](https://docs.dapr.io/getting-started/install-dapr-cli/) 和 [初始化本地 Dapr ](https://docs.dapr.io/getting-started/install-dapr-selfhost/)

如前所述，你可以使用.NET 或 Java 执行作业。每个技术堆栈都有其自己的先决条件：

使用 .NET 作业:

- .NET 5 SDK ([下载](https://dotnet.microsoft.com/download/dotnet/5.0))
- [C# extension for Visual Studio Code](https://marketplace.visualstudio.com/items?itemName=ms-dotnettools.csharp)

使用 Java 作业:

- Java 16 或之上 ([下载](https://adoptopenjdk.net/?variant=openjdk16))
- [Visual Studio Code Extension Pack for Java](https://marketplace.visualstudio.com/items?itemName=vscjava.vscode-java-pack)
- Apache Maven 3.6.3 或之上是必须的; Apache Maven 3.8.1 是经过验证的 ([下载](http://maven.apache.org/download.cgi))
  - 通过运行 `mvn -version`命令 确保你的 Maven 使用正确的Java运行时。

使用 Python 作业:

- Python 3.9 ([下载](https://www.python.org/downloads/))
- Python Extension for Visual Studio Code ([下载](https://marketplace.visualstudio.com/items?itemName=ms-python.python))

说明中的所有脚本都是Powershell脚本。如果你正在开发 Mac，建议为 Mac 安装Powershell：

- Powershell for Mac ([安装指南](https://docs.microsoft.com/zh-cn/powershell/scripting/install/installing-powershell-core-on-macos?view=powershell-7.1))

### 版本

研讨会已测试了以下版本：

| 组件                        | 详细    |
| --------------------------- | ------- |
| Dapr runtime version        | v1.4.0  |
| Dapr CLI version            | v1.4.0  |
| .NET version                | .NET 5  |
| Java version                | Java 16 |
| Python version              | 3.9.6   |
| Dapr SDK for .NET version   | v1.4.0  |
| Dapr SDK for Java version   | v1.3.0  |
| Dapr SDK for Python version | v1.3.0  |

### 操作指南

每个作业都包含在此仓库的单独文件夹中。每个文件夹都包含你可以遵循的作业的描述。

**重要的是，你要按顺序完成所有作业，并且不要跳过任何作业。每个作业的说明取决于你已成功完成之前作业的事实。**

研讨会将为你提供的一个起点。此起点是应用程序可工作版本，其中服务使用普通 HTTP 相互通信，状态存储在内存中。研讨会中的每个作业都会让你在解决方案中添加一个 Dapr 构建基块。

每个作业都提供如何完成作业的说明。除作业1外，每个作业提供两个版本的说明：**DIY** 版本和**分步**版本。DIY 版本只说明你需要实现的结果，而无需提供进一步说明。完全取决于你在 Dapr 文档的帮助下实现目标。分步版本准确地描述了你需要在应用中逐步更改的内容。这要由你来选择一种方法。如果你选择 DIY 方法并遇到问题，你总是可以查看分步说明来提供一些帮助。

#### 集成终端

在研讨会期间，你应该只使用1个 VS Code实例工作。你会用到 VS Code 中的集成终端功能。所有终端命令都已在 Windows 系统上的VS Code 中的 Powershell终端进行过测试。如果你对 Linux 或 Mac 上的命令有任何问题，请创建issue或PR以添加适当的命令。

#### 防止端口冲突

在研讨会期间，你将在本地机器上运行解决方案中的服务。为了防止端口冲突，所有服务都监听不同的 HTTP 端口。在 Dapr 运行服务时，你需要额外的端口来与边车(sidecars)进行 HTTP 和 gRPC 通信。默认情况下，这些端口为"3500"和"50001"。但为了防止冲突，你将在作业中使用完全不同的端口号。如果你按照说明操作，服务将使用以下端口用于 Dapr 边车(sidecars)，以防止端口冲突：

| 服务                       | 应用端口         | Dapr 边车 HTTP 端口    | Dapr 边车 gRPC 端口    |
| -------------------------- | ---------------- | ---------------------- | ---------------------- |
| TrafficControlService      | 6000             | 3600                   | 60000                  |
| FineCollectionService      | 6001             | 3601                   | 60001                  |
| VehicleRegistrationService | 6002             | 3602                   | 60002                  |

如果你正在执行 DIY 方法，请确保使用上面表中指定的端口。

在 Dapr CLI 启动服务时，可以在命令行上指定端口。可使用以下命令行参数：

- `--app-port`
- `--dapr-http-port`
- `--dapr-grpc-port`

如果你的Windows上启动了Hyper-V ，则可能会遇到某些端口无法使用问题。这可能与 Hyper-V 的保留端口有关。你可以通过执行此命令来检查是否为此情况：

```powershell
netsh int ipv4 show excludedportrange protocol=tcp
```

如果你在输出中看到保留端口，可以在管理员模式终端中执行以下命令来修复它：

```powershell
dism.exe /Online /Disable-Feature:Microsoft-Hyper-V
netsh int ipv4 add excludedportrange protocol=tcp startport=6000 numberofports=3
netsh int ipv4 add excludedportrange protocol=tcp startport=3600 numberofports=3
netsh int ipv4 add excludedportrange protocol=tcp startport=60000 numberofports=3
dism.exe /Online /Enable-Feature:Microsoft-Hyper-V /All
```

#### 开始

现在是时候让你动手操作了，先第一个任务开始。研讨会的开始代码位于不同的存储库中。研讨会为每种编程语言都有提供一个单独的存储库：

- C#: [https://github.com/EdwinVW/dapr-workshop-csharp](https://github.com/EdwinVW/dapr-workshop-csharp)
- Java: [https://github.com/mthmulders/dapr-workshop-java](https://github.com/mthmulders/dapr-workshop-java)
- Python: [https://github.com/wmeints/dapr-workshop-python](https://github.com/wmeints/dapr-workshop-python)

按照下面的说明开始：

1. 将你要使用的编程语言的源代码存储库克隆到电脑的本地文件夹。例如：

   ```console
   git clone https://github.com/EdwinVW/dapr-workshop-csharp.git
   ```

   **从现在起，此文件夹称为"源代码"文件夹。**

2. 在开始作业之前，我建议你查看不同服务的代码。作业中使用的所有文件夹都是相对于源代码文件夹的指定相对路径。

3. 从 [作业 1](Assignment01/README.md)开始。

## 面向 .NET 开发人员的 Dapr

如果你想在做研讨会后更多地了解 Dapr，你可以阅读由这个研讨会的创建者合著的"面向 .NET 开发人员的 Dapr"一书。虽然这本书是针对.NET开发人员，但它涵盖了Dapr的所有概念和通用API。因此，对于使用不同技术堆栈的开发人员来说，它也应该很有用。

[下载PDF](https://aka.ms/dapr-ebook)
[在线阅读（有中文版本）](https://docs.microsoft.com/dotnet/architecture/dapr-for-net-developers/)

![Dapr for .NET Developers](img/dapr-for-net-devs-cover-thumb.png)
