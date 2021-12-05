
在这个环节中，我们将通过预先准备好的 CloudFormation 模板创建实验的基础网络环境和实验资源。AWS CloudFormation 是一种“基础设施即代码”，提供一种简单的方式，对一系列 AWS 资源进行建模，快速而又一致地对这些资源进行预置。

请通过以下链接下载本实验使用的 CloudFormation 模板至本地环境：[template.yaml](/template/template.yaml)。这个模板里面主要包含了 VPC、SageMaker Notebook。CloudFormation 包含的详细内容包括：

- 创建跨三个不同的 AZ 的 VPC
- 创建的 VPC 在每个 AZ 都包含 public、private、isolated 三个子网
- 公有（public）子网：
  - 该子网内的资源会暴露在互联网上，可被用户或客户端直接访问。通常用于部署 NAT Gateway, 堡垒机，ELB 负载均衡器等；
- 私有子网：
  - 该子网内的资源不能被用户或客户端直接访问，但可以通过 NAT Gateway 单向访问公网。通常用于部署业务服务器、容器集群、大数据集群等；
- 数据库子网：
  - 该子网内的资源不能被用户或客户端直接访问，也不能访问公网。通常仅用于部署对安全要求最高的数据库等。
- SageMaker Notebook 实例
- 用于 SageMaker Notebook 和其他托管服务使用的 IAM 角色


### 实验步骤

首先，导航到 CloudFormation 控制台，部署刚刚下载的模板：

- 通过此链接直接导航到 CloudFormation 控制台创建新堆栈的界面：[https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/create/template](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/create/template)
- 选择**Upload a template file**，然后点击**Chose file**，选择上一步中下载的模板
![](/images/upload_template.png)

- 上传完成后点击**Next**
- 在下一步中，列出了模板所需的参数。在 **Stack name** 选项中填入 **AlgoTradingWorkshop**，其他参数保持默认，然后点击**Next**：
![](/images/template_parameters.png)

- 直接点击**Next**到最终的审核步骤
- 下滑到最下方，勾选**I acknowledge that AWS CloudFormation might create IAM resources with custom names.**，然后点击右下方的**Create stack**按钮：
![](/images/create_stack.png)

大概等待5分钟左右，堆栈创建成功。成功后可以看到堆栈变成 **CREATE_COMPLETE** 状态。这时可以点开堆栈的 **Outputs** 页面查看堆栈创建的各类 AWS 资源 id：
![](/images/update_complete.png)

接下来我们可以在控制台中简单确认这个 CloudFormation 模板中创建的资源：

#### VPC

Amazon Virtual Private Cloud（VPC）允许您在 Amazon Web Services 云中预置一个逻辑隔离分区，并在虚拟网络中启动亚马逊云科技资源。在 VPC 中，您可以完全掌控您的虚拟联网环境。
![](/images/vpc_3az.png)

VPC 类似于传统网络，包括以下概念：

- CIDR 段：用于 IP 地址分配
- 子网：从 CIDR 段中划出的 VPC 中可用的一系列 IP 地址范围
- 路由表：一组称为路由的规则，规定将网络流量定向到何处
- 网关：VPC 中资源和互联网之间的网关
- 安全组：动态的网络访问控制规则

您可以导航到 VPC 控制台点击细项查看详细的网络配置：[https://console.aws.amazon.com/vpc/home?region=us-east-1](https://console.aws.amazon.com/vpc/home?region=us-east-1)
![](/images/vpc_dashboard.png)

#### SageMaker Notebook 实例

Amazon SageMaker Notebook 实例是一个机器学习 (ML) 计算实例，运行 Jupyter Notebook 应用程序。Jupyter Notebook 提供一种网页形式的简单 IDE，可以在网页中交互式地编写代码和运行代码，直接返回逐段代码的运行结果。同时 Notebook 中还可以穿插必要的说明文档和图片，便于对代码进行说明和解释。SageMaker Notebook 实例还提供了多种内核，便于进行机器学习开发。详情可以参考：[https://docs.aws.amazon.com/zh_cn/sagemaker/latest/dg/nbi.html](https://docs.aws.amazon.com/zh_cn/sagemaker/latest/dg/nbi.html)

您可以导航到 SageMaker 控制台确认 CloudFormation 创建好的 SageMaker Notebook 实例：[https://us-east-1.console.aws.amazon.com/sagemaker/home?region=us-east-1#/notebook-instances](https://us-east-1.console.aws.amazon.com/sagemaker/home?region=us-east-1#/notebook-instances)。点击右侧的 **Open JupyterLab** 按钮就可以直接跳转到 Jupyter Notebook的使用界面：
![](/images/sagemaker_notebook.png)