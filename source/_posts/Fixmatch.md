---
title: Fixmatch主要代码注释
author: Ahtong
tags:
  - Python
  - 深度学习
  - 图像分类
categories: 论文笔记
top: false
cover: false
toc: true
abbrlink: 94813030
date: 2022-11-12 09:58:45
description:
img:
coverImg:
---

## 设置种子

```python
def set_seed(args):
    random.seed(args.seed) # python的随机性
    np.random.seed(args.seed) # np的随机性
    torch.manual_seed(args.seed) # torch的CPU随机性，为CPU设置随机种子
    if args.n_gpu > 0:
        torch.cuda.manual_seed_all(args.seed) # torch的GPU随机性，为所有GPU设置随机种子
```

1. 设置随机种子
2. 将种子赋予np
3. 将种子赋予torch
4. 将种子赋予cuda

## GPU设置

```python
if args.local_rank == -1:
    device = torch.device('cuda', args.gpu_id)
    args.world_size = 1
    args.n_gpu = torch.cuda.device_count()
else:
    torch.cuda.set_device(args.local_rank)
    device = torch.device('cuda', args.local_rank)
    torch.distributed.init_process_group(backend='nccl')
    args.world_size = torch.distributed.get_world_size()
    args.n_gpu = 1
```

根据local_rank决定是否采取分布式。如果local_rank=-1，说明分布式失效；如果local_rank不等于-1，则根据不同的卡配置不同的进程数；获取设备device方便后续将数据和模型加载在上面（代码为.to(device))；初始化设置分布式的后端等。

**torch.distributed.barrier()的使用：**

①数据集：

```python
if args.local_rank not in [-1, 0]:
    torch.distributed.barrier()

# 获取数据集
labeled_dataset, unlabeled_dataset, test_dataset = DATASET_GETTERS[args.dataset](
    args, './data')

if args.local_rank == 0:
    torch.distributed.barrier()
```

有些操作是不需要多卡同时运行的，如数据集和模型的加载。因此，PyTorch对非主进程的卡上面的运行进行了barrier设置。如果是在并行训练非主卡上，其它进行需要先等待主进程读取并缓存数据集，再从缓存中读取数据，以同步不同进程的数据，避免出现数据处理不同步的现象。

②模型

```python
if args.local_rank not in [-1, 0]:
    torch.distributed.barrier()

model = create_model(args)

if args.local_rank == 0:
    torch.distributed.barrier()
```

先对其余进程设置一个障碍，等到主进程加载完模型和数据后，再对主进程设置障碍，使所有进程都处于同一“出发线”，最后再同时释放。

<!--more-->

## 数据集划分

本代码使用的数据集分为三类：带标签的训练集，不带标签的训练集，测试集。虽然表面上需要一个训练集是“不带标签”的，但是PyTorch并没有直接舍去标签的数据集设置。一开始我在想，如果是我自己来写代码，应该要怎么处理呢？后来发现代码根本没有拘泥于“不带标签”这个事情，因为在返回数据集和标签时，使用“_”直接代替掉标签即可，损失函数也不需要使用标签。

核心API：
```python
labeled_dataset, unlabeled_dataset, test_dataset = DATASET_GETTERS[args.dataset](args, './data')
```

dataset->cifar.py，发现调用了如下函数（get_cifar10和get_cifar100极其类似，只是数据集分类的类别数不一样而已。下面仅以get_cifar100为例）：

```
def get_cifar100(args, root):
    # 图像变换
    transform_labeled = transforms.Compose([
        transforms.RandomHorizontalFlip(),
        transforms.RandomCrop(size=32,
                              padding=int(32*0.125),
                              padding_mode='reflect'),
        transforms.ToTensor(),
        transforms.Normalize(mean=cifar100_mean, std=cifar100_std)])

    transform_val = transforms.Compose([
        transforms.ToTensor(),
        transforms.Normalize(mean=cifar100_mean, std=cifar100_std)])

    # 数据集设置
    base_dataset = datasets.CIFAR100(
        root, train=True, download=True)

    train_labeled_idxs, train_unlabeled_idxs = x_u_split(
        args, base_dataset.targets)

    train_labeled_dataset = CIFAR100SSL(
        root, train_labeled_idxs, train=True,
        transform=transform_labeled)

    train_unlabeled_dataset = CIFAR100SSL(
        root, train_unlabeled_idxs, train=True,
        transform=TransformFixMatch(mean=cifar100_mean, std=cifar100_std))

    test_dataset = datasets.CIFAR100(
        root, train=False, transform=transform_val, download=False)

    return train_labeled_dataset, train_unlabeled_dataset, test_dataset
```

