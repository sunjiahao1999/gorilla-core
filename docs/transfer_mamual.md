<!-- Copyright (c) Gorilla-Lab. All rights reserved. -->

# 代码迁移手册
该手册的目的在于辅助同学们按照相应的模式进行代码的迁移。

---
## solver
首先是 `solver` 部分，该部分在于实现训练Pipeline的搭建，在以后的项目中希望同学们将网络训练部分的代码以 `solver`**类** 的形式进行包装，而不是分散地写在脚本中。
在我们的核心库`gorilla-core`中提供了`solver`的范式`BaseSolver`：
```python
class BaseSolver(metaclass=ABCMeta):
    def __init__(self,
                 model,
                 dataloaders,
                 cfg,
                 logger=None,
                 **kwargs):
        # 初始化必要成员
        self.model = model
        self.dataloaders = dataloaders
        # meta 用于存放有需要的参数，例如时间/epoch数/正确率等,
        self.meta = {}
        # 构建优化器以及学习率策略（具体可以看源码实现）
        self.optimizer = build_optimizer(model, cfg.optimizer)
        self.lr_scheduler = build_lr_scheduler(self.optimizer, cfg.lr_scheduler)
        # 读取配置文件
        self.cfg = cfg
        # 初始化logger
        self.logger = logger
        # 初始化起始epoch和iter
        self.epoch = cfg.get("start_epoch", 1)
        self.iter = 0
        # 初始化记录容器，用于写入tensorboard，要了解原理观看源码
        self.log_buffer = LogBuffer()
        # tensorboard 容器
        self.tb_writer = SummaryWriter(log_dir=cfg.log) 

        # 准备训练前的设置
        self.get_ready()

    def get_ready(self, **kwargs):
        # NOTE: 该函数的目的在于进行训练前设置（可自行定义）
        # 根据配置文件设置随机数种子
        seed = self.cfg.get("seed", 0)
        if seed != 0:
            from ..core import set_random_seed
            set_random_seed(seed, logger=self.logger)

    def resume(self, checkpoint, **kwargs):
        # 加载权重的API
        check_file_exist(checkpoint) # 检查文件是否存在
        # 加载权重，同时读取 meta 信息
        self.meta = resume(self.model,
                           checkpoint,
                           self.optimizer,
                           self.lr_scheduler,
                           **kwargs)
        # 如果 meta 中有epoch信息，则写入当前solver
        if "epoch" in self.meta:
            self.epoch = self.meta["epoch"] + 1

    def write(self, **kwargs):
        # NOTE: 由于大家使用tensorboard的习惯不同，这里仅实现了最
        #       基本的从 dict 利用 add_scalar 写入 tensorboard
        #       如果有别的需求复写这个函数即可
        self.logger.info("Epoch: {}".format(self.epoch))
        # 对记录容器中的值求取均值然后写入tensorboard中
        self.log_buffer.average()
        for key, avg in self.log_buffer.output.items():
            self.tb_writer.add_scalar(key, avg, self.epoch)

    def clear(self, **kwargs):
        # 清空记录容器（例如每个epoch清空以便记录下一个epoch的数据）
        self.log_buffer.clear()

    @abstractmethod
    def solve(self, **kwargs):
        self.clear()
        # solver启动函数，对应整个训练流程

    @abstractmethod
    def train(self, **kwargs):
        self.clear()
        # 进行每个epoch的训练，根据自己任务编写

    @abstractmethod
    def evaluate(self, **kwargs):
        # 验证网络，根据自己任务编写
        self.clear()
    
```
我们提供了`BaseSolver`这个范式，其中`train/evaluate`分别对应同学们训练脚本中的训练和验证部分也是同学们要自行编写的主体。
关于编写规范，这里介绍以一个非常简单的实现为例：
```python
import gorilla
class SpecificSolver(gorilla.BaseSolver):
    ...
    # solve 函数其功能可以看作是训练的启动器和管理器，不负责具体实现
    def solve(self):
        while self.epoch < self.cfg.max_epoch:
            # epoch训练
            self.train()
            if self.val_flag: # 是否触发验证条件
                self.evaluate()
            self.epoch += 1

    # train 函数可以看作 low-level的solve，不过其重点在于遍历
    # 数据集以及对网络前传的结果进行处理
    def train(self):
        # 清空记录容器
        self.clear()
        for i, (data, gt) in enumerate(self.train_data_loader):
            # 具体网络前传以这种形式实现
            loss = self.step(data, gt)
            # 反传及更新参数
            self.optimizer.zero_grad()
            loss.backward()
            self.optimizer.step()
            self.lr_scheduler.step()
        
        # NOTE: 自定义保存的位置并保存权重
        checkpoint = get_save_path(...)
        gorilla.save_checkpoint(self.model,
                                checkpoint,
                                self.optimizer,
                                self.lr_scheduler,
                                self.meta)
        # 写入tensorboard
        self.write()

    # step 函数专注于网络的前传以及参数的获取记录
    def step(self, data, gt, mode="train"):
        # 每一个iter的前传操作
        # 解析获取数据
        data = data.cuda()
        gt = gt.cuda()
        # 网络前传
        predict = self.model(data)
        # 计算loss
        loss, acc = self.criterion(predict, gt)
        # 获取记录的参数并传递给记录容器
        param_dict = {"loss": loss,
                      "acc": acc}
        self.log_buffer.update(param_dict)
    
    def evaluate(self):
        self.clear()
        # 根据自己的需求写验证函数
        ...
```
训练脚本方便希望同学们根据自己的情况套以上模板，由于同学们测试条件和处理各异，测试脚本则不对同学们做要求。

