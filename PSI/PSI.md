<span id = "top"></span>
# Secretflow简单实现PSI

* [概述](#概述)
* [环境](#环境)
* [具体实现](#具体实现)
    * [容器配置](#容器配置)
    * [Ray集群配置](#Ray集群配置)
    * [开始前的测试](#开始前的测试)
    * [双方PSI](#psi_d)
    * [一些扩展](#extension)
    * [多方PSI](#psi_m)
    * [一些问题](#problem)


## <span id="概述">概述</span>
该PSI方法基于蚂蚁的Secretflow框架实现，现阶段可以实现双容器之间的沟通，单服务器多容器尚在实现，多服务器多容器由于条件不允许，暂时无法实现。该PSI仅尝试Secretflow官方推荐的使用docker的host网络去实现。

[back to the top](#top)
## <span id="环境">环境</span>
* OS : Ubuntu 18.04.5 LTS
* CPU: Xeon Gold 6326 x2, 32-cores
* Memory : 384GB
* Secretflow Version: 1.0.0b3

[back to the top](#top)
## <span id="具体实现">具体实现</span>
以下分为几部分来分析具体实现
### <span id="容器配置">容器配置</span>
在Secretflow的部署教程中，提到了“使用docker部署仿真Secretflow”，但并没有详细提到具体该如何配置host网络下的docker容器。按照一些secretflow的github中相关issue中的描述，采用以下方法进行配置
```bash
docker run -it --network --name XXX secretflow-registry.cn-hangzhou.cr.aliyuncs.com/secretflow/secretflow-lite-anolis8:1.0.0b3
# secretflow-registry.cn-hangzhou.cr.aliyuncs.com/secretflow/secretflow-lite-anolis8:1.0.0b3
```
这里并未使用本地的镜像，按照使用docker安装的教程中的描述尝试过下载，但因网络原因下载速度较慢，换用该镜像源。由于安装容器的数量不多，安装速度并未产生影响，以及日常使用的secretflow框架是使用conda和pypi安装，所以并未将镜像安装到本地。现在服务器上留存有三个容器: *sfnet1*, *sfnet2*, *sfnet3*。

[back to the top](#top)
### <span id="Ray集群配置">Ray集群配置</span>
现阶段Secretflow和Ray的相性不是很好，基本上只兼容2.2.0版本的Ray（高版本的Ray会导致Secretflow的import失败），或者不安装Ray（部分ray的指令可能无法生效）。对*sfnet1*（alice, *sfnet2*（bob）和 *sfnet3* （carol）进行配置。

**sfnet1**
```bash
ray start --head --port=5000 --resources='{"alice": 16}' --include-dashboard=False --disable-usage-stats
```

**sfnet2**
```bash
ray start --address="192.168.11.93:5000" --resources='{"bob": 16}' --disable-usage-stats
```

**sfnet3**
```bash
ray start --address="192.168.11.93:5000" --resources='{"carol": 16}' --disable-usage-stats
```
在配置中，net1作为head，net2和net3作为worker，2和3的配置类似。
配置完成后，使用
```bash
ray status
```
查看，结果有
```
======== Autoscaler status: 2023-07-26 03:38:56.921751 ========
Node status
---------------------------------------------------------------
Healthy:
 1 node_08937480474a69e31bd50a282b5e53fdf1d1e9c2f550eefa11d082d3
 1 node_877a1faf1ff54a0e0a691b3291c61b5f720399c10a950959e050ed8b
 1 node_df98e01797ab98c4f7ca470b488438410fef370d60fd78af89e91396
Pending:
 (no pending nodes)
Recent failures:
 (no failures)

Resources
---------------------------------------------------------------
Usage:
 0.0/192.0 CPU
 0.0/16.0 alice
 0.0/16.0 bob
 0.0/16.0 carol
 0.00/1082.468 GiB memory
 0.00/28.769 GiB object_store_memory

Demands:
 (no resource demands)
```
（这里应该指定一下CPU的数量，每一次“ray start”都会选择全部的CPU核心（64个逻辑核心），然后执行了3次。但对结果影响不是很大）

[back to the top](#top)
### <span id="开始前的测试">开始前的测试</span>
由于环境本身还处于开发阶段，在开始运行程序之前一般会先看一下框架能否正常加载
一般是以
```bash
Python 3.8.15 (default, Nov 24 2022, 15:19:38) 
[GCC 11.2.0] :: Anaconda, Inc. on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import secretflow as sf
>>> sf.init(['alice', 'bob', 'carol'], address='local')
2023-07-26 06:11:46,170 INFO worker.py:1529 -- Started a local Ray instance. View the dashboard at http://127.0.0.1:8265 
>>> 
```
这样的方式去检验环境是否存在问题

**例如**，在解决容器配置的问题的时候，曾经尝试过安装raylet，结果安装完以后按照上面运行就会出现"*No module named 'fed.proxy.grpc*'"这样的问题出现，由于框架本身较新且还处于开发中的原因，对于其他的库的兼容性应该很值得改进。

[back to the top](#top)
### <span id = "psi_d">双方PSI</span>
```python
import secretflow as sf
import spu
import ray

sf.shutdown() 
#清除正在运行的相关进程
sf.init(parties=['alice','bob'], address='local') 
# 以本地地址进行初始化
```
数据来自于 *`sklearn.datasets`* 中的  *`iris`* 数据集
```python
import numpy as np
from sklearn.datasets import load_iris

data, _ = load_iris(return_X_y=True,as_frame=True)
data['uid'] = np.arange(len(data)).astype('str')
data['month'] = ['July'] * (len(data) // 2) + ['August'] * (len(data) - len(data) // 2)

data
```
对数据集进行随机取样
```python
import os

os.makedirs('.data', exist_ok=True)
# 创建'.data'文件夹存放csv的数据
data_alice, data_bob = data.sample(frac=0.8), data.sample(frac=0.7)
# alice 随机取80%的数据
# bob 随机取70%的数据
data_alice.to_csv('.data/alice.csv',index=False)
data_bob.to_csv('.data/bob.csv',index=False)
# 写入csv文件
```
创建一个集群的配置，将该配置应用于创建的SPU中。针对双方的PSI，协议使用semi2K（若有三方，使用ABY3）
```python
import spu

cluster_def={
    'nodes': [
        {
            'party': 'alice',
            'address': '192.168.11.93:5000',
        },
        {
            'party': 'bob',
            'address': '192.168.11.93:5000',
        },
    ], # 本机的IP为192.168.11.93，选取port=5000
    'runtime_config': {
        'protocol': spu.spu_pb2.SEMI2K,
        'field': spu.spu_pb2.FM128,
        'sigmoid_mode': spu.spu_pb2.RuntimeConfig.SIGMOID_REAL,
    }
}

alice = sf.PYU('alice')
bob = sf.PYU('bob')
spu_psi = sf.SPU(cluster_def=cluster_def)

# spu_psi = sf.SPU(sf.utils.testing.cluster_def(['alice','bob']))
# 该行可以作为非正式模拟使用，使用Secretflow自带的配置，实际效用意义不大
```
设定输入文件路径和输出文件的路径，开始计算
```python
input_path = {alice:'.data/alice.csv' , bob: '.data/bob.csv'}
output_path = {alice:'.data/alice_psi.csv' , bob : '.data/bob_psi.csv'}
# 设置输入输出的文件的路径
spu_psi.psi_csv('uid',input_path,output_path,'alice')
# SPU开始运算
```
SPU会输出很多行的log，在普通的python文件中，命令行的输出全部为log。而如果选择notebook的方式去运行，会出现类似于
```bash
[{'party': 'alice', 'original_count': 120, 'intersection_count': 84},
 {'party': 'bob', 'original_count': 105, 'intersection_count': 84}]
```
这样的输出

### <span id = "extension">一些扩展</span>
上述实现的是双方PSI的单键求交
```python
spu_psi.psi_csv('uid',input_path,output_path,'alice')
```
该行使用uid作为键去求交集，若是需要使用多键求交，可将输入模式改为list

```python
spu_psi.psi_csv(['uid','month'],input_path,output_path,'alice')
```

[back to the top](#top)

### <span id = "psi_m">多方PSI<span>
由于已经建立了第三个容器，所以这里的多方先选取三方作为代表。三方PSI运用的理论和双方的相对类似，但是再Secretflow框架下的构建还是略有差异。

部分组件加入了第三方相关内容的初始化
```python
data_carol = data.sample(frac=0.4)
# 第三方数据集的初始化
carol = sf.PYU('carol')
# PYU的初始化
cluster_def={
    'nodes': [
        {
            'party': 'alice',
            'address': '192.168.11.93:5000',
        },
        {
            'party': 'bob',
            'address': '192.168.11.93:5000',
        },
        {   
            'party': 'carol',
            'address': '192.168.11.93:5000',
        },
    ],...
}
# 集群配置中加入了第三方的地址
input_path = {alice:'.data/alice.csv' , bob: '.data/bob.csv', carol: '.data/carol.csv'}
output_path = {alice:'.data/alice_psi.csv' , bob : '.data/bob_psi.csv', carol:'.data/carol_psi.csv'}
# 输入输出方的路径添加
```
部分通信协议改动
```python
cluster_def={...
    'runtime_config': {
        'protocol': spu.spu_pb2.ABY3,
        'field': spu.spu_pb2.FM64,
        'sigmoid_mode': spu.spu_pb2.RuntimeConfig.SIGMOID_REAL,
    }
}
# 协议改为ABY3
```
本来以为上述内容修改完成就可以正常运行
但现在Secretflow的三方求交需要指定协议为"ECDH_PSI_3PC"
```python
# spu_psi.psi_csv(['uid','month'],input_path,output_path,'alice')
# 上述代码会引起错误
spu_psi.psi_csv(['uid','month'],input_path,output_path,'alice', protocol='ECDH_PSI_3PC')
```
最终输出（有修改过部分取样的概率，结果和双方的时候会有不同）
```bash
[{'party': 'alice', 'original_count': 120, 'intersection_count': 30},
 {'party': 'bob', 'original_count': 105, 'intersection_count': 30},
 {'party': 'carol', 'original_count': 60, 'intersection_count': 30}]
```
[back to the top](#top)

### <span id="problem">一些问题</span>
在尝试做该内容的过程中，确实也遇到了不少的问题。
1. 一方面是我个人在这个领域还是初学者，对这一块的知识还不是很熟练，加之在这个过程中还遇到了很多不同方面的没有接触过的内容，确实会存在一些对于新的内容的困惑，也在不断的学习中尽力去解决。
2. 框架本身也算是不完美的。现在框架本身还处于开发状态，稳定性还是不错的（在配置正确的情况下基本没有出现过bug）。但是框架对于环境配置的要求还是比较严格的，像是固定版本的ray，以及安装后一些部件需要去升级或者降级（比如pydantic），有的时候东西装多了反而莫名其妙跑不起来（比如raylet）。
3. 框架相关文档的完成度也不算完全。像是在配置docker容器的时候，文档在部署部分推荐使用host网络，但关于host网络的内容反而相对较少，而不推荐的bridge网络却写的很具体（有可能是host网络的配置相对简单吧），不过总体来说完成度还是不错的。
4. 还有一些综合性的问题，这几个方面可能都是偏向于主观的。
* 像是对于框架中的SPU这一部件的作用，虽然在前面的课程和自学当中对这一部件已经有所了解，但在使用过程中，很难感觉到它到底是如何安全的处理数据的。主要是本人对于攻击方式的具体使用并不是很熟悉，所以就会感觉到好像经过这个SPU的计算返回的结果，和一个简单的筛选器并没有什么本质上的区别。前面我也试过用ElGamel加密的方法去实现一个spu，虽然效率上跟这个框架中的很有差距，暗示感觉还是直观一些。不过这个框架未来可能是会用于生产方面的话，这一点的意义就不是很大了
* 也是这个SPU，现在还没有添加对于像是GPU的支持，现在的使用范围还是比较窄的，可能绝大部分的使用场景都在PSI这一部分，也是这次专注于PSI的原因。像是这两天添加了LLaMA-7B的密态模型的支持，但是只能推理，可能还无法做到训练。感觉相对还是有些遗憾的。其实小模型，像是一些简单的线性回归其实也没什么问题，前两天我也试过。

[back to the top](#top)