get_cifar100函数包括两部分：transform的设置和数据集设置。

（1）transform

对于测试集和带标签的训练集，可以根据论文[1]的介绍进行设置。但是对于不带标签的训练集，代码调用了TransformFixMatch类，因为这部分的训练集需要使用弱增强和强增强的方法，两种方法是不同的，所以需要特别设置一个callable的类，能够将两种transform手段凑在一块。当构建dataset调用transform时，可以直接调用call函数，直接返回两个增强手段处理后的图像。

```python
class TransformFixMatch(object):
    def __init__(self, mean, std):
        self.weak = transforms.Compose([
            transforms.RandomHorizontalFlip(),
            transforms.RandomCrop(size=32,
                                  padding=int(32*0.125),
                                  padding_mode='reflect')])
        self.strong = transforms.Compose([
            transforms.RandomHorizontalFlip(),
            transforms.RandomCrop(size=32,
                                  padding=int(32*0.125),
                                  padding_mode='reflect'),
            RandAugmentMC(n=2, m=10)])
        self.normalize = transforms.Compose([
            transforms.ToTensor(),
            transforms.Normalize(mean=mean, std=std)])

    def __call__(self, x):
        weak = self.weak(x)
        strong = self.strong(x)
        return self.normalize(weak), self.normalize(strong)
```

（2）数据索引设置

怎么从原始的CIFAR数据集提取出带标签的训练集和无标签的训练集？注意到PyTorch数据集类有一个函数成员def __getitem__(self, index)，核心参数是index，所以我们构建以上两个训练集，本质上是构建训练集对应的索引值。下面是索引生成函数x_u_split的代码：

```
def x_u_split(args, labels):
    label_per_class = args.num_labeled // args.num_classes
    labels = np.array(labels) #每个label是一个数字
    labeled_idx = []
    unlabeled_idx = np.array(range(len(labels)))
    for i in range(args.num_classes):
        idx = np.where(labels == i)[0] #有[0]是因为np.where得到的是一个tuple,需要把tuple的元素提取出来
        idx = np.random.choice(idx, label_per_class, False)
        labeled_idx.extend(idx)
    labeled_idx = np.array(labeled_idx)
    assert len(labeled_idx) == args.num_labeled

    if args.expand_labels or args.num_labeled < args.batch_size:
        num_expand_x = math.ceil( #向上取整
            args.batch_size * args.eval_step / args.num_labeled) #等于17

        #将参数元组的元素数组按水平方向进行叠加
        labeled_idx = np.hstack([labeled_idx for _ in range(num_expand_x)])
    np.random.shuffle(labeled_idx)
    return labeled_idx, unlabeled_idx
```

每个类带标签数据的个数是均衡的，每个类带标签的数据个数 = 带标签数据总个数//类数，所以，使用一个循环（10个类）。
对于每一个类，找出他们在总数据（labels）中的数据索引，然后将labels（原本是列表）转换为numpy数组。并用random.choice随机选择label_per_class个数据，将他们加入到带标签的数据索引labeled_idx中。
对于不带标签的数据，原文作者使用了所有的数据（包含带标签的数据），所以他的索引为全部数据的索引，unlabeled_idx可以直接对应全体数据。
需要注意的一个点是，args.expand_labels参数作者默认为true的，所以我们要进行数据重复。
或者num_labeled比batch_size还小，则对数组进行扩充。
这里重复的次数num_expand_x为 64（batch_size ）* 1024（eval_step）/ 4000 （num_labeled）=17次
所以带标签的数据为 68000个（每个索引都重复了17次）。