## config & argparse
如何管理实验流程，大体上无非就两种方式，一种是配置文件，将其读取成 `dict` 对象后从中索引参数；第二种就是 `Argparse`，通过命令行输入以及默认的设置来确定参数。
大部分情况则是两者进行结合，配置文件确定不常改变的参数例如optimizer参数，数据集路径等，`Argparse`则是用来设置较为灵活的参数，例如resume权重，保存地址等。
这里的话我们也是提供了辅助同学们进行配置管理的工具函数。
在配置管理方面，我们这里对同学们提四个要求：
1. 配置文件要有分层，而且至少有以下分层（以 `.yaml` 为例）：
```yaml
optimizer:
    ...
lr_scheduler:
    ...
```
可以有多的子配置项，但是这两个子配置项是不可少的，下面将说明原因。

1. 其中的子配置项 `optimizer` 以及 `scheduler` 要符合以下要求（ `model` 以及 `data` 由于同学们需求各异，则不对内部参数进行统一要求）：
```yaml
optimizer:
    name: Adam,
    lr: 0.001
lr_scheduler:
    name: StepLR,
    step_size: 10000
```
这里希望同学们统一使用 `torch.optim.lr_scheduler` 中的 `_LRScheduler` 派生的各种学习率策略实现，尽量不要使用自己的学习率函数，如果有自己的需求可以基于 `_LRScheduler` 派生出自己的学习率策略并贡献到代码库（例如我们就利用这种方式实现了 `PolyLR` 以及带有各种预热机制的学习率策略）或者使用 `LambdaLR` 实现自定义学习率，以上的解决方案基本能够覆盖学习率策略的不同情况。
上面 `optimizer/lr_scheduler` 的配置格式是为了配合我们的 `build_optimizer/build_lr_scheduler` 函数使用，在上面提到的`BaseSolver`中这样调用：
```python
class BaseSolver(metaclass=ABCMeta):
    def __init__(self, **kwargs):
        ...
        # 构建优化器以及学习率策略（具体可以看源码实现）
        self.optimizer = build_optimizer(model, cfg.optimizer)
        self.lr_scheduler = build_lr_scheduler(self.optimizer, cfg.lr_scheduler)
        ...
```
这两个构建函数的原理也非常简单，根据 `name` 成员获取相应的声明后剩余的参数结合 `model/optimizer` 作为初始化的参数得到相应的 `optimizer/lr_scheduler`。
以 `build_optimizer(model, cfg.optimizer)` 为例，根据 `name: "Adam"` 获得了 `torch.optim.Adam` 的声明，其 API 如下：
```python
torch.optim.Adam(params, lr=0.001, betas=(0.9, 0.999), eps=1e-08, weight_decay=0, amsgrad=False)
```
对于大部分同学来说要负责的参数是 `params` 和 `lr`，所以 `build_optimizer` 内部会获取 `model` 中的参数作为 `params`，紧接着把配置文件中的 `cfg: 0.001` 作为 `lr`，如果要给的那个 `betas` 假设为 `[0.9, 0.99]`，那么配置文件改为以下即可：
```yaml
optimizer:
    name: Adam,
    lr: 0.001
    betas: [0.9, 0.99]
```
`build_lr_scheduler` 同理，它根据 `name` 获取相应的声明，接受 `optimizer` 为首位参数，剩余的从配置文件中读取。

3. 以上配置文件尽可能使用我们的 `Config` 类进行管理，这部分我们直接使用了 `mmcv` 提供了 `Config` 类，所以是有 bug free 的保证的，我们也会，里面提供了非常好的性质其中主要函数为 `Config.fromfile` 同学们可通过阅读之前的介绍手册了解。
4. 同学们在获取 `cfg` 和 `args` 后应尽早将两者合成一个 `cfg`，保证后面的 pipeline 只通过一个 `cfg` 管理配置参数。`args` 作为 `argparse.NameSpace` 可以通过Python内置的 `vars(.)` 转换成 `dict`。
   当然为了辅助同学们进行融合，我们也提供了 `merge_cfg_and_args` 函数，直接将 `cfg` 和 `args` 合并成一个总的 `cfg`，本质就是将 `args` 中的参数融合进 `cfg` 中，如果有重合的参数，则以 `args` 中的参数对 `cfg` 中的参数进行覆盖。

