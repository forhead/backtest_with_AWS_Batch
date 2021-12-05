## 通过AWS Batch并行执行回测任务

在上一节的实验中，我们开发了趋势跟随策略（15日均线）策略，并且在BTCUSDT的交易对中获得了还算不错的收益，那么我们一定会想要把这个策略应用到其他的币对中使用。这时候，我们就可以利用AWS Batch来并行执行回测任务，根据任务结果来确认同样的交易策略是否可以用到其他的币对当中。

下面的实验会让大家在Jupyter Notebook中一步步通过代码，来实现使用AWS Batch，并行回测的任务。

### AWS Batch介绍
AWS Batch可帮助您运行任意规模的批量计算工作负载。AWS Batch将根据工作负载的数量和规模自动预配置计算资源并优化工作负载分配。通过使用AWS Batch，不再需要安装或管理批量计算软件，从而使您可以将时间放在分析结果和解决问题上。

要使用AWS Batch，您需要了解以下概念：

* compute environment
计算环境是一组用于运行任务的托管或非托管计算资源。使用托管计算环境，您可以在多个详细级别指定所需的计算类型（Fargate 或 EC2）。您可以设置使用特定类型 EC2 实例的计算环境，例如c5.2xlarge或者m5.10xlarge。或者，您可以选择仅指定要使用最新的实例类型。您还可以指定环境的最小、所需和最大 vCPUs 数，以及您愿意为 Spot 实例支付的金额占按需实例价格的百分比以及目标 VPC 子网集。AWS Batch根据需要高效地启动、管理和终止计算类型。您还可以管理自己的计算环境。因此，您负责在 Amazon ECS 集群中设置和扩展实例。AWS Batch会为您创建。有关更多信息，请参阅 计算环境。

* queue
当您提交AWS Batch作业时，会将其提交到特定的任务队列中，然后作业驻留在那里直到被安排到计算环境中为止。您将一个或多个计算环境与一个作业队列关联。您还可以为这些计算环境甚至跨任务队列本身分配优先级值。例如，您可以有一个高优先级队列用以提交时间敏感型任务，以及一个低优先级队列供可在计算资源较便宜时随时运行的任务使用。

* Job Definition
Job Definition指定作业的运行方式。您可以把任务定义看成是任务中的资源的蓝图。您可以为您的任务提供 IAM 角色，以提供对其他AWS资源的费用。您还可以指定内存和 CPU 要求。任务定义还可以控制容器属性、环境变量和持久性存储的挂载点。任务定义中的许多规范可以通过在提交单个任务时指定新值来覆盖。

* Jobs
提交到 AWS Batch 的工作单位 (如 shell 脚本、Linux 可执行文件或 Docker 容器映像)。会作为一个容器化应用程序运行在AWS Fargate或 Amazon EC2 上，使用您在*Job Definition*中指定的参数。任务可以按名称或按 ID 引用其他任务，并且可以依赖于其他任务的成功完成。

通过使用AWS Batch，只需提交经过改造的代码，并行的执行大量的任务，在任务结束之后，Batch会自动的关闭启用的资源，完全的利用的云的弹性特性。

### 
- 下载本环节的 Notebook 代码到本地：[parallel_backtest.ipynb](/notebook/parallel_backtest.ipynb)
- 回到 JupyterLab 界面，点击左上角的 **Upload Files** 按钮：
![](/images/upload_notebook.png)

- 选择刚刚下载的 **parallel_backtest.ipynb** 文件并上传。上传成功后打开 Notebook：
![](/images/aws_batch_job_upload.png)

依次按照notebook的文档，把上一个实验章节中的代码做容器化改造，并且推送到AWS ECR，供下一步的AWS Batch调用