（3）数据集设置

```python
class CIFAR100SSL(datasets.CIFAR100):
    def __init__(self, root, indexs, train=True,
                 transform=None, target_transform=None,
                 download=False):
        super().__init__(root, train=train,
                         transform=transform,
                         target_transform=target_transform,
                         download=download)
        if indexs is not None:
            self.data = self.data[indexs]
            self.targets = np.array(self.targets)[indexs]

    def __getitem__(self, index):
        img, target = self.data[index], self.targets[index]
        img = Image.fromarray(img)

        if self.transform is not None:
            img = self.transform(img)

        if self.target_transform is not None:
            target = self.target_transform(target)

        return img, target
```

## scheduler

```python
def get_cosine_schedule_with_warmup(optimizer,
                                    num_warmup_steps,
                                    num_training_steps,
                                    num_cycles=7./16.,
                                    last_epoch=-1):
    def _lr_lambda(current_step):
        if current_step < num_warmup_steps:
            return float(current_step) / float(max(1, num_warmup_steps))
        no_progress = float(current_step - num_warmup_steps) / \
            float(max(1, num_training_steps - num_warmup_steps))
        return max(0., math.cos(math.pi * num_cycles * no_progress))

    return LambdaLR(optimizer, _lr_lambda, last_epoch)
# LambdaLR设置学习率为初始学习率乘以给定lr_lambda函数的值
# 当last_epoch=-1时, base_lr为optimizer优化器中的lr
# 每次执行 scheduler.step(),  last_epoch=last_epoch +1
```

scheduler是为了动态调整训练期间的学习率，使模型更好地收敛。论文使用的是带有warmup性质的余弦退火学习率调整器。核心是返回了一个自定义函数的学习率调整器，调整的函数是_lr_lambda，如果当前的step少于warmup的步数，则使用线性递增的策略一直增加到初始学习率；而后使用余弦变化的策略改变学习率:
$$
\eta_1=\cos{(\frac{7\pi k}{16K})}
$$
$\eta$是初始学习率，$k$是当前的步数，$K$是总步数。

## 混合精度

本代码使用的是英伟达开发的apex库，可以通过使用混合精度，在保证精度丢失很少的情况下，减少内存，增快训练速度。混合精度涉及对模型和优化器的重初始化、损失函数的反向传播等。代码如下：

~~~python
from apex import amp
model, optimizer = amp.initialize(model, optimizer, opt_level=args.opt_level)

with amp.scale_loss(loss, optimizer) as scaled_loss:
    scaled_loss.backward()
~~~

## 指数移动平均（EMA）

EMA在本代码是用于更新模型权重的，核心公式就这一条：
$$
v_t=\beta v_{t-1}+(1-\beta)\theta_t
$$
这里的参数$v$代表测试用模型的参数权重。训练时，原模型就按照正常的节奏来训练、更新权重，而另外开辟一个EMA模型，在原模型更新权重的同时也跟着更新权重，并作为最后使用的模型，检测在测试集上的表现。

~~~python
class ModelEMA(object):
    def __init__(self, args, model, decay):
        self.ema = deepcopy(model)
        self.ema.to(args.device)
        self.ema.eval()
        self.decay = decay
        self.ema_has_module = hasattr(self.ema, 'module')
        # Fix EMA. https://github.com/valencebond/FixMatch_pytorch thank you!
        self.param_keys = [k for k, _ in self.ema.named_parameters()]
        self.buffer_keys = [k for k, _ in self.ema.named_buffers()]
        for p in self.ema.parameters():
            p.requires_grad_(False)

    def update(self, model): # 核心模块
        # hasattr函数用于判断对象是否包含对应的属性。
        needs_module = hasattr(model, 'module') and not self.ema_has_module
        with torch.no_grad():
            msd = model.state_dict() # torch.nn.Module模块中的state_dict变量存放训练过程中需要学习的权重
            esd = self.ema.state_dict()
            for k in self.param_keys:
                if needs_module:
                    j = 'module.' + k
                else:
                    j = k
                model_v = msd[j].detach()
                ema_v = esd[k]
                # ema_v是过去的平均状态，model_v是现在的参数
                esd[k].copy_(ema_v * self.decay + (1. - self.decay) * model_v)

            for k in self.buffer_keys:
                if needs_module:
                    j = 'module.' + k
                else:
                    j = k
                esd[k].copy_(msd[j])