配置文件剩余的部分，例如网络和数据集部分，同学们可以平铺地写，但是一定要根据性质进行区分以及给定相应的注释，例如：
```yaml
# params of reproduction
seed: 1

# params of network
arch: "resnet50"
pretrained: True

# params of dataset
dataset: "Office31"
data_root: "data/"
num_classes: 31

# params of DataLoader
samples_per_gpu: 32  # batch_size
workers_per_gpu: 8  # num_workers
```
当也推荐同学们以这样的形式：
```yaml
# params of reproduction
seed: 1

network:
    arch: "resnet50"
    pretrained: True

dataset:
    dataset: "Office31"
    data_root: "data/"
    num_classes: 31

dataloader:
    samples_per_gpu: 32  # batch_size
    workers_per_gpu: 8  # num_workers
```
这样的话只是读取的时候可能需要多写些索引，但是这样可以通过 `Network(**cfg.network)`, `Datset(**cfg.dataset)` 直接读取参数进行初始化也是很好的方式。

## comment
由于同学们以后或多或少会对代码库进行贡献，为了更好地配合我们的工作，我们需要同学们在平时写代码的时候养成写注释的习惯，至少在上传代码库的部分要按照相应的规范编写代码，以下 `build_scheduler` 为例：
```python
def build_lr_scheduler(
        optimizer: torch.optim.Optimizer,
        lr_scheduler_cfg: [Config, Dict]=SCHEDULER,
        lambda_func=None) -> torch.optim.lr_scheduler._LRScheduler:
    r"""Author: liang.zhihao
    Build a LR scheduler from config.

    Note:
        "name" must be in lr_scheduler_cfg

    Args:
        optimizer (torch.optim.Optimizer): Input Optimizer
        lr_scheduler_cfg ([Cofnig, Dict]): learning rate scheduler
        lambda_func(lambda, optional): Custom learning rate function,
                                       for using LambdaLR

    Example:
        cfg = Config.fromfile(cfg.config_file)
        model = build_model(cfg)
        optimizer = build_optimizer(model, cfg.optimizer)
        lr_scheduler = build_lr_scheduler(optimizer, cfg.lr_scheduler)

    Returns:
        _LRScheduler: the learning rate scheduler
    """
    name = lr_scheduler_cfg.pop("name")
    lr_scheduler_cfg["optimizer"] = optimizer

    # specificial for LambdaLR
    if name == "LambdaLR":
        assert lambda_func is not None
        assert isinstance(lambda_func, Callable) or \
            is_seq_of(lambda_func, Callable), "lambda_func is invalid"
        lr_scheduler_cfg["lr_lambda"] = lambda_func

    # get the caller
    scheduler_caller = getattr(lr_schedulers, name)
    return scheduler_caller(**lr_scheduler_cfg)
```
这个就是构建 `lr_scheduler` 的函数，注释要求如下：
1. 如果有条件建议，参数输入可以写成 `arg: type=default` 的形式以及指定输出 `-> output_type`，这是Python的类型标注特性，方便代码的阅读。
2. 采用 `r"""comment"""`的形式（无效转义字符）。
3. 顶格写 `author: xing.ming` 标注贡献者名称（如是外部代码则可不写，如果代码中有 **Copyright** 则要保留在文件头部-法律意识）。
4. 换行写明函数功能以及介绍。
5. `Args/Returns` 是必要的最好标注类型，`torch.Tensor` 可标注形状，但参数含义一定要写。
6. `Note/Example` 等无特殊要求可不写，这里仅为展示使用。

如果是类的声明函数同样，无需写 `Returns`，运行函数，如模型的 `forward` 以及其他类别的 `__call__`，如有必要也应该按照上面的标准进行相应注释的编写。

这里使用VSCode的同学使用 **Python Docstring Generator** 这款注释生成插件，非常好用。同学们在写完代码后写注释也方便进行相应的 review。

另外很多同学也有在网络前传或者进行tensor操作时在最后标注tensor尺寸的习惯。如果要标注张量尺寸，这里统一一下标注形式：
```python
out = conv(inp) # [B, H, W, C]
```
即以方括号包住尺寸表示，如果在函数注释中标注向量尺寸则是：
```python
r"""
    ...
    Args:
        in_feats (torch.Tensor, [B, H, W, C]): Input feaure
"""
```
同样以方括号的形式放在含义解释前。