### Batch环境创建
请打开以下网页 [AWS Batch](https://console.aws.amazon.com/batch/home?region=us-east-1#compute-environments)， 按照步骤创建Batch环境
#### 创建 compute environment
* 点击**Create**按钮，创建 compute environment
![](/images/create_compute_env_1.png)
* service role选择默认的 *Batch service-linked role*，compute environment name输入backtestenv-<你的姓名拼音>
![](/images/create_compute_env_2.png)
* 选择fargate，无需再考虑虚拟机的运维工作
![](/images/create_compute_env_3.png)
* 网络选择我们实验中创建的vpc，在这里是 *AlgoTradingWorkshop*
![](/images/create_compute_env_4.png)
* 最后点击**Create computer environment**按钮，确认该计算环境的状态为VALID以后，再进行下面的操作。
![](/images/create_compute_env_5.png)

#### 创建 job queue
* 选择左侧的Job queues菜单，然后在右边点击**Create**按钮。
![](/images/create_job_queue_1.png)
* 名称输入backtestqueue-<你的姓名拼音>，然后在 compute environment中选择上一步创建的计算环境。
![](/images/create_job_queue_2.png)
![](/images/create_job_queue_3.png)
* 最后点击**Create**按钮，并确认该job的状态变为VALID。
![](/images/create_job_queue_4.png)

#### 创建 job definition
* job definition会用到我们在jupyter notebook中推送的容器镜像，通过以下地址找到ECR的镜像 [ECR](https://us-east-1.console.aws.amazon.com/ecr/repositories?region=us-east-1)，拷贝镜像地址
![](/images/ecr_repo_image_url.png)
* 接着回到[AWS Batch](https://console.aws.amazon.com/batch/home?region=us-east-1#job-definition)的控制台，创建job definition。选择左侧的Job definitions菜单选项，并在右侧点击**Create**按钮。
![](/images/create_job_def_1.png)
* 名称输入：btc-usdt-job-def-<你的姓名拼音>。选择Fargate模式。“Job attempts”输入1。
![](/images/create_job_def_2.png)
* 把ECR镜像地址写入 *Image*选项中，并且输入Command
```
python backtest.py salen-datalab dwd/fulldata/btcusd/hour/part-00000-856800ad-3133-4334-ad10-537de18b6879-c000.snappy.parquet
```
![](/images/create_job_def_3.png)

* 选择默认的Task Role
![](/images/create_job_def_4.png)

* 在 *Additional configuration* 中选择默认的 *ecsTaskExecutionRole* 
![](/images/create_job_def_5.png)
* 最后点击**Create**按钮。

### 提交任务
* 接下来，通过定义好的Job Definition提交一个任务，计算BTCUSDT的回测结果。在“Job definitions”界面上，选中你刚才创建的job definition，然后点击**Submit new job**按钮，从而提交一个新的任务。
![](/images/submit_job_btc_1.png)
* Job run-time中的name输入：btcusdtjob-<你的姓名拼音>。“Job queue”下拉框里选择你前面创建的backtestqueue。“Execution timeout”输入300。
![](/images/submit_job_btc_2.png)
* 其他保留缺省值，点击**Submit**按钮。提交后，等待任务成功。
![](/images/submit_job_btc_3.png)
* 点击任务，可以看到Log Stream
![](/images/submit_job_btc_result_1.png)
* 进入后，可以看到程序的输出，我们也可以根据自己的需要，把结果输出到dynamodb，mysql或者s3
![](/images/submit_job_btc_result_2.png)

* 我们再次提交一个eth的任务。按照前面的操作，选择我们刚才创建的job definition，然后点击**Submit new job**。
在Job run-time中的name输入：eth-usdt-job-<你的姓名拼音>。“Job queue”下拉框里选择你前面创建的backtestqueue。“Execution timeout”输入300。
![](/images/submit_job_eth_3.png)
* 在Command代码框中输入：
```
python backtest.py salen-datalab dwd/fulldata/ethusd/hour/part-00000-3141f899-6907-4349-b97f-ef42743c018c-c000.snappy.parquet
```
![](/images/submit_job_eth_1.png)
* 其他保留缺省值，点击**Submit**按钮。提交后，等待任务成功。
![](/images/submit_job_eth_4.png)
* 同样我们也可以看到结果输出
![](/images/submit_job_eth_result_1.png)
![](/images/submit_job_eth_result_2.png)

### 并行提交任务
* 以上演示了如何提交单个任务，我们也可以通过程序提交多个已经定义好的job definition的任务，然后等待结果

* 为了方便调用，我们把回测任务的执行指令都定义在job definition中，然后可以直接通过程序提交即可，所以需要定义一个针对eth-usdt的任务，过程和步骤同上接着只需要程序调用即可。

* 按照之前的方式，创建一个新的Job definition，名称输入：eth-usdt-job-def-<你的姓名拼音>。选择Fargate模式。“Job attempts”输入1。
把ECR镜像地址写入 *Image*选项中，并且输入Command
```
python backtest.py salen-datalab dwd/fulldata/ethusd/hour/part-00000-3141f899-6907-4349-b97f-ef42743c018c-c000.snappy.parquet
```
选择默认的Task Role，注意勾选 *Assign public Ip*。
最后点击**Create**按钮创建该job definition。
![](/images/submit_job_eth_2.png)

* 我们回到之前的Jupyter Notebook：parallel_backtest.ipynb，然后点击工具栏里的“+”按钮，添加一个新的代码块。
把下面的代码粘贴到该代码框中，执行，从而并行的提交回测任务。
```
import boto3

batch_client = boto3.client('batch')

def submit_job(job_name, queue_name, job_definition):
    response = batch_client.submit_job(
        jobName=job_name,
        jobQueue= queue_name,
        jobDefinition=job_definition
    )


# 在AWS Batch中定义好的任务queue
quene_name = 'backtestqueue'
# 所有在AWS Batch中定义好的job definition名
job_definition_list=['btc-usdt-job-def','eth-usdt-job-def']

# 循环提交所有的任务
for definition in job_definition_list:
    submit_job(definition,quene_name,definition)

```

* 执行以下代码可以查看正在运行的任务，或者直接在AWS Batch的控制台查看
```
response = batch_client.list_jobs(
    jobQueue=quene_name,
    jobStatus='RUNNING',
    maxResults=100
)
for item in response['jobSummaryList']:
    print(item)
```

* 以上就是通过代码提交并行任务，我们甚至可以通过修改参数，传入更多的参数，计算更多的交易对。在实际的环境中，我们甚至可以把行情数据下载到S3，通过自有环境来做更高频的回测测试