~~~

## 权重衰减（Weight Decay）

```python
grouped_parameters = [
        # 若网络层不包含 bias 或 BatchNorm，则应用 weight_decay
        # any() 函数用于判断给定的可迭代参数 iterable 是否全部为 False，则返回 False，如果有一个为 True，则返回 True
        {'params': [p for n, p in model.named_parameters() if not any(
            nd in n for nd in no_decay)], 'weight_decay': args.wdecay},
        # 反之，则不用 weight_decay
        {'params': [p for n, p in model.named_parameters() if any(
            nd in n for nd in no_decay)], 'weight_decay': 0.0}
    ]
```

## 核心算法

```python
labeled_iter = iter(labeled_trainloader)
unlabeled_iter = iter(unlabeled_trainloader)

model.train()
for epoch in range(args.start_epoch, args.epochs):
    # 平均处理器，用于存储一些统计信息
    batch_time = AverageMeter()
    data_time = AverageMeter()
    losses = AverageMeter()
    losses_x = AverageMeter()
    losses_u = AverageMeter()
    mask_probs = AverageMeter()
    if not args.no_progress:
        p_bar = tqdm(range(args.eval_step),
                     disable=args.local_rank not in [-1, 0])
    for batch_idx in range(args.eval_step):
        # 使用iter(next)读取指定次数的batch，而不通过Dataloader。Dataloader的长度也不同
        try:
            inputs_x, targets_x = labeled_iter.next()
            #print(inputs_x.shape) # torch.Size([64, 3, 32, 32])
            #print(targets_x.shape) # torch.Size([64])
            #print(targets_x)
        except: # 当循环结束时，重新开始循环
            if args.world_size > 1:
                labeled_epoch += 1
                labeled_trainloader.sampler.set_epoch(labeled_epoch)
            labeled_iter = iter(labeled_trainloader)
            inputs_x, targets_x = labeled_iter.next()

        try:
            (inputs_u_w, inputs_u_s), _ = unlabeled_iter.next()
            #print(inputs_u_w.shape) #torch.Size([448, 3, 32, 32])
            #print(inputs_u_s.shape) #torch.Size([448, 3, 32, 32])
        except:
            if args.world_size > 1:
                unlabeled_epoch += 1
                unlabeled_trainloader.sampler.set_epoch(unlabeled_epoch)
            unlabeled_iter = iter(unlabeled_trainloader)
            (inputs_u_w, inputs_u_s), _ = unlabeled_iter.next() # 忽略标签

        data_time.update(time.time() - end)
        batch_size = inputs_x.shape[0] # 64
        # 带标签的数据每批次有B个，无标签数据每批次有μB个(每个2张图像)，加起来就是(2μ+1)B个
        inputs = interleave(
            torch.cat((inputs_x, inputs_u_w, inputs_u_s)), 2*args.mu+1).to(args.device)
        # print(inputs.shape) #torch.Size([960, 3, 32, 32]) 960=448+448+64 960=64*(2*7+1) 将数据合并一起
        targets_x = targets_x.to(args.device)
        # print(targets_x.shape) #torch.Size([64])

        logits = model(inputs)
        #print(logits.shape) #torch.Size([960, 10])
        logits = de_interleave(logits, 2*args.mu+1)
        #print(logits.shape) #torch.Size([960, 10])
        logits_x = logits[:batch_size] # 前B个
        #print(logits_x.shape) #torch.Size([64, 10])
        logits_u_w, logits_u_s = logits[batch_size:].chunk(2)
        # torch.chunk 将输入Tensor拆分为特定数量的块。如果给定维度dim上的Tensor大小不能够被整除，则最后一个块会小于之前的块。
        #print(logits_u_w.shape) #torch.Size([448, 10])
        del logits # 省出内存，del删除的是变量，而不是数据。

        # 带标签数据的损失函数
        Lx = F.cross_entropy(logits_x, targets_x, reduction='mean')

        # 通过weak_augment样本计算伪标记pseudo label和mask，
        # 其中，mask用来筛选哪些样本最大预测概率超过阈值，可以拿来使用，哪些不能使用

        pseudo_label = torch.softmax(logits_u_w.detach()/args.T, dim=-1) # 与 dim=2 等价，对某一维度的行进行softmax运算，和为1
        # Softmax为T＝1时的特例
        max_probs, targets_u = torch.max(pseudo_label, dim=-1)
        #print(max_probs.shape) # torch.Size([448]) 448个最大概率值
        #print(targets_u.shape) # torch.Size([448]) 448个伪标签的值，实际上是pseudo_label中最大位置的索引
        #print(targets_u) #tensor([3, 5, 1 ....], device='cuda:0')
        mask = max_probs.ge(args.threshold).float() # greater and equal（大于等于）
        # 比0.95大才说明这个标签置信度高，如果低于这个阈值，即使计算了交叉熵，也会被mask为0
        # torch.ge(a,b)逐个元素比较a，b的大小
        # print(mask.shape) #torch.Size([448]) 448个0/1
        # print(F.cross_entropy(logits_u_s, targets_u,reduction='none')) # reduction='none'不求平均，返回448个值

        # 不带标签数据的损失函数
        Lu = (F.cross_entropy(logits_u_s, targets_u,
                              reduction='none') * mask).mean()

        loss = Lx + args.lambda_u * Lu # 完整的损失函数
```

其中torch.max的用法参考如下

```python
a = torch.randn(24).reshape(2,3,4)
print(a)
```

运行结果:

```python
tensor([[[-0.9135,  1.3096,  0.2803, -0.9314],
         [-0.2687, -0.0968, -0.7156, -0.8814],
         [-1.0099,  1.6910,  0.3458, -0.6547]],

        [[-0.4334, -0.0464, -1.9236,  0.3148],
         [ 0.3628, -0.7063, -0.1750,  1.5068],
         [ 1.1270, -0.9374, -0.8419, -0.0050]]])
```

```python
A = torch.softmax(a.detach()/1, dim=-1) # 与 dim=2 等价，对某一维度的行进行softmax运算，和为1
# A = torch.softmax(a, dim=-1)
print(A)
```

```python
tensor([[[0.0689, 0.6362, 0.2273, 0.0677],
         [0.2968, 0.3525, 0.1899, 0.1608],
         [0.0472, 0.7025, 0.1830, 0.0673]],

        [[0.2079, 0.3061, 0.0468, 0.4392],
         [0.1974, 0.0678, 0.1153, 0.6196],
         [0.6294, 0.0799, 0.0879, 0.2029]]])
```

```python
max_probs, targets_u = torch.max(A, dim=-1)
print(max_probs)
print(max_probs.shape)
print(targets_u)
print(targets_u.shape)
```

```python
tensor([[0.6362, 0.3525, 0.7025],
        [0.4392, 0.6196, 0.6294]])
torch.Size([2, 3])
tensor([[1, 1, 1],
        [3, 3, 0]])
# cifar10标签序号从0-9，一共十类，IN数据集标签序号从1-16，一共十六类
# 而torch.max返回的最大位置索引是从0开始，这会导致序号对不上
# 上句话不对，加载数据集的时候已经把0标签排除了，所以不影响
torch.Size([2, 3])
```

## 模型保存与加载

### 模型保存过程

```python
if args.local_rank in [-1, 0]:
    test_loss, test_acc = test(args, test_loader, test_model, epoch)

    args.writer.add_scalar('train/1.train_loss', losses.avg, epoch)
    args.writer.add_scalar('train/2.train_loss_x', losses_x.avg, epoch)
    args.writer.add_scalar('train/3.train_loss_u', losses_u.avg, epoch)
    args.writer.add_scalar('train/4.mask', mask_probs.avg, epoch)
    args.writer.add_scalar('test/1.test_acc', test_acc, epoch)
    args.writer.add_scalar('test/2.test_loss', test_loss, epoch)

    is_best = test_acc > best_acc
    best_acc = max(test_acc, best_acc)

    model_to_save = model.module if hasattr(model, "module") else model # hasattr() 函数用于判断对象是否包含对应的属性
    if args.use_ema:
        ema_to_save = ema_model.ema.module if hasattr(
            ema_model.ema, "module") else ema_model.ema
        save_checkpoint({
            'epoch': epoch + 1,
            'state_dict': model_to_save.state_dict(),
            'ema_state_dict': ema_to_save.state_dict() if args.use_ema else None,
            'acc': test_acc,
            'best_acc': best_acc,
            'optimizer': optimizer.state_dict(),
            'scheduler': scheduler.state_dict(),
        }, is_best, args.out)

        test_accs.append(test_acc)
        logger.info('Best top-1 acc: {:.3f}'.format(best_acc))
        logger.info('Mean top-1 acc: {:.3f}\n'.format(
            np.mean(test_accs[-20:])))
```

### 状态字典：state_dict：

在PyTorch中，`torch.nn.Module`模型的可学习参数（即权重和偏差）包含在模型的参数中，（使用`model.parameters()`可以进行访问）。 `state_dict`是Python字典对象，它将每一层映射到其参数张量。注意，只有具有可学习参数的层（如卷积层，线性层等）的模型才具有`state_dict`这一项。目标优化`torch.optim`也有`state_dict`属性，它包含有关优化器的状态信息，以及使用的超参数。
因为state_dict的对象是Python字典，所以它们可以很容易的保存、更新、修改和恢复，为PyTorch模型和优化器添加了大量模块。
下面通过从简单模型训练一个分类器中来了解一下`state_dict`的使用。

```python
import torch.nn as nn
import torch.nn.functional as F
import torch.optim as optim
# 定义模型
class TheModelClass(nn.Module):
    def __init__(self):
        super(TheModelClass, self).__init__()
        self.conv1 = nn.Conv2d(3, 6, 5)
        self.pool = nn.MaxPool2d(2, 2)
        self.conv2 = nn.Conv2d(6, 16, 5)
        self.fc1 = nn.Linear(16 * 5 * 5, 120)
        self.fc2 = nn.Linear(120, 84)
        self.fc3 = nn.Linear(84, 10)

    def forward(self, x):
        x = self.pool(F.relu(self.conv1(x)))
        x = self.pool(F.relu(self.conv2(x)))
        x = x.view(-1, 16 * 5 * 5)
        x = F.relu(self.fc1(x))
        x = F.relu(self.fc2(x))
        x = self.fc3(x)
        return x

# 初始化模型
model = TheModelClass()

# 初始化优化器
optimizer = optim.SGD(model.parameters(), lr=0.001, momentum=0.9)

# 打印模型的状态字典
print("Model's state_dict:")
for param_tensor in model.state_dict():
    print(param_tensor, "\t", model.state_dict()[param_tensor].size())

# 打印优化器的状态字典
print("Optimizer's state_dict:")
for var_name in optimizer.state_dict():
    print(var_name, "\t", optimizer.state_dict()[var_name])
```

输出：

```python
Model's state_dict:
conv1.weight 	 torch.Size([6, 3, 5, 5])
conv1.bias 	 torch.Size([6])
conv2.weight 	 torch.Size([16, 6, 5, 5])
conv2.bias 	 torch.Size([16])
fc1.weight 	 torch.Size([120, 400])
fc1.bias 	 torch.Size([120])
fc2.weight 	 torch.Size([84, 120])
fc2.bias 	 torch.Size([84])
fc3.weight 	 torch.Size([10, 84])
fc3.bias 	 torch.Size([10])
Optimizer's state_dict:
state 	 {}
param_groups 	 [{'lr': 0.001, 'momentum': 0.9, 'dampening': 0, 'weight_decay': 0, 'nesterov': False, 'maximize': False, 'params': [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]}]
```

### 定义save_checkpoint保存完整模型

```python
def save_checkpoint(state, is_best, checkpoint, filename='checkpoint.pth.tar'):
    filepath = os.path.join(checkpoint, filename)
    torch.save(state, filepath) # 保存模型
    if is_best:
        shutil.copyfile(filepath, os.path.join(checkpoint, 'model_best.pth.tar')) # 复制文件
```

当保存好模型用来推断的时候，只需要保存模型学习到的参数，使用`torch.save()`函数来保存模型`state_dict`。
在 PyTorch 中最常见的模型保存使‘.pt’或者是‘.pth’作为模型文件扩展名。
在运行推理之前，务必调用`model.eval()`去设置 dropout 和 batch normalization 层为评估模式。如果不这么做，可能导致模型推断结果不一致。
注意：
`load_state_dict()`函数只接受字典对象，而不是保存对象的路径。这就意味着在你传给`load_state_dict()`函数之前，你必须反序列化你保存的`state_dict`。例如，你无法通过 `model.load_state_dict(PATH)`来加载模型。

### 保存和加载 Checkpoint 用于推理/继续训练

#### **保存Checkpoint：**

```python
save_checkpoint({
            'epoch': epoch + 1,
            'state_dict': model_to_save.state_dict(),
            'ema_state_dict': ema_to_save.state_dict() if args.use_ema else None,
            'acc': test_acc,
            'best_acc': best_acc,
            'optimizer': optimizer.state_dict(),
            'scheduler': scheduler.state_dict(),
        }, is_best, args.out)
```

当保存成 Checkpoint 的时候，可用于推理或者是继续训练，保存的不仅仅是模型的`state_dict`。保存优化器的`state_dict`也很重要, 因为它包含作为模型训练更新的缓冲区和参数。你也许想保存其他项目，比如最新记录的训练损失，外部的`torch.nn.Embedding`层等等。
要保存多个组件，请在字典中组织它们并使用`torch.save()`来序列化字典。PyTorch 中常见的保存checkpoint是使用 .tar 文件扩展名。
要加载项目，首先需要初始化模型和优化器，然后使用`torch.load()`来加载本地字典。这里，你可以非常容易的通过简单查询字典来访问你所保存的项目。
请记住在运行推理之前，务必调用`model.eval()`去设置 dropout 和 batch normalization 为评估。如果不这样做，有可能得到不一致的推断结果。 如果你想要恢复训练，请调用`model.train()`以确保这些层处于训练模式。

#### **加载Checkpoint：**

```python
if args.resume:
    logger.info("==> Resuming from checkpoint..")
    # os.path.isfile判断某一对象(需提供绝对路径)是否为文件
    assert os.path.isfile(
        args.resume), "Error: no checkpoint directory found!"
    args.out = os.path.dirname(args.resume)
    checkpoint = torch.load(args.resume)
    best_acc = checkpoint['best_acc']
    args.start_epoch = checkpoint['epoch']
    model.load_state_dict(checkpoint['state_dict'])
    if args.use_ema:
        ema_model.ema.load_state_dict(checkpoint['ema_state_dict'])
    optimizer.load_state_dict(checkpoint['optimizer'])
    scheduler.load_state_dict(checkpoint['scheduler'])
```

#### 加载最优模型：

```python
# 自己写的
filepath = os.path.join(args.out, 'model_best.pth.tar')
    assert os.path.isfile(filepath), "Error: no model_best directory found!"
    model.load_state_dict(torch.load(filepath)['state_dict'])
    if args.use_ema:
        ema_model.ema.load_state_dict(torch.load(filepath)['ema_state_dict'])
```



## accuracy

```python
# outputs.shape为 [batch_size, category_count]
# targets.shape为 [batch_size]，每个样本中只有一个真实的类
# topk是要包含在精度中的类的元组，例如topk=(1,5)，则包含五个类
# topk必须是一个元组，所以给出一个数字，不要忘记逗号
def accuracy(output, target, topk=(1,)):
    """Computes the precision@k for the specified values of k"""
    maxk = max(topk)
    # size函数：总元素的个数
    batch_size = target.size(0)

    # topk函数output的d维度中选取前k个最大的值，下式为前maxk个
    # _, pred = output.topk(maxk, 1, True, True)
    # output.shape为 [batch_size, category_count]，dim=1，所以我们为每个batch选择最大的类别数
    # 输入结果为[ batch_size,maxk]
    # topk返回结果的元组 (values, indexes) (值，索引)
    # 我们只需要索引(pred)
    _, pred = output.topk(maxk, dim=1, largest=True, sorted=True)
    # 然后我们将索引转置为 [maxk, batch_size]
    pred = pred.t()
    correct = pred.eq(target.reshape(1, -1).expand_as(pred))
    # torch.eq对两个张量Tensor进行逐元素的比较，若相同位置的两个元素相同，则返回True；若不同，返回False
    # 将target展平，并将target扩展成类似于pred
    # target [batch_size] 变成 [1,batch_size]
    # target [1,batch_size] 通过广播？重复相同的类maxk次，从 [1,batch_size] 变为 [maxk, batch_size]
    # 当将索引(pred)与扩展后的target比较时，即torch.eq，会得到形状为 [maxk, batch_size] 的矩阵correct
    """ correct=([[0, 0, 1,  ..., 0, 0, 0],
         [1, 0, 0,  ..., 0, 0, 0],
         [0, 0, 0,  ..., 1, 0, 0],
         [0, 0, 0,  ..., 0, 0, 0],
         [0, 1, 0,  ..., 0, 0, 0]], device='cuda:0', dtype=torch.uint8) """

    res = []
    for k in topk:
        correct_k = correct[:k].reshape(-1).float().sum(0)
        # correct[:k]：[maxk, batch_size] -> [k, batch_size]
        # .reshape(-1)：[k, batch_size] -> [k*batch_size]
        # .sum(0)：[k*batch_size] -> [1]
        res.append(correct_k.mul_(100.0 / batch_size))
        # mul 乘法
        # 所有带_都是inplace，意思就是操作后，原数也会改动
    return res
```

topk函数：

```
torch.topk(input, k, dim=None, largest=True, sorted=True, out=None) -> (Tensor, LongTensor)
```

input (Tensor)：输入张量，一个tensor数据
k (int)：指明是得到前k个数据以及其index
dim (int, optional)： 指定在哪个维度上排序， 默认是最后一个维度
largest (bool, optional)：如果为True，按照大到小排序； 如果为False，按照小到大排序
sorted (bool, optional) ：控制返回值是否排序
out (tuple, optional)：可选输出张量 (Tensor, LongTensor)

例如：

```python
a = torch.tensor([[ 0,  1,  1,  0],
        [ 0,  0,  0,  0],
        [ 0,  0,  0,  0],
        [ 1,  0,  0,  0],
        [ 0,  0,  0,  1]])
```

```python
print(a)
print(a.shape)
```

```python
tensor([[0, 1, 1, 0],
        [0, 0, 0, 0],
        [0, 0, 0, 0],
        [1, 0, 0, 0],
        [0, 0, 0, 1]])
torch.Size([5, 4])
```

```python
correct_1 = a[:1].reshape(-1)
print(correct_1.shape)
print(correct_1)
correct_1 = correct_1.float().sum(0)
print(correct_1.shape)
print(correct_1)
correct_1.mul_(100.0 / 4)
print(correct_1.shape)
print(correct_1)
```

```python
torch.Size([4])
tensor([0, 1, 1, 0])
torch.Size([])
tensor(2.)
torch.Size([])
tensor(50.) # top_1 = 50%
```

```python
correct_5 = a[:5].reshape(-1)
print(correct_5.shape)
print(correct_5)
correct_5 = correct_5.float().sum(0)
print(correct_5.shape)
print(correct_5)
correct_5.mul_(100.0 / 4)
print(correct_5.shape)
print(correct_5)
```

```python
torch.Size([20])
tensor([0, 1, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 0, 0, 0, 0, 0, 0, 1])
torch.Size([])
tensor(4.)
torch.Size([])
tensor(100.) # top_5 = 100%
```
