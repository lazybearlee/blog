# 项目代码

## 结构

```plaintext
main.py
model.py
dataset.py
```

## main.py

```python
"""
推荐系统训练主程序 - 基于Transformer的多特征推荐模型训练脚本

功能概述：
该脚本实现了基于Transformer的多特征推荐模型的完整训练流程，
包括数据加载、模型初始化、训练循环、验证和模型保存等功能。

核心流程：
1. 参数解析：解析命令行参数和配置
2. 环境初始化：创建日志目录、初始化日志记录器
3. 数据准备：加载和预处理训练数据
4. 模型构建：初始化推荐模型和优化器
5. 模型训练：执行训练循环，包括前向传播、损失计算和反向传播
6. 模型验证：在验证集上评估模型性能
7. 模型保存：保存训练好的模型参数

技术特点：
1. 模块化设计：清晰分离数据处理、模型定义和训练逻辑
2. 灵活配置：支持多种模型参数和训练配置
3. 完善日志：记录训练过程中的关键指标和事件
4. 断点续训：支持从检查点恢复训练

使用方法：
1. 训练模式：python main.py
2. 推理模式：python main.py --inference_only --state_dict_path /path/to/model.pt
3. 自定义参数：python main.py --batch_size 256 --lr 0.0005 --num_epochs 10

环境要求：
- Python 3.8+
- PyTorch 1.8+
- CUDA支持（推荐）
"""

import argparse
import json
import os
import time
from pathlib import Path

import numpy as np
import torch
from torch.utils.data import DataLoader
from torch.utils.tensorboard import SummaryWriter
from tqdm import tqdm

from dataset import MyDataset
from model import BaselineModel


def get_args():
    """
    解析命令行参数 - 配置训练和模型参数
    
    功能说明：
    该函数定义了模型训练所需的所有命令行参数，包括训练参数、
    模型结构参数和特征配置参数等。通过命令行参数可以灵活
    地控制训练过程和模型结构。
    
    参数分类：
    1. 训练参数：控制训练过程的超参数
    2. 模型参数：定义模型结构和正则化
    3. 特征参数：指定使用的多模态特征
    4. 运行参数：控制程序运行模式
    
    参数说明：
    - batch_size: 批次大小，影响训练速度和内存使用
    - lr: 学习率，控制模型参数更新步长
    - maxlen: 序列最大长度，控制输入序列的长度
    - hidden_units: 隐藏层维度，影响模型容量
    - num_blocks: Transformer块数量，影响模型深度
    - num_epochs: 训练轮数，控制训练总时长
    - num_heads: 注意力头数量，影响模型表达能力
    - dropout_rate: Dropout率，用于防止过拟合
    - l2_emb: Embedding L2正则化系数，用于防止过拟合
    - device: 计算设备，'cuda'或'cpu'
    - inference_only: 是否仅执行推理，不进行训练
    - state_dict_path: 模型检查点路径，用于断点续训
    - norm_first: 是否使用Pre-LN（先归一化）
    - mm_emb_id: 多模态特征ID列表，指定使用的特征
    
    Returns:
        args: 解析后的参数对象
    """
    parser = argparse.ArgumentParser()

    # 训练参数 - 控制训练过程
    parser.add_argument('--batch_size', default=128, type=int, help='批次大小，影响训练速度和内存使用')
    parser.add_argument('--lr', default=0.001, type=float, help='学习率，控制模型参数更新步长')
    parser.add_argument('--maxlen', default=101, type=int, help='序列最大长度，控制输入序列的长度')

    # 模型结构参数 - 定义模型架构
    parser.add_argument('--hidden_units', default=32, type=int, help='隐藏层维度，影响模型容量')
    parser.add_argument('--num_blocks', default=1, type=int, help='Transformer块数量，影响模型深度')
    parser.add_argument('--num_epochs', default=3, type=int, help='训练轮数，控制训练总时长')
    parser.add_argument('--num_heads', default=1, type=int, help='注意力头数量，影响模型表达能力')
    parser.add_argument('--dropout_rate', default=0.2, type=float, help='Dropout率，用于防止过拟合')
    parser.add_argument('--l2_emb', default=0.0, type=float, help='Embedding L2正则化系数，用于防止过拟合')
    parser.add_argument('--device', default='cuda', type=str, help='计算设备，"cuda"或"cpu"')
    parser.add_argument('--inference_only', action='store_true', help='是否仅执行推理，不进行训练')
    parser.add_argument('--state_dict_path', default=None, type=str, help='模型检查点路径，用于断点续训')
    parser.add_argument('--norm_first', action='store_true', help='是否使用Pre-LN（先归一化）')

    # 多模态特征参数 - 指定使用的特征
    parser.add_argument('--mm_emb_id', nargs='+', default=['81'], type=str,
                        choices=[str(s) for s in range(81, 87)],
                        help='多模态特征ID列表，指定使用的特征，可选范围81-86')

    args = parser.parse_args()

    return args


if __name__ == '__main__':
    """
    主程序入口 - 执行完整的模型训练流程
    
    执行流程：
    1. 环境初始化：创建日志目录、初始化日志记录器
    2. 参数解析：获取命令行参数
    3. 数据准备：加载和预处理训练数据
    4. 模型构建：初始化推荐模型和优化器
    5. 模型训练：执行训练循环
    6. 模型验证：在验证集上评估模型性能
    7. 模型保存：保存训练好的模型参数
    8. 资源清理：关闭日志记录器和文件
    
    环境变量：
    - TRAIN_LOG_PATH: 训练日志保存路径
    - TRAIN_TF_EVENTS_PATH: TensorBoard事件保存路径
    - TRAIN_DATA_PATH: 训练数据路径
    - TRAIN_CKPT_PATH: 模型检查点保存路径
    """
    # 创建日志目录
    Path(os.environ.get('TRAIN_LOG_PATH')).mkdir(parents=True, exist_ok=True)
    Path(os.environ.get('TRAIN_TF_EVENTS_PATH')).mkdir(parents=True, exist_ok=True)
    
    # 初始化日志记录器
    log_file = open(Path(os.environ.get('TRAIN_LOG_PATH'), 'train.log'), 'w')
    writer = SummaryWriter(os.environ.get('TRAIN_TF_EVENTS_PATH'))
    
    # 获取数据路径
    data_path = os.environ.get('TRAIN_DATA_PATH')

    # 解析命令行参数
    args = get_args()
    
    # 创建数据集
    dataset = MyDataset(data_path, args)
    
    # 划分训练集和验证集（90%训练，10%验证）
    train_dataset, valid_dataset = torch.utils.data.random_split(dataset, [0.9, 0.1])
    
    # 创建数据加载器
    train_loader = DataLoader(
        train_dataset, batch_size=args.batch_size, shuffle=True, num_workers=0, collate_fn=dataset.collate_fn
    )
    valid_loader = DataLoader(
        valid_dataset, batch_size=args.batch_size, shuffle=False, num_workers=0, collate_fn=dataset.collate_fn
    )
    
    # 获取数据集统计信息
    usernum, itemnum = dataset.usernum, dataset.itemnum
    feat_statistics, feat_types = dataset.feat_statistics, dataset.feature_types

    # 初始化模型
    model = BaselineModel(usernum, itemnum, feat_statistics, feat_types, args).to(args.device)

    # 模型参数初始化
    for name, param in model.named_parameters():
        try:
            # 使用Xavier正态分布初始化权重
            torch.nn.init.xavier_normal_(param.data)
        except Exception:
            # 某些参数可能不支持Xavier初始化，跳过异常
            pass

    # 将padding位置的Embedding初始化为零
    model.pos_emb.weight.data[0, :] = 0
    model.item_emb.weight.data[0, :] = 0
    model.user_emb.weight.data[0, :] = 0

    # 将稀疏特征的padding位置Embedding初始化为零
    for k in model.sparse_emb:
        model.sparse_emb[k].weight.data[0, :] = 0

    # 设置训练起始轮次
    epoch_start_idx = 1

    # 尝试加载预训练模型（断点续训）
    if args.state_dict_path is not None:
        try:
            # 加载模型状态字典
            model.load_state_dict(torch.load(args.state_dict_path, map_location=torch.device(args.device)))
            
            # 从文件名中提取训练轮次
            tail = args.state_dict_path[args.state_dict_path.find('epoch=') + 6 :]
            epoch_start_idx = int(tail[: tail.find('.')]) + 1
        except:
            print('failed loading state_dicts, pls check file path: ', end="")
            print(args.state_dict_path)
            raise RuntimeError('failed loading state_dicts, pls check file path!')

    # 定义损失函数和优化器
    bce_criterion = torch.nn.BCEWithLogitsLoss(reduction='mean')
    optimizer = torch.optim.Adam(model.parameters(), lr=args.lr, betas=(0.9, 0.98))

    # 初始化最佳性能指标
    best_val_ndcg, best_val_hr = 0.0, 0.0
    best_test_ndcg, best_test_hr = 0.0, 0.0
    T = 0.0
    t0 = time.time()
    global_step = 0
    
    print("Start training")
    
    # 训练循环
    for epoch in range(epoch_start_idx, args.num_epochs + 1):
        # 设置模型为训练模式
        model.train()
        
        # 如果是推理模式，跳过训练循环
        if args.inference_only:
            break
            
        # 训练批次循环
        for step, batch in tqdm(enumerate(train_loader), total=len(train_loader)):
            # 解包批次数据
            seq, pos, neg, token_type, next_token_type, next_action_type, seq_feat, pos_feat, neg_feat = batch
            
            # 将数据移动到指定设备
            seq = seq.to(args.device)
            pos = pos.to(args.device)
            neg = neg.to(args.device)
            
            # 前向传播
            pos_logits, neg_logits = model(
                seq, pos, neg, token_type, next_token_type, next_action_type, seq_feat, pos_feat, neg_feat
            )
            
            # 创建标签（正样本为1，负样本为0）
            pos_labels, neg_labels = torch.ones(pos_logits.shape, device=args.device), torch.zeros(
                neg_logits.shape, device=args.device
            )
            
            # 清零梯度
            optimizer.zero_grad()
            
            # 只计算点击样本（next_action_type == 1）的损失
            indices = np.where(next_token_type == 1)
            loss = bce_criterion(pos_logits[indices], pos_labels[indices])
            loss += bce_criterion(neg_logits[indices], neg_labels[indices])

            # 记录训练日志
            log_json = json.dumps(
                {'global_step': global_step, 'loss': loss.item(), 'epoch': epoch, 'time': time.time()}
            )
            log_file.write(log_json + '\n')
            log_file.flush()
            print(log_json)

            # 记录到TensorBoard
            writer.add_scalar('Loss/train', loss.item(), global_step)

            global_step += 1

            # 添加L2正则化项
            for param in model.item_emb.parameters():
                loss += args.l2_emb * torch.norm(param)
                
            # 反向传播和参数更新
            loss.backward()
            optimizer.step()

        # 验证阶段
        model.eval()
        valid_loss_sum = 0
        for step, batch in tqdm(enumerate(valid_loader), total=len(valid_loader)):
            # 解包批次数据
            seq, pos, neg, token_type, next_token_type, next_action_type, seq_feat, pos_feat, neg_feat = batch
            
            # 将数据移动到指定设备
            seq = seq.to(args.device)
            pos = pos.to(args.device)
            neg = neg.to(args.device)
            
            # 前向传播
            pos_logits, neg_logits = model(
                seq, pos, neg, token_type, next_token_type, next_action_type, seq_feat, pos_feat, neg_feat
            )
            
            # 创建标签（正样本为1，负样本为0）
            pos_labels, neg_labels = torch.ones(pos_logits.shape, device=args.device), torch.zeros(
                neg_logits.shape, device=args.device
            )
            
            # 只计算点击样本（next_action_type == 1）的损失
            indices = np.where(next_token_type == 1)
            loss = bce_criterion(pos_logits[indices], pos_labels[indices])
            loss += bce_criterion(neg_logits[indices], neg_labels[indices])
            
            # 累积验证损失
            valid_loss_sum += loss.item()
            
        # 计算平均验证损失
        valid_loss_sum /= len(valid_loader)
        
        # 记录验证损失到TensorBoard
        writer.add_scalar('Loss/valid', valid_loss_sum, global_step)

        # 保存模型检查点
        save_dir = Path(os.environ.get('TRAIN_CKPT_PATH'), f"global_step{global_step}.valid_loss={valid_loss_sum:.4f}")
        save_dir.mkdir(parents=True, exist_ok=True)
        torch.save(model.state_dict(), save_dir / "model.pt")

    print("Done")
    
    # 关闭日志记录器和文件
    writer.close()
    log_file.close()
```

## model.py

```python
import json
import pickle
import struct
from pathlib import Path

import numpy as np
import torch
from tqdm import tqdm


class MyDataset(torch.utils.data.Dataset):
    """
    用户序列数据集 - 基于Transformer的推荐系统数据集
    
    该数据集实现了用户行为序列的高效加载和预处理，支持多种特征类型：
    - 稀疏特征(Sparse): 类别型特征，通过Embedding处理
    - 数组特征(Array): 多值类别特征，通过Embedding后求和
    - 多模态特征(Emb): 预训练的向量特征，通过线性变换
    - 连续特征(Continual): 数值型特征，直接使用

    关键算法：
    1. 序列填充：使用left-padding策略，保持序列的时间顺序
    2. 负采样：使用随机负采样，确保负样本不在用户历史序列中
    3. 特征处理：支持多种特征类型的统一处理和缺失值填充

    Args:
        data_dir: 数据文件目录，包含seq.jsonl, item_feat_dict.json等
        args: 全局参数，包含maxlen, mm_emb_id等配置

    Attributes:
        data_dir: 数据文件目录
        maxlen: 最大长度，用于序列截断和填充
        item_feat_dict: 物品特征字典，key为item_id，value为特征字典
        mm_emb_ids: 激活的mm_emb特征ID列表，如['81', '82']
        mm_emb_dict: 多模态特征字典，key为特征ID，value为embedding字典
        itemnum: 物品数量，用于负采样范围
        usernum: 用户数量，用于数据集大小
        indexer_i_rev: 物品索引字典 (reid -> item_id)，用于反向查找
        indexer_u_rev: 用户索引字典 (reid -> user_id)，用于反向查找
        indexer: 索引字典，包含user、item、feature的映射关系
        feature_default_value: 特征缺省值，用于缺失特征填充
        feature_types: 特征类型，分为user和item的sparse, array, emb, continual类型
        feat_statistics: 特征统计信息，包括user和item的特征数量
    """

    def __init__(self, data_dir, args):
        """
        初始化数据集 - 加载所有必要的数据文件和索引
        
        实现trick：
        1. 使用文件偏移量优化：通过seq_offsets.pkl实现快速随机访问
        2. 延迟加载：只在__getitem__中动态加载用户数据，减少内存占用
        3. 多模态特征预加载：提前加载所有需要的多模态特征，避免训练时IO瓶颈（但是会导致embedding维度大的向量加载占用内存过多）
        
        加载的数据文件：
        - seq.jsonl: 用户行为序列数据
        - seq_offsets.pkl: 每行数据的文件偏移量
        - item_feat_dict.json: 物品特征字典
        - indexer.pkl: 用户、物品、特征的索引映射
        - creative_emb/: 多模态特征目录
        
        Args:
            data_dir: 数据文件目录路径
            args: 全局参数对象
        """
        super().__init__()
        self.data_dir = Path(data_dir)
        self._load_data_and_offsets()  # 加载序列数据和文件偏移量
        self.maxlen = args.maxlen
        self.mm_emb_ids = args.mm_emb_id

        # 加载物品特征字典，用于负采样时的特征获取
        self.item_feat_dict = json.load(open(Path(data_dir, "item_feat_dict.json"), 'r'))
        
        # 加载多模态特征，支持多种维度的预训练embedding
        self.mm_emb_dict = load_mm_emb(Path(data_dir, "creative_emb"), self.mm_emb_ids)
        
        # 加载索引字典，包含用户、物品、特征的映射关系
        with open(self.data_dir / 'indexer.pkl', 'rb') as ff:
            indexer = pickle.load(ff)
            self.itemnum = len(indexer['i'])  # 物品数量，用于负采样范围
            self.usernum = len(indexer['u'])  # 用户数量，用于数据集大小
        
        # 构建反向索引，用于从re-id查找原始ID
        self.indexer_i_rev = {v: k for k, v in indexer['i'].items()}
        self.indexer_u_rev = {v: k for k, v in indexer['u'].items()}
        self.indexer = indexer

        # 初始化特征信息，包括特征类型、缺省值和统计信息
        self.feature_default_value, self.feature_types, self.feat_statistics = self._init_feat_info()

    def _load_data_and_offsets(self):
        """
        加载用户序列数据和文件偏移量 - 实现高效的随机访问
        
        核心优化：
        1. 文件偏移量技术：预先计算每行数据的起始位置，避免逐行扫描
        2. 二进制模式：使用'rb'模式打开文件，提高读取效率
        3. 内存映射：通过seek()直接定位到目标行，减少IO时间
        
        复杂度分析：
        - 传统方法：O(n)时间复杂度，需要逐行扫描到目标行
        - 偏移量方法：O(1)时间复杂度，直接通过seek定位
        
        文件格式：
        - seq.jsonl: 每行一个用户的序列数据，JSON格式
        - seq_offsets.pkl: 每行对应的文件偏移量列表，pickle格式
        
        Trick说明：
        这种设计特别适合大规模数据集，当只需要访问部分用户数据时，
        可以避免加载整个数据集到内存，显著减少内存占用。
        """
        # 以二进制模式打开序列数据文件，支持随机访问
        self.data_file = open(self.data_dir / "seq.jsonl", 'rb')
        
        # 加载预计算的文件偏移量，key为用户ID，value为文件偏移量
        with open(Path(self.data_dir, 'seq_offsets.pkl'), 'rb') as f:
            self.seq_offsets = pickle.load(f)

    def _load_user_data(self, uid):
        """
        从数据文件中加载单个用户的数据 - 基于文件偏移量的高效访问
        
        实现原理：
        1. 使用文件偏移量直接定位到目标用户的数据行
        2. 读取单行JSON数据并解析为Python对象
        3. 支持动态加载，避免内存中保存所有用户数据
        
        数据格式说明：
        每行数据包含多个时间步的用户行为序列，每个时间步的格式为：
        [user_id, item_id, user_feature, item_feature, action_type, timestamp]
        
        特殊处理：
        - user_profile记录：item_id和item_feature为null
        - item_interaction记录：user_feature为null
        
        Args:
            uid: 用户ID(reid)，用于在seq_offsets中查找文件偏移量

        Returns:
            data: 用户序列数据，格式为[(user_id, item_id, user_feat, item_feat, action_type, timestamp)]
                  其中user_feat和item_feat为字典格式，key为特征ID，value为特征值
        """
        # 使用文件偏移量直接定位到目标用户的数据行
        self.data_file.seek(self.seq_offsets[uid])
        
        # 读取单行数据（JSON格式）
        line = self.data_file.readline()
        
        # 解析JSON数据为Python对象
        data = json.loads(line)
        return data

    def _random_neq(self, l, r, s):
        """
        生成一个不在序列s中的随机整数 - 用于训练时的负采样
        
        负采样策略：
        1. 随机采样：在指定范围内随机选择一个整数
        2. 冲突检测：确保采样结果不在用户历史序列中
        3. 有效性检查：确保采样结果在物品特征字典中存在
        
        数学表示：
        给定用户历史序列 S = {i₁, i₂, ..., iₙ}，负采样目标是找到物品 j 满足：
        - j ∉ S
        - j ∈ [l, r)
        - j ∈ item_feat_dict.keys()
        
        应用场景：
        在推荐系统的训练中，对于每个正样本(user, item, pos_item)，
        需要采样一个负样本(user, item, neg_item)，其中neg_item是用户
        未交互过的物品，用于计算对比损失。

        Args:
            l: 随机整数的最小值（包含）
            r: 随机整数的最大值（不包含）
            s: 用户历史序列集合，用于冲突检测

        Returns:
            t: 不在序列s中的随机整数，确保为有效的物品ID
        """
        # 在指定范围内随机采样
        t = np.random.randint(l, r)
        
        # 循环直到找到有效的负样本
        # 条件1：不在用户历史序列中
        # 条件2：在物品特征字典中存在（确保是有效物品）
        while t in s or str(t) not in self.item_feat_dict:
            t = np.random.randint(l, r)
        return t

    def __getitem__(self, uid):
        """
        获取单个用户的数据 - 核心数据预处理逻辑
        
        这是整个数据集最核心的方法，实现了：
        1. 序列构建：将原始用户行为序列转换为模型可用的格式
        2. 正负采样：为每个位置生成正样本和负样本
        3. 特征处理：处理多种类型的特征，包括缺失值填充
        4. 序列填充：使用left-padding策略处理变长序列
        
        关键算法：
        1. 序列表示：使用滑动窗口方式构建训练样本
           给定序列 [x₁, x₂, ..., xₙ]，构建样本 (x₁, x₂), (x₂, x₃), ..., (xₙ₋₁, xₙ)
           其中前者为上下文序列，后者为预测目标
        
        2. Left-padding策略：
           为了保持时间顺序，使用从右向左的填充方式
           原始序列：[A, B, C, D]
           填充后：[0, 0, A, B, C] (maxlen=5)
           
        3. 多类型特征处理：
           - 稀疏特征：转换为ID，通过Embedding处理
           - 数组特征：多值特征，通过Embedding后求和
           - 多模态特征：预训练向量，通过线性变换
           - 连续特征：直接使用数值

        Args:
            uid: 用户ID(reid)，用于加载用户数据

        Returns:
            seq: 用户序列ID，形状为 [maxlen+1]，包含用户历史交互的物品ID
            pos: 正样本ID，形状为 [maxlen+1]，下一个真实交互的物品ID
            neg: 负样本ID，形状为 [maxlen+1]，负采样得到的物品ID
            token_type: 用户序列类型，形状为 [maxlen+1]，1表示item，2表示user
            next_token_type: 下一个token类型，形状为 [maxlen+1]，1表示item，2表示user
            next_action_type: 下一个token动作类型，形状为 [maxlen+1]，0表示曝光，1表示点击
            seq_feat: 用户序列特征，形状为 [maxlen+1]，每个元素为特征字典
            pos_feat: 正样本特征，形状为 [maxlen+1]，每个元素为特征字典
            neg_feat: 负样本特征，形状为 [maxlen+1]，每个元素为特征字典
        """
        # 动态加载用户数据，避免内存中保存所有用户序列
        user_sequence = self._load_user_data(uid)

        # 扩展用户序列，分离用户画像和物品交互
        # Trick：将用户画像插入到序列开头，物品交互按时间顺序排列
        ext_user_sequence = []
        for record_tuple in user_sequence:
            u, i, user_feat, item_feat, action_type, _ = record_tuple
            if u and user_feat:
                # 用户画像记录，插入到序列开头（类型2）
                ext_user_sequence.insert(0, (u, user_feat, 2, action_type))
            if i and item_feat:
                # 物品交互记录，按时间顺序追加（类型1）
                ext_user_sequence.append((i, item_feat, 1, action_type))

        # 初始化输出数组，使用left-padding策略
        seq = np.zeros([self.maxlen + 1], dtype=np.int32)  # 序列ID
        pos = np.zeros([self.maxlen + 1], dtype=np.int32)  # 正样本ID
        neg = np.zeros([self.maxlen + 1], dtype=np.int32)  # 负样本ID
        token_type = np.zeros([self.maxlen + 1], dtype=np.int32)  # token类型
        next_token_type = np.zeros([self.maxlen + 1], dtype=np.int32)  # 下一个token类型
        next_action_type = np.zeros([self.maxlen + 1], dtype=np.int32)  # 下一个动作类型

        # 特征数组，使用object类型支持字典存储
        seq_feat = np.empty([self.maxlen + 1], dtype=object)
        pos_feat = np.empty([self.maxlen + 1], dtype=object)
        neg_feat = np.empty([self.maxlen + 1], dtype=object)

        # 使用滑动窗口方式构建训练样本
        nxt = ext_user_sequence[-1]  # 下一个token（预测目标）
        idx = self.maxlen  # 从数组末尾开始填充（left-padding）

        # 收集用户历史交互的物品ID，用于负采样冲突检测
        ts = set()
        for record_tuple in ext_user_sequence:
            if record_tuple[2] == 1 and record_tuple[0]:  # 只收集物品交互
                ts.add(record_tuple[0])

        # 核心算法：从后往前遍历，构建滑动窗口样本
        # Trick：这种处理方式保持了时间顺序，同时实现了left-padding
        for record_tuple in reversed(ext_user_sequence[:-1]):
            i, feat, type_, act_type = record_tuple  # 当前token
            next_i, next_feat, next_type, next_act_type = nxt  # 下一个token
            
            # 填充缺失特征
            feat = self.fill_missing_feat(feat, i)
            next_feat = self.fill_missing_feat(next_feat, next_i)
            
            # 填充当前位置的信息
            seq[idx] = i
            token_type[idx] = type_
            next_token_type[idx] = next_type
            if next_act_type is not None:
                next_action_type[idx] = next_act_type
            seq_feat[idx] = feat
            
            # 如果下一个token是物品且不为0，则生成正负样本
            if next_type == 1 and next_i != 0:
                pos[idx] = next_i
                pos_feat[idx] = next_feat
                # 负采样：确保不在用户历史序列中，ts为用户历史物品ID集合
                neg_id = self._random_neq(1, self.itemnum + 1, ts)
                neg[idx] = neg_id
                neg_feat[idx] = self.fill_missing_feat(self.item_feat_dict[str(neg_id)], neg_id)
            
            nxt = record_tuple  # 移动到前一个token
            idx -= 1
            if idx == -1:
                break

        # 处理缺失特征，用默认值填充
        seq_feat = np.where(seq_feat == None, self.feature_default_value, seq_feat)
        pos_feat = np.where(pos_feat == None, self.feature_default_value, pos_feat)
        neg_feat = np.where(neg_feat == None, self.feature_default_value, neg_feat)

        return seq, pos, neg, token_type, next_token_type, next_action_type, seq_feat, pos_feat, neg_feat

    def __len__(self):
        """
        返回数据集长度 - 即用户数量
        
        实现原理：
        通过seq_offsets的长度来确定用户数量，因为每个用户对应
        一个文件偏移量。这种设计避免了单独维护用户计数器，
        保证了数据的一致性。

        复杂度：O(1)时间复杂度，直接返回列表长度

        Returns:
            usernum: 用户数量，等于seq_offsets的长度
        """
        return len(self.seq_offsets)

    def _init_feat_info(self):
        """
        初始化特征信息 - 构建特征类型、缺省值和统计信息
        
        特征分类体系：
        1. 稀疏特征(Sparse): 单值类别特征，如用户性别、物品类别
           - 处理方式：转换为整数ID，通过Embedding层处理
           - 缺省值：0（对应padding_idx）
           
        2. 数组特征(Array): 多值类别特征，如用户标签、物品关键词
           - 处理方式：转换为ID列表，通过Embedding后求和
           - 缺省值：[0]（空列表）
           
        3. 多模态特征(Emb): 预训练向量特征，如图片、文本embedding
           - 处理方式：通过线性变换降维到hidden_units
           - 缺省值：零向量，维度与原embedding相同
           
        4. 连续特征(Continual): 数值型特征，如用户年龄、物品价格
           - 处理方式：直接使用或归一化
           - 缺省值：0

        特征ID说明：
        - 用户特征：103-109（用户画像相关）
        - 物品特征：100-122（物品属性相关）
        - 多模态特征：81-86（预训练embedding）

        Args:
            无参数，使用self.indexer和self.mm_emb_dict

        Returns:
            feat_default_value: 特征缺省值字典，key为特征ID，value为缺省值
            feat_types: 特征类型字典，key为类型名称，value为特征ID列表
            feat_statistics: 特征统计信息，key为特征ID，value为特征值数量
        """
        feat_default_value = {}
        feat_statistics = {}
        feat_types = {}
        
        # 定义用户稀疏特征：单值类别特征
        feat_types['user_sparse'] = ['103', '104', '105', '109']
        
        # 定义物品稀疏特征：单值类别特征
        feat_types['item_sparse'] = [
            '100', '117', '111', '118', '101', '102', '119', '120',
            '114', '112', '121', '115', '122', '116',
        ]
        
        # 定义物品数组特征：多值类别特征（当前为空）
        feat_types['item_array'] = []
        
        # 定义用户数组特征：多值类别特征
        feat_types['user_array'] = ['106', '107', '108', '110']
        
        # 定义多模态特征：预训练embedding特征
        feat_types['item_emb'] = self.mm_emb_ids
        
        # 定义连续特征：数值型特征（当前为空）
        feat_types['user_continual'] = []
        feat_types['item_continual'] = []

        # 为每种特征类型设置缺省值和统计信息
        for feat_id in feat_types['user_sparse']:
            feat_default_value[feat_id] = 0  # 稀疏特征缺省值为0
            feat_statistics[feat_id] = len(self.indexer['f'][feat_id])  # 特征值数量
            
        for feat_id in feat_types['item_sparse']:
            feat_default_value[feat_id] = 0
            feat_statistics[feat_id] = len(self.indexer['f'][feat_id])
            
        for feat_id in feat_types['item_array']:
            feat_default_value[feat_id] = [0]  # 数组特征缺省值为[0]
            feat_statistics[feat_id] = len(self.indexer['f'][feat_id])
            
        for feat_id in feat_types['user_array']:
            feat_default_value[feat_id] = [0]
            feat_statistics[feat_id] = len(self.indexer['f'][feat_id])
            
        for feat_id in feat_types['user_continual']:
            feat_default_value[feat_id] = 0  # 连续特征缺省值为0
            
        for feat_id in feat_types['item_continual']:
            feat_default_value[feat_id] = 0
            
        for feat_id in feat_types['item_emb']:
            # 多模态特征缺省值为零向量，维度与原embedding相同
            feat_default_value[feat_id] = np.zeros(
                list(self.mm_emb_dict[feat_id].values())[0].shape[0], dtype=np.float32
            )

        return feat_default_value, feat_types, feat_statistics

    def fill_missing_feat(self, feat, item_id):
        """
        填充缺失特征 - 确保所有特征字段都存在，处理缺失值
        
        核心功能：
        1. 缺失字段填充：为数据中缺失的特征字段添加缺省值
        2. 多模态特征处理：特殊处理多模态特征的加载和填充
        3. 特征完整性：确保返回的特征字典包含所有预定义的特征类型
        
        算法流程：
        1. 初始化：如果输入特征为None，创建空字典
        2. 复制：保留现有特征值
        3. 缺失检测：找出所有预定义但缺失的特征字段
        4. 填充：为缺失字段添加对应的缺省值
        5. 多模态处理：特殊处理多模态特征的动态加载

        多模态特征处理trick：
        - 多模态特征存储在mm_emb_dict中，需要动态加载
        - 通过item_id查找原始ID，再查找对应的embedding
        - 只有当embedding存在且为numpy数组时才进行填充

        Args:
            feat: 原始特征字典，可能包含部分特征字段
            item_id: 物品ID（reid形式），用于查找多模态特征

        Returns:
            filled_feat: 填充后的完整特征字典，包含所有预定义的特征字段
        """
        # 处理None输入，初始化为空字典
        if feat == None:
            feat = {}
        filled_feat = {}
        
        # 复制现有特征值
        for k in feat.keys():
            filled_feat[k] = feat[k]

        # 收集所有预定义的特征ID
        all_feat_ids = []
        for feat_type in self.feature_types.values():
            all_feat_ids.extend(feat_type)
        
        # 找出缺失的特征字段
        missing_fields = set(all_feat_ids) - set(feat.keys())
        
        # 为缺失字段填充缺省值
        for feat_id in missing_fields:
            filled_feat[feat_id] = self.feature_default_value[feat_id]
            
        # 特殊处理多模态特征：动态加载embedding
        for feat_id in self.feature_types['item_emb']:
            # 只对有效物品ID进行处理
            if item_id != 0 and self.indexer_i_rev[item_id] in self.mm_emb_dict[feat_id]:
                # 确保embedding存在且为numpy数组格式
                if type(self.mm_emb_dict[feat_id][self.indexer_i_rev[item_id]]) == np.ndarray:
                    filled_feat[feat_id] = self.mm_emb_dict[feat_id][self.indexer_i_rev[item_id]]

        return filled_feat

    @staticmethod
    def collate_fn(batch):
        """
        批处理函数 - 将多个样本合并为一个batch
        
        功能说明：
        1. 数据聚合：将多个__getitem__返回的样本合并为batch
        2. 类型转换：将numpy数组转换为torch.Tensor
        3. 维度处理：确保数据维度符合模型输入要求
        
        批处理策略：
        - ID类数据（seq, pos, neg等）：转换为torch.Tensor
        - 特征类数据（seq_feat, pos_feat, neg_feat）：保持list形式
          因为特征是字典结构，每个样本的特征字典可能不同，
          需要在模型中进行动态处理
        
        数据流：
        输入：[(sample1), (sample2), ..., (sampleN)]
        输出：(batch_seq, batch_pos, batch_neg, batch_token_type,
              batch_next_token_type, batch_next_action_type,
              batch_seq_feat, batch_pos_feat, batch_neg_feat)
              
        维度信息：
        - batch_seq: [batch_size, maxlen+1]
        - batch_pos: [batch_size, maxlen+1]
        - batch_neg: [batch_size, maxlen+1]
        - batch_token_type: [batch_size, maxlen+1]
        - batch_next_token_type: [batch_size, maxlen+1]
        - batch_next_action_type: [batch_size, maxlen+1]
        - batch_seq_feat: [batch_size]，每个元素为[maxlen+1]的字典列表
        - batch_pos_feat: [batch_size]，每个元素为[maxlen+1]的字典列表
        - batch_neg_feat: [batch_size]，每个元素为[maxlen+1]的字典列表

        Args:
            batch: 多个__getitem__返回的数据，每个元素为元组
                  (seq, pos, neg, token_type, next_token_type, next_action_type,
                   seq_feat, pos_feat, neg_feat)

        Returns:
            seq: 用户序列ID, torch.Tensor形式，形状为[batch_size, maxlen+1]
            pos: 正样本ID, torch.Tensor形式，形状为[batch_size, maxlen+1]
            neg: 负样本ID, torch.Tensor形式，形状为[batch_size, maxlen+1]
            token_type: 用户序列类型, torch.Tensor形式，形状为[batch_size, maxlen+1]
            next_token_type: 下一个token类型, torch.Tensor形式，形状为[batch_size, maxlen+1]
            next_action_type: 下一个token动作类型, torch.Tensor形式，形状为[batch_size, maxlen+1]
            seq_feat: 用户序列特征, list形式，长度为batch_size
            pos_feat: 正样本特征, list形式，长度为batch_size
            neg_feat: 负样本特征, list形式，长度为batch_size
        """
        # 解包batch数据
        seq, pos, neg, token_type, next_token_type, next_action_type, seq_feat, pos_feat, neg_feat = zip(*batch)
        
        # 将ID类数据转换为torch.Tensor
        seq = torch.from_numpy(np.array(seq))
        pos = torch.from_numpy(np.array(pos))
        neg = torch.from_numpy(np.array(neg))
        token_type = torch.from_numpy(np.array(token_type))
        next_token_type = torch.from_numpy(np.array(next_token_type))
        next_action_type = torch.from_numpy(np.array(next_action_type))
        
        # 特征类数据保持list形式，因为包含字典结构
        seq_feat = list(seq_feat)
        pos_feat = list(pos_feat)
        neg_feat = list(neg_feat)
        
        return seq, pos, neg, token_type, next_token_type, next_action_type, seq_feat, pos_feat, neg_feat


class MyTestDataset(MyDataset):
    """
    测试数据集 - 专门用于推理和评估的数据集
    
    与训练数据集的主要区别：
    1. 数据源：使用predict_seq.jsonl而不是seq.jsonl
    2. 样本构建：不需要生成正负样本，只需要构建用户序列
    3. 冷启动处理：支持处理训练集中未见过的新用户和新物品
    4. 输出格式：返回用户原始ID，便于结果评估
    
    应用场景：
    - 模型推理：生成用户推荐结果
    - 离线评估：计算推荐系统的性能指标
    - A/B测试：比较不同模型的效果
    
    设计考虑：
    1. 兼容性：继承MyDataset的大部分功能，复用特征处理逻辑
    2. 差异化：重写关键方法以适应测试场景的特殊需求
    3. 鲁棒性：处理测试数据中的各种异常情况
    """

    def __init__(self, data_dir, args):
        """
        初始化测试数据集
        
        与训练数据集的主要区别：
        1. 使用不同的数据文件：predict_seq.jsonl vs seq.jsonl
        2. 支持冷启动场景：处理训练集中未见过的新用户和新物品
        
        Args:
            data_dir: 测试数据目录
            args: 全局参数
        """
        super().__init__(data_dir, args)

    def _load_data_and_offsets(self):
        """
        加载测试数据 - 使用预测专用的数据文件
        
        与训练数据的区别：
        1. 文件名：predict_seq.jsonl vs seq.jsonl
        2. 偏移量文件：predict_seq_offsets.pkl vs seq_offsets.pkl
        3. 数据格式：可能包含训练集中未见过的新用户和新物品
        
        这种设计允许测试数据与训练数据分离，支持：
        - 标准测试：使用与训练分布相似的测试数据
        - 冷启动测试：包含新用户和新物品的测试数据
        - 时间序列测试：模拟真实场景中的时间推移
        """
        # 使用预测专用的序列数据文件
        self.data_file = open(self.data_dir / "predict_seq.jsonl", 'rb')
        
        # 加载对应的文件偏移量
        with open(Path(self.data_dir, 'predict_seq_offsets.pkl'), 'rb') as f:
            self.seq_offsets = pickle.load(f)

    def _process_cold_start_feat(self, feat):
        """
        处理冷启动特征 - 解决测试数据中的新特征值问题
        
        冷启动问题：
        在测试数据中，可能会出现训练集中未见过的新特征值，
        这些特征值通常以字符串形式出现，无法直接用于模型推理。
        
        处理策略：
        1. 字符串特征值：转换为缺省值0
        2. 列表中的字符串：逐个处理，将字符串替换为0
        3. 数值特征值：保持不变
        
        当前实现：
        采用简单的缺省值填充策略，将所有未知的特征值转换为0。
        这种方法简单有效，但可能不是最优的冷启动处理方案。
        
        改进方向：
        1. 基于相似度的填充：使用相似特征的特征值
        2. 概率模型：预测新特征值的可能分布
        3. 领域知识：根据业务规则设置合理的缺省值
        
        Args:
            feat: 原始特征字典，可能包含字符串形式的未知特征值

        Returns:
            processed_feat: 处理后的特征字典，所有特征值都转换为数值形式
        """
        processed_feat = {}
        for feat_id, feat_value in feat.items():
            if type(feat_value) == list:
                # 处理列表类型的特征值
                value_list = []
                for v in feat_value:
                    if type(v) == str:
                        # 列表中的字符串转换为0
                        value_list.append(0)
                    else:
                        value_list.append(v)
                processed_feat[feat_id] = value_list
            elif type(feat_value) == str:
                # 字符串特征值转换为0
                processed_feat[feat_id] = 0
            else:
                # 数值特征值保持不变
                processed_feat[feat_id] = feat_value
        return processed_feat

    def __getitem__(self, uid):
        """
        获取测试用户的数据 - 构建推理用的用户序列
        
        与训练数据的主要区别：
        1. 不需要生成正负样本：测试阶段只需要用户序列，不需要预测目标
        2. 支持冷启动：处理训练集中未见过的新用户和新物品
        3. 返回用户ID：返回原始用户ID，便于结果评估和对照
        
        核心算法：
        1. 序列构建：与训练数据类似，但不需要滑动窗口和正负采样
        2. 冷启动处理：新用户和新物品的特殊处理
        3. ID转换：支持原始ID和re-id之间的转换
        
        冷启动处理策略：
        - 新用户：将用户ID设置为0（padding），但保留原始user_id用于评估
        - 新物品：将物品ID设置为0，但保留原始特征信息
        - 新特征值：使用_process_cold_start_feat方法处理字符串形式的特征值

        Args:
            uid: 用户在self.data_file中储存的行号
            
        Returns:
            seq: 用户序列ID，形状为[maxlen+1]
            token_type: 用户序列类型，形状为[maxlen+1]，1表示item，2表示user
            seq_feat: 用户序列特征，形状为[maxlen+1]，每个元素为特征字典
            user_id: 用户原始ID，格式为"user_xxxxxx"，用于结果对照
        """
        user_sequence = self._load_user_data(uid)  # 动态加载用户数据

        ext_user_sequence = []
        user_id = None  # 初始化用户ID
        
        for record_tuple in user_sequence:
            u, i, user_feat, item_feat, _, _ = record_tuple
            
            # 处理用户ID：支持原始ID和re-id的转换
            if u:
                if type(u) == str:  # 如果是字符串，说明是原始user_id
                    user_id = u
                else:  # 如果是int，说明是re_id，需要转换为原始ID
                    user_id = self.indexer_u_rev[u]
                    
            # 处理用户画像记录
            if u and user_feat:
                if type(u) == str:
                    # 冷启动用户：将用户ID设置为0（padding）
                    u = 0
                if user_feat:
                    # 处理冷启动特征
                    user_feat = self._process_cold_start_feat(user_feat)
                ext_user_sequence.insert(0, (u, user_feat, 2))

            # 处理物品交互记录
            if i and item_feat:
                # 冷启动物品处理：如果物品ID超过训练时的物品数量，说明是新物品
                if i > self.itemnum:
                    i = 0  # 将新物品ID设置为0（padding）
                if item_feat:
                    # 处理冷启动特征
                    item_feat = self._process_cold_start_feat(item_feat)
                ext_user_sequence.append((i, item_feat, 1))

        # 初始化输出数组
        seq = np.zeros([self.maxlen + 1], dtype=np.int32)
        token_type = np.zeros([self.maxlen + 1], dtype=np.int32)
        seq_feat = np.empty([self.maxlen + 1], dtype=object)

        idx = self.maxlen

        # 收集用户历史交互的物品ID（用于特征处理）
        ts = set()
        for record_tuple in ext_user_sequence:
            if record_tuple[2] == 1 and record_tuple[0]:
                ts.add(record_tuple[0])

        # 使用left-padding策略构建序列
        for record_tuple in reversed(ext_user_sequence[:-1]):
            i, feat, type_ = record_tuple
            feat = self.fill_missing_feat(feat, i)
            seq[idx] = i
            token_type[idx] = type_
            seq_feat[idx] = feat
            idx -= 1
            if idx == -1:
                break

        # 处理缺失特征
        seq_feat = np.where(seq_feat == None, self.feature_default_value, seq_feat)

        return seq, token_type, seq_feat, user_id

    def __len__(self):
        """
        返回测试数据集长度 - 即测试用户数量
        
        与训练数据集的区别：
        1. 文件名：使用predict_seq_offsets.pkl而不是seq_offsets.pkl
        2. 动态加载：每次调用都重新加载文件，确保数据一致性
        
        设计考虑：
        测试数据可能会动态更新，所以每次都重新加载偏移量文件，
        而不是在初始化时缓存。这种设计保证了数据的实时性，
        但会带来一定的性能开销。如果测试数据固定不变，
        可以考虑在初始化时缓存偏移量数据。

        Returns:
            len(self.seq_offsets): 测试用户数量
        """
        # 动态加载测试数据的偏移量文件
        with open(Path(self.data_dir, 'predict_seq_offsets.pkl'), 'rb') as f:
            temp = pickle.load(f)
        return len(temp)

    @staticmethod
    def collate_fn(batch):
        """
        测试数据批处理函数 - 将多个测试样本合并为一个batch
        
        与训练数据批处理的主要区别：
        1. 数据量少：不需要处理正负样本和相关特征
        2. 包含用户ID：需要保留原始用户ID用于结果评估
        3. 输出格式：返回的数据结构更简单
        
        批处理策略：
        - 序列数据：转换为torch.Tensor，便于GPU加速
        - 特征数据：保持list形式，因为包含字典结构
        - 用户ID：保持原始字符串形式，用于结果对照
        
        应用场景：
        1. 批量推理：一次性处理多个用户的推荐请求
        2. 离线评估：批量计算推荐系统的性能指标
        3. 结果分析：将推荐结果与用户ID关联

        Args:
            batch: 多个__getitem__返回的数据，每个元素为元组
                  (seq, token_type, seq_feat, user_id)

        Returns:
            seq: 用户序列ID, torch.Tensor形式，形状为[batch_size, maxlen+1]
            token_type: 用户序列类型, torch.Tensor形式，形状为[batch_size, maxlen+1]
            seq_feat: 用户序列特征, list形式，长度为batch_size
            user_id: 用户原始ID, tuple形式，包含batch_size个字符串
        """
        # 解包batch数据
        seq, token_type, seq_feat, user_id = zip(*batch)
        
        # 将序列数据转换为torch.Tensor
        seq = torch.from_numpy(np.array(seq))
        token_type = torch.from_numpy(np.array(token_type))
        
        # 特征数据保持list形式
        seq_feat = list(seq_feat)

        return seq, token_type, seq_feat, user_id


def save_emb(emb, save_path):
    """
    将Embedding保存为二进制文件 - 高效的向量存储格式
    
    文件格式设计：
    1. 文件头：8字节，包含两个32位无符号整数
       - 前4字节：向量数量（num_points）
       - 后4字节：向量维度（num_dimensions）
    2. 数据体：连续存储的向量数据
       - 数据类型：float32
       - 存储顺序：行优先（C风格）
       - 总大小：num_points × num_dimensions × 4字节
    
    优势分析：
    1. 存储效率：二进制格式比文本格式节省空间
    2. 读取速度：内存映射加载，无需解析
    3. 随机访问：支持向量的随机访问
    4. 兼容性：标准的二进制格式，易于跨平台使用
    
    应用场景：
    1. 向量检索：保存物品embedding用于相似度计算
    2. 模型部署：将训练好的embedding保存为独立文件
    3. 结果缓存：缓存中间计算结果，避免重复计算
    
    文件结构示例：
    ```
    [文件头: 8字节]
    [向量1: 4×num_dimensions字节]
    [向量2: 4×num_dimensions字节]
    ...
    [向量N: 4×num_dimensions字节]
    ```

    Args:
        emb: 要保存的Embedding，形状为 [num_points, num_dimensions]的numpy数组
        save_path: 保存路径，字符串或Path对象
    """
    num_points = emb.shape[0]  # 数据点数量
    num_dimensions = emb.shape[1]  # 向量的维度
    print(f'saving {save_path}')
    
    # 以二进制写模式打开文件
    with open(Path(save_path), 'wb') as f:
        # 写入文件头：向量数量和向量维度
        # 使用'II'格式：两个32位无符号整数
        f.write(struct.pack('II', num_points, num_dimensions))
        
        # 写入向量数据：连续存储，float32格式
        emb.tofile(f)


def load_mm_emb(mm_path, feat_ids):
    """
    加载多模态特征Embedding - 支持多种预训练向量格式
    
    多模态特征说明：
    该系统支持6种多模态特征，每种特征有不同的维度和存储格式：
    - 特征81：32维，pickle格式存储
    - 特征82：1024维，JSON格式存储
    - 特征83：3584维，JSON格式存储
    - 特征84：4096维，JSON格式存储
    - 特征85：3584维，JSON格式存储
    - 特征86：3584维，JSON格式存储
    
    存储格式处理：
    1. JSON格式（特征82-86）：
       - 每个特征一个目录，包含多个JSON文件
       - 每行一个物品的embedding数据
       - 需要解析并转换为numpy数组
       
    2. Pickle格式（特征81）：
       - 单个pickle文件，包含完整的embedding字典
       - 直接加载即可使用
    
    性能优化：
    1. 批量处理：使用tqdm显示加载进度
    2. 异常处理：捕获并报告加载错误
    3. 内存管理：按需加载，避免一次性加载所有特征
    
    数据结构：
    ```
    mm_emb_dict = {
        '81': {item_id_1: emb_1, item_id_2: emb_2, ...},
        '82': {item_id_1: emb_1, item_id_2: emb_2, ...},
        ...
    }
    ```

    Args:
        mm_path: 多模态特征Embedding根目录路径
        feat_ids: 要加载的多模态特征ID列表，如['81', '82', '83']

    Returns:
        mm_emb_dict: 多模态特征Embedding字典，结构为：
                    {
                        feature_id: {
                            item_id: embedding_vector,
                            ...
                        },
                        ...
                    }
                    其中embedding_vector为numpy数组，dtype=np.float32
    """
    # 定义各特征的维度映射
    SHAPE_DICT = {"81": 32, "82": 1024, "83": 3584, "84": 4096, "85": 3584, "86": 3584}
    mm_emb_dict = {}
    
    # 逐个加载指定的多模态特征
    for feat_id in tqdm(feat_ids, desc='Loading mm_emb'):
        shape = SHAPE_DICT[feat_id]  # 获取特征维度
        emb_dict = {}
        
        # 特殊处理特征81（pickle格式）
        if feat_id != '81':
            try:
                # JSON格式处理：遍历目录下的所有JSON文件
                base_path = Path(mm_path, f'emb_{feat_id}_{shape}')
                for json_file in base_path.glob('*.json'):
                    with open(json_file, 'r', encoding='utf-8') as file:
                        for line in file:
                            # 解析每行JSON数据
                            data_dict_origin = json.loads(line.strip())
                            insert_emb = data_dict_origin['emb']
                            
                            # 将列表转换为numpy数组
                            if isinstance(insert_emb, list):
                                insert_emb = np.array(insert_emb, dtype=np.float32)
                            
                            # 构建embedding字典
                            data_dict = {data_dict_origin['anonymous_cid']: insert_emb}
                            emb_dict.update(data_dict)
            except Exception as e:
                print(f"transfer error: {e}")
        
        # 特征81使用pickle格式
        if feat_id == '81':
            with open(Path(mm_path, f'emb_{feat_id}_{shape}.pkl'), 'rb') as f:
                emb_dict = pickle.load(f)
        
        # 将加载的特征添加到结果字典
        mm_emb_dict[feat_id] = emb_dict
        print(f'Loaded #{feat_id} mm_emb')
    
    return mm_emb_dict
```

## model.py

```python
from pathlib import Path

import numpy as np
import torch
import torch.nn.functional as F
from tqdm import tqdm

from dataset import save_emb


class FlashMultiHeadAttention(torch.nn.Module):
    """
    Flash多头注意力机制 - 高优化的注意力实现
    
    核心算法：
    多头注意力将输入序列投影到多个子空间，在每个子空间中独立计算注意力，
    然后将结果拼接并投影回原始空间。这种设计允许模型同时关注
    序列中的不同位置和不同表示子空间。
    
    数学表示：
    Q_i = X * W_Q_i, K_i = X * W_K_i, V_i = X * W_V_i
    Attention(Q_i, K_i, V_i) = softmax(Q_i * K_i^T / sqrt(d_k)) * V_i
    MultiHead(X) = Concat(Attention(Q_1, K_1, V_1), ..., Attention(Q_h, K_h, V_h)) * W_O
    
    优化特性：
    1. Flash Attention：使用PyTorch 2.0+的优化实现，显著提升计算效率
    2. 内存高效：通过优化的kernel减少内存使用
    3. 自动降级：在不支持Flash Attention的环境中自动使用标准实现
    
    性能对比：
    - 标准注意力：O(N²)时间和空间复杂度
    - Flash Attention：O(N)空间复杂度，显著减少内存占用
    
    Args:
        hidden_units: 隐藏层维度，必须能被num_heads整除
        num_heads: 注意力头数量
        dropout_rate: 注意力权重的dropout率
    """
    def __init__(self, hidden_units, num_heads, dropout_rate):
        super(FlashMultiHeadAttention, self).__init__()

        self.hidden_units = hidden_units
        self.num_heads = num_heads
        self.head_dim = hidden_units // num_heads  # 每个注意力头的维度
        self.dropout_rate = dropout_rate

        # 确保hidden_units能被num_heads整除
        assert hidden_units % num_heads == 0, "hidden_units must be divisible by num_heads"

        # Q, K, V的线性变换层
        self.q_linear = torch.nn.Linear(hidden_units, hidden_units)
        self.k_linear = torch.nn.Linear(hidden_units, hidden_units)
        self.v_linear = torch.nn.Linear(hidden_units, hidden_units)
        self.out_linear = torch.nn.Linear(hidden_units, hidden_units)

    def forward(self, query, key, value, attn_mask=None):
        """
        前向传播 - 实现优化的多头注意力计算
        
        计算流程：
        1. 线性变换：将输入投影到Q, K, V空间
        2. 形状重塑：将张量重塑为多头格式
        3. 注意力计算：使用Flash Attention或标准注意力
        4. 输出处理：重塑并应用输出线性变换
        
        维度变化：
        输入: [batch_size, seq_len, hidden_units]
        线性变换后: [batch_size, seq_len, hidden_units]
        多头重塑后: [batch_size, num_heads, seq_len, head_dim]
        注意力计算后: [batch_size, num_heads, seq_len, head_dim]
        输出: [batch_size, seq_len, hidden_units]
        
        Args:
            query: 查询向量，形状为[batch_size, seq_len, hidden_units]
            key: 键向量，形状为[batch_size, seq_len, hidden_units]
            value: 值向量，形状为[batch_size, seq_len, hidden_units]
            attn_mask: 注意力掩码，形状为[batch_size, seq_len, seq_len]

        Returns:
            output: 注意力输出，形状为[batch_size, seq_len, hidden_units]
            attention_weights: 注意力权重（当前实现返回None）
        """
        batch_size, seq_len, _ = query.size()

        # 计算Q, K, V向量
        Q = self.q_linear(query)
        K = self.k_linear(key)
        V = self.v_linear(value)

        # 重塑为multi-head格式：[batch_size, num_heads, seq_len, head_dim]
        Q = Q.view(batch_size, seq_len, self.num_heads, self.head_dim).transpose(1, 2)
        K = K.view(batch_size, seq_len, self.num_heads, self.head_dim).transpose(1, 2)
        V = V.view(batch_size, seq_len, self.num_heads, self.head_dim).transpose(1, 2)

        # 优先使用Flash Attention（PyTorch 2.0+）
        if hasattr(F, 'scaled_dot_product_attention'):
            # Flash Attention实现：更高效的内存使用和计算速度
            attn_output = F.scaled_dot_product_attention(
                Q, K, V,
                dropout_p=self.dropout_rate if self.training else 0.0,
                attn_mask=attn_mask.unsqueeze(1)
            )
        else:
            # 降级到标准注意力机制
            scale = (self.head_dim) ** -0.5  # 缩放因子，防止梯度消失
            scores = torch.matmul(Q, K.transpose(-2, -1)) * scale

            # 应用注意力掩码
            if attn_mask is not None:
                scores.masked_fill_(attn_mask.unsqueeze(1).logical_not(), float('-inf'))

            # 计算注意力权重并应用dropout
            attn_weights = F.softmax(scores, dim=-1)
            attn_weights = F.dropout(attn_weights, p=self.dropout_rate, training=self.training)
            attn_output = torch.matmul(attn_weights, V)

        # 重塑回原来的格式：[batch_size, seq_len, hidden_units]
        attn_output = attn_output.transpose(1, 2).contiguous().view(batch_size, seq_len, self.hidden_units)

        # 应用输出线性变换
        output = self.out_linear(attn_output)

        return output, None


class PointWiseFeedForward(torch.nn.Module):
    """
    逐点前馈网络 - Transformer中的核心组件之一
    
    网络结构：
    实现了一个两层的全连接网络，中间使用ReLU激活函数和dropout。
    使用1D卷积来模拟全连接层，这种实现在某些情况下更高效。
    
    数学表示：
    FFN(x) = max(0, x * W1 + b1) * W2 + b2
    其中W1, W2是权重矩阵，b1, b2是偏置向量
    
    设计考虑：
    1. 卷积实现：使用kernel_size=1的1D卷积代替全连接层
       这种实现方式在GPU上通常有更好的性能
    2. 维度处理：需要在通道和序列长度维度之间进行转置
    3. 正则化：使用dropout防止过拟合
    
    维度变化：
    输入: [batch_size, seq_len, hidden_units]
    转置后: [batch_size, hidden_units, seq_len]
    Conv1D后: [batch_size, hidden_units, seq_len]
    转置后: [batch_size, seq_len, hidden_units]
    
    Args:
        hidden_units: 隐藏层维度
        dropout_rate: dropout率
    """
    def __init__(self, hidden_units, dropout_rate):
        super(PointWiseFeedForward, self).__init__()

        # 第一个1D卷积层，相当于全连接层
        self.conv1 = torch.nn.Conv1d(hidden_units, hidden_units, kernel_size=1)
        self.dropout1 = torch.nn.Dropout(p=dropout_rate)
        self.relu = torch.nn.ReLU()
        
        # 第二个1D卷积层，相当于全连接层
        self.conv2 = torch.nn.Conv1d(hidden_units, hidden_units, kernel_size=1)
        self.dropout2 = torch.nn.Dropout(p=dropout_rate)

    def forward(self, inputs):
        """
        前向传播 - 实现逐点前馈网络
        
        计算流程：
        1. 维度转置：将序列长度维度和隐藏维度交换
        2. 第一个卷积：应用第一个全连接变换
        3. 激活函数：应用ReLU非线性变换
        4. Dropout：应用dropout正则化
        5. 第二个卷积：应用第二个全连接变换
        6. Dropout：应用dropout正则化
        7. 维度恢复：转置回原始维度顺序
        
        维度变换技巧：
        - Conv1D期望输入格式为(N, C, L)，其中N是batch_size，
          C是通道数（hidden_units），L是序列长度
        - 需要在forward开始和结束时进行维度转置
        
        Args:
            inputs: 输入张量，形状为[batch_size, seq_len, hidden_units]

        Returns:
            outputs: 输出张量，形状为[batch_size, seq_len, hidden_units]
        """
        # 转置维度以适应Conv1D的输入格式要求
        # [batch_size, seq_len, hidden_units] -> [batch_size, hidden_units, seq_len]
        inputs_transposed = inputs.transpose(-1, -2)
        
        # 第一个全连接层 + ReLU + Dropout
        x = self.conv1(inputs_transposed)
        x = self.dropout1(x)
        x = self.relu(x)
        
        # 第二个全连接层 + Dropout
        x = self.conv2(x)
        outputs = self.dropout2(x)
        
        # 恢复原始维度顺序
        # [batch_size, hidden_units, seq_len] -> [batch_size, seq_len, hidden_units]
        outputs = outputs.transpose(-1, -2)
        
        return outputs


class BaselineModel(torch.nn.Module):
    """
    基于Transformer的推荐系统基线模型
    
    模型架构：
    该模型实现了一个多特征的Transformer推荐系统，支持用户和物品的
    多种特征类型，包括稀疏特征、数组特征、多模态特征和连续特征。
    
    核心组件：
    1. Embedding层：处理用户ID、物品ID和位置编码
    2. 特征处理：统一处理多种类型的特征
    3. Transformer编码器：使用Flash Attention优化的多层Transformer
    4. 特征融合：通过DNN融合多种特征表示
    
    关键算法：
    1. 特征嵌入：将不同类型的特征转换为稠密向量
    2. 位置编码：使用可学习的位置编码捕获序列顺序信息
    3. 自注意力：通过多头注意力机制捕获序列内部依赖关系
    4. 特征融合：将多种特征表示融合为统一的用户/物品表示
    
    优化特性：
    1. Flash Attention：显著提升长序列的处理效率
    2. 批处理特征：统一处理多种特征类型，减少计算开销
    3. 内存优化：使用高效的张量操作减少内存占用

    Args:
        user_num: 用户数量
        item_num: 物品数量
        feat_statistics: 特征统计信息，key为特征ID，value为特征数量
        feat_types: 各个特征的特征类型，key为特征类型名称，value为包含的特征ID列表
        args: 全局参数，包含hidden_units, num_blocks等配置

    Attributes:
        user_num: 用户数量
        item_num: 物品数量
        dev: 计算设备（CPU/GPU）
        norm_first: 是否先归一化（Pre-LN或Post-LN）
        maxlen: 序列最大长度
        item_emb: 物品ID的Embedding表
        user_emb: 用户ID的Embedding表
        pos_emb: 位置编码的Embedding表
        sparse_emb: 稀疏特征的Embedding表
        emb_transform: 多模态特征的线性变换层
        userdnn: 用户特征融合的全连接层
        itemdnn: 物品特征融合的全连接层
    """

    def __init__(self, user_num, item_num, feat_statistics, feat_types, args):
        """
        初始化基线模型 - 构建完整的推荐系统架构
        
        构建流程：
        1. 基础配置：设置模型参数和设备
        2. Embedding层：初始化用户、物品和位置编码的Embedding
        3. 特征处理：初始化各种特征的Embedding和变换层
        4. Transformer层：构建多层Transformer编码器
        5. 特征融合：构建特征融合的DNN层
        
        设计考虑：
        1. 参数共享：某些特征类型可以共享Embedding参数
        2. 维度对齐：确保所有特征的输出维度一致
        3. 正则化：使用dropout和LayerNorm防止过拟合
        4. 优化：使用Flash Attention提升计算效率
        
        Args:
            user_num: 用户数量，用于用户Embedding表的大小
            item_num: 物品数量，用于物品Embedding表的大小
            feat_statistics: 特征统计信息，用于确定Embedding表的大小
            feat_types: 特征类型信息，用于构建特征处理层
            args: 全局参数，包含模型配置信息
        """
        super(BaselineModel, self).__init__()

        # 基础配置
        self.user_num = user_num
        self.item_num = item_num
        self.dev = args.device
        self.norm_first = args.norm_first  # Pre-LN或Post-LN
        self.maxlen = args.maxlen
        
        # TODO: 可以添加L2正则化项来正则化Embedding向量
        # loss += args.l2_emb * (||embedding||^2)

        # 核心Embedding表
        self.item_emb = torch.nn.Embedding(self.item_num + 1, args.hidden_units, padding_idx=0)
        self.user_emb = torch.nn.Embedding(self.user_num + 1, args.hidden_units, padding_idx=0)
        self.pos_emb = torch.nn.Embedding(2 * args.maxlen + 1, args.hidden_units, padding_idx=0)
        self.emb_dropout = torch.nn.Dropout(p=args.dropout_rate)
        
        # 特征处理层
        self.sparse_emb = torch.nn.ModuleDict()  # 稀疏和数组特征的Embedding
        self.emb_transform = torch.nn.ModuleDict()  # 多模态特征的线性变换

        # Transformer编码器层
        self.attention_layernorms = torch.nn.ModuleList()  # 注意力层的LayerNorm
        self.attention_layers = torch.nn.ModuleList()  # 多头注意力层
        self.forward_layernorms = torch.nn.ModuleList()  # 前馈网络的LayerNorm
        self.forward_layers = torch.nn.ModuleList()  # 前馈网络层

        # 初始化特征信息
        self._init_feat_info(feat_statistics, feat_types)

        # 计算用户和物品特征的维度
        # 用户特征维度 = (稀疏特征数量 + 用户ID + 数组特征数量) * hidden_units + 连续特征数量
        userdim = args.hidden_units * (len(self.USER_SPARSE_FEAT) + 1 + len(self.USER_ARRAY_FEAT)) + len(
            self.USER_CONTINUAL_FEAT
        )
        # 物品特征维度 = (稀疏特征数量 + 物品ID + 数组特征数量) * hidden_units + 连续特征数量 + 多模态特征维度
        itemdim = (
            args.hidden_units * (len(self.ITEM_SPARSE_FEAT) + 1 + len(self.ITEM_ARRAY_FEAT))
            + len(self.ITEM_CONTINUAL_FEAT)
            + args.hidden_units * len(self.ITEM_EMB_FEAT)
        )

        # 特征融合DNN层
        self.userdnn = torch.nn.Linear(userdim, args.hidden_units)
        self.itemdnn = torch.nn.Linear(itemdim, args.hidden_units)

        # 输出LayerNorm
        self.last_layernorm = torch.nn.LayerNorm(args.hidden_units, eps=1e-8)

        # 构建多层Transformer编码器
        for _ in range(args.num_blocks):
            # 注意力层的LayerNorm
            new_attn_layernorm = torch.nn.LayerNorm(args.hidden_units, eps=1e-8)
            self.attention_layernorms.append(new_attn_layernorm)

            # 多头注意力层（使用Flash Attention优化）
            new_attn_layer = FlashMultiHeadAttention(
                args.hidden_units, args.num_heads, args.dropout_rate
            )
            self.attention_layers.append(new_attn_layer)

            # 前馈网络的LayerNorm
            new_fwd_layernorm = torch.nn.LayerNorm(args.hidden_units, eps=1e-8)
            self.forward_layernorms.append(new_fwd_layernorm)

            # 前馈网络层
            new_fwd_layer = PointWiseFeedForward(args.hidden_units, args.dropout_rate)
            self.forward_layers.append(new_fwd_layer)

        # 初始化稀疏特征的Embedding表
        for k in self.USER_SPARSE_FEAT:
            self.sparse_emb[k] = torch.nn.Embedding(self.USER_SPARSE_FEAT[k] + 1, args.hidden_units, padding_idx=0)
        for k in self.ITEM_SPARSE_FEAT:
            self.sparse_emb[k] = torch.nn.Embedding(self.ITEM_SPARSE_FEAT[k] + 1, args.hidden_units, padding_idx=0)
        for k in self.ITEM_ARRAY_FEAT:
            self.sparse_emb[k] = torch.nn.Embedding(self.ITEM_ARRAY_FEAT[k] + 1, args.hidden_units, padding_idx=0)
        for k in self.USER_ARRAY_FEAT:
            self.sparse_emb[k] = torch.nn.Embedding(self.USER_ARRAY_FEAT[k] + 1, args.hidden_units, padding_idx=0)
            
        # 初始化多模态特征的线性变换层
        for k in self.ITEM_EMB_FEAT:
            self.emb_transform[k] = torch.nn.Linear(self.ITEM_EMB_FEAT[k], args.hidden_units)

    def _init_feat_info(self, feat_statistics, feat_types):
        """
        初始化特征信息 - 构建特征处理所需的字典结构
        
        功能说明：
        该方法将原始的特征统计信息和特征类型转换为模型内部使用的字典格式，
        便于后续的特征Embedding表构建和特征处理。
        
        特征分类体系：
        1. 稀疏特征(Sparse)：单值类别特征，如用户性别、物品类别
           - 用户稀疏特征：USER_SPARSE_FEAT
           - 物品稀疏特征：ITEM_SPARSE_FEAT
           
        2. 数组特征(Array)：多值类别特征，如用户标签、物品关键词
           - 用户数组特征：USER_ARRAY_FEAT
           - 物品数组特征：ITEM_ARRAY_FEAT
           
        3. 多模态特征(Emb)：预训练向量特征，如图片、文本embedding
           - 物品多模态特征：ITEM_EMB_FEAT
           
        4. 连续特征(Continual)：数值型特征，如用户年龄、物品价格
           - 用户连续特征：USER_CONTINUAL_FEAT
           - 物品连续特征：ITEM_CONTINUAL_FEAT
        
        多模态特征维度：
        - 特征81：32维（可能是用户画像特征）
        - 特征82：1024维（可能是文本特征）
        - 特征83：3584维（可能是图像特征）
        - 特征84：4096维（可能是高维图像特征）
        - 特征85：3584维（可能是视频特征）
        - 特征86：3584维（可能是多模态融合特征）
        
        Args:
            feat_statistics: 特征统计信息，key为特征ID，value为特征值数量
            feat_types: 特征类型信息，key为类型名称，value为特征ID列表
        """
        # 用户稀疏特征：单值类别特征，用于Embedding查找
        self.USER_SPARSE_FEAT = {k: feat_statistics[k] for k in feat_types['user_sparse']}
        
        # 用户连续特征：数值型特征，直接使用
        self.USER_CONTINUAL_FEAT = feat_types['user_continual']
        
        # 物品稀疏特征：单值类别特征，用于Embedding查找
        self.ITEM_SPARSE_FEAT = {k: feat_statistics[k] for k in feat_types['item_sparse']}
        
        # 物品连续特征：数值型特征，直接使用
        self.ITEM_CONTINUAL_FEAT = feat_types['item_continual']
        
        # 用户数组特征：多值类别特征，Embedding后求和
        self.USER_ARRAY_FEAT = {k: feat_statistics[k] for k in feat_types['user_array']}
        
        # 物品数组特征：多值类别特征，Embedding后求和
        self.ITEM_ARRAY_FEAT = {k: feat_statistics[k] for k in feat_types['item_array']}
        
        # 多模态特征维度映射
        EMB_SHAPE_DICT = {"81": 32, "82": 1024, "83": 3584, "84": 4096, "85": 3584, "86": 3584}
        
        # 物品多模态特征：预训练向量，需要线性变换
        self.ITEM_EMB_FEAT = {k: EMB_SHAPE_DICT[k] for k in feat_types['item_emb']}

    def feat2tensor(self, seq_feature, k):
        """
        将特征字典转换为张量 - 统一处理不同类型的特征
        
        功能说明：
        该方法将序列中的特征字典转换为PyTorch张量，支持稀疏特征和数组特征
        的不同处理方式。这是特征处理流水线的关键步骤。
        
        处理策略：
        1. 数组特征(Array)：多值类别特征，需要二级padding
           - 第一级：序列长度padding（max_seq_len）
           - 第二级：数组长度padding（max_array_len）
           - 输出形状：[batch_size, max_seq_len, max_array_len]
           
        2. 稀疏特征(Sparse)：单值类别特征，只需要序列长度padding
           - 输出形状：[batch_size, max_seq_len]
        
        算法复杂度：
        - 时间复杂度：O(batch_size * max_seq_len * max_array_len)
        - 空间复杂度：O(batch_size * max_seq_len * max_array_len)
        
        优化考虑：
        1. 预分配内存：先计算最大长度，然后一次性分配numpy数组
        2. 批量处理：避免循环中的小内存分配
        3. 最小拷贝：只拷贝实际需要的数据
        
        Args:
            seq_feature: 序列特征列表，形状为[batch_size, maxlen]，
                        每个元素为特征字典{feature_id: feature_value}
            k: 特征ID，指定要处理的特征

        Returns:
            batch_data: 特征值的张量，形状为：
                       - 数组特征：[batch_size, max_seq_len, max_array_len]
                       - 稀疏特征：[batch_size, max_seq_len]
        """
        batch_size = len(seq_feature)

        # 判断特征类型并选择处理策略
        if k in self.ITEM_ARRAY_FEAT or k in self.USER_ARRAY_FEAT:
            # 数组特征处理：多值类别特征，需要二级padding
            
            # 计算最大序列长度和最大数组长度
            max_array_len = 0
            max_seq_len = 0

            for i in range(batch_size):
                seq_data = [item[k] for item in seq_feature[i]]
                max_seq_len = max(max_seq_len, len(seq_data))
                max_array_len = max(max_array_len, max(len(item_data) for item_data in seq_data))

            # 预分配numpy数组，填充0值
            batch_data = np.zeros((batch_size, max_seq_len, max_array_len), dtype=np.int64)
            
            # 填充实际数据，处理长度不一致的情况
            for i in range(batch_size):
                seq_data = [item[k] for item in seq_feature[i]]
                for j, item_data in enumerate(seq_data):
                    actual_len = min(len(item_data), max_array_len)
                    batch_data[i, j, :actual_len] = item_data[:actual_len]

            return torch.from_numpy(batch_data).to(self.dev)
        else:
            # 稀疏特征处理：单值类别特征，只需要序列长度padding
            
            # 计算最大序列长度
            max_seq_len = max(len(seq_feature[i]) for i in range(batch_size))
            
            # 预分配numpy数组，填充0值
            batch_data = np.zeros((batch_size, max_seq_len), dtype=np.int64)

            # 填充实际数据
            for i in range(batch_size):
                seq_data = [item[k] for item in seq_feature[i]]
                batch_data[i] = seq_data

            return torch.from_numpy(batch_data).to(self.dev)

    def feat2emb(self, seq, feature_array, mask=None, include_user=False):
        """
        特征转换为Embedding - 核心特征处理流水线
        
        功能说明：
        该方法实现了多种特征类型的统一Embedding处理，是整个模型的
        核心特征处理组件。它将原始特征数据转换为稠密的向量表示。
        
        处理流程：
        1. ID特征Embedding：处理用户ID和物品ID
        2. 稀疏特征Embedding：处理单值类别特征
        3. 数组特征Embedding：处理多值类别特征
        4. 多模态特征变换：处理预训练向量特征
        5. 连续特征处理：处理数值型特征
        6. 特征融合：将多种特征表示融合为统一表示
        
        特征处理策略：
        1. 稀疏特征：Embedding查找 -> [batch_size, seq_len, hidden_units]
        2. 数组特征：Embedding查找 -> 求和 -> [batch_size, seq_len, hidden_units]
        3. 多模态特征：线性变换 -> [batch_size, seq_len, hidden_units]
        4. 连续特征：直接使用 -> [batch_size, seq_len, 1]
        
        应用场景控制：
        include_user参数控制是否包含用户特征：
        - True：处理用户序列（包含用户画像和物品交互）
        - False：仅处理物品特征（用于正负样本和候选库生成）
        
        Args:
            seq: 序列ID张量，形状为[batch_size, seq_len]
            feature_array: 特征数组，形状为[batch_size, seq_len]，
                          每个元素为特征字典{feature_id: feature_value}
            mask: 掩码张量，形状为[batch_size, seq_len]，
                  1表示item，2表示user，0表示padding
            include_user: 是否包含用户特征，布尔值

        Returns:
            seqs_emb: 序列特征的Embedding，形状为[batch_size, seq_len, hidden_units]
        """
        seq = seq.to(self.dev)
        
        # 预计算ID特征Embedding
        if include_user:
            # 用户序列处理：包含用户画像和物品交互
            user_mask = (mask == 2).to(self.dev)
            item_mask = (mask == 1).to(self.dev)
            user_embedding = self.user_emb(user_mask * seq)  # 用户ID Embedding
            item_embedding = self.item_emb(item_mask * seq)  # 物品ID Embedding
            item_feat_list = [item_embedding]
            user_feat_list = [user_embedding]
        else:
            # 仅物品特征处理：用于正负样本和候选库生成
            item_embedding = self.item_emb(seq)  # 物品ID Embedding
            item_feat_list = [item_embedding]

        # 批量处理所有特征类型
        all_feat_types = [
            (self.ITEM_SPARSE_FEAT, 'item_sparse', item_feat_list),
            (self.ITEM_ARRAY_FEAT, 'item_array', item_feat_list),
            (self.ITEM_CONTINUAL_FEAT, 'item_continual', item_feat_list),
        ]

        # 如果需要，添加用户特征类型
        if include_user:
            all_feat_types.extend(
                [
                    (self.USER_SPARSE_FEAT, 'user_sparse', user_feat_list),
                    (self.USER_ARRAY_FEAT, 'user_array', user_feat_list),
                    (self.USER_CONTINUAL_FEAT, 'user_continual', user_feat_list),
                ]
            )

        # 批量处理每种特征类型
        for feat_dict, feat_type, feat_list in all_feat_types:
            if not feat_dict:
                continue

            for k in feat_dict:
                # 将特征字典转换为张量
                tensor_feature = self.feat2tensor(feature_array, k)

                # 根据特征类型选择处理方式
                if feat_type.endswith('sparse'):
                    # 稀疏特征：直接Embedding查找
                    feat_list.append(self.sparse_emb[k](tensor_feature))
                elif feat_type.endswith('array'):
                    # 数组特征：Embedding查找后求和
                    feat_list.append(self.sparse_emb[k](tensor_feature).sum(2))
                elif feat_type.endswith('continual'):
                    # 连续特征：增加维度以保持一致性
                    feat_list.append(tensor_feature.unsqueeze(2))

        # 处理多模态特征：预训练向量特征
        for k in self.ITEM_EMB_FEAT:
            # 收集所有数据到numpy数组，然后批量转换
            batch_size = len(feature_array)
            emb_dim = self.ITEM_EMB_FEAT[k]
            seq_len = len(feature_array[0])

            # 预分配张量内存
            batch_emb_data = np.zeros((batch_size, seq_len, emb_dim), dtype=np.float32)

            # 填充多模态特征数据
            for i, seq in enumerate(feature_array):
                for j, item in enumerate(seq):
                    if k in item:
                        batch_emb_data[i, j] = item[k]

            # 批量转换并传输到GPU
            tensor_feature = torch.from_numpy(batch_emb_data).to(self.dev)
            item_feat_list.append(self.emb_transform[k](tensor_feature))

        # 融合所有特征
        all_item_emb = torch.cat(item_feat_list, dim=2)
        all_item_emb = torch.relu(self.itemdnn(all_item_emb))  # 物品特征融合
        
        if include_user:
            # 包含用户特征时，融合用户和物品特征
            all_user_emb = torch.cat(user_feat_list, dim=2)
            all_user_emb = torch.relu(self.userdnn(all_user_emb))  # 用户特征融合
            seqs_emb = all_item_emb + all_user_emb  # 特征相加融合
        else:
            # 仅物品特征
            seqs_emb = all_item_emb
        return seqs_emb

    def log2feats(self, log_seqs, mask, seq_feature):
        """
        序列转换为特征表示 - Transformer编码器的核心实现
        
        功能说明：
        该方法实现了完整的Transformer编码器流程，将用户行为序列
        转换为丰富的特征表示。这是整个模型的核心推理组件。
        
        处理流程：
        1. 特征嵌入：将ID和特征转换为稠密向量
        2. 位置编码：添加位置信息捕获序列顺序
        3. 注意力掩码：构建因果掩码和padding掩码
        4. Transformer编码：通过多层Transformer处理序列
        5. 输出归一化：应用最终的LayerNorm
        
        关键算法：
        1. 特征缩放：将特征向量乘以embedding维度的平方根，
           防止点积值过大导致梯度消失
           
        2. 位置编码：使用可学习的位置编码，为每个位置
           分配唯一的向量表示
           
        3. 注意力掩码：组合两种掩码
           - 因果掩码：确保当前位置只能关注之前的位置
           - Padding掩码：忽略padding位置的注意力计算
           
        4. Pre-LN/Post-LN：根据norm_first参数选择
           - Pre-LN：先LayerNorm再注意力/前馈网络
           - Post-LN：先注意力/前馈网络再LayerNorm
           
        维度变化：
        输入: [batch_size, seq_len] (ID序列)
        特征嵌入后: [batch_size, seq_len, hidden_units]
        位置编码后: [batch_size, seq_len, hidden_units]
        Transformer后: [batch_size, seq_len, hidden_units]
        输出: [batch_size, seq_len, hidden_units]
        
        Args:
            log_seqs: 序列ID张量，形状为[batch_size, seq_len]
            mask: token类型掩码，形状为[batch_size, seq_len]，
                  1表示item token，2表示user token，0表示padding
            seq_feature: 序列特征列表，形状为[batch_size, seq_len]，
                          每个元素为特征字典{feature_id: feature_value}

        Returns:
            seqs_emb: 序列的特征表示，形状为[batch_size, seq_len, hidden_units]
        """
        batch_size = log_seqs.shape[0]
        maxlen = log_seqs.shape[1]
        
        # 1. 特征嵌入：将ID和特征转换为稠密向量
        seqs = self.feat2emb(log_seqs, seq_feature, mask=mask, include_user=True)
        
        # 2. 特征缩放：乘以embedding维度的平方根，防止点积值过大
        seqs *= self.item_emb.embedding_dim**0.5
        
        # 3. 位置编码：为序列中的每个位置添加位置信息
        poss = torch.arange(1, maxlen + 1, device=self.dev).unsqueeze(0).expand(batch_size, -1).clone()
        poss *= log_seqs != 0  # padding位置的位置编码为0
        seqs += self.pos_emb(poss)  # 添加位置编码
        
        # 4. 应用dropout正则化
        seqs = self.emb_dropout(seqs)

        # 5. 构建注意力掩码：组合因果掩码和padding掩码
        maxlen = seqs.shape[1]
        ones_matrix = torch.ones((maxlen, maxlen), dtype=torch.bool, device=self.dev)
        attention_mask_tril = torch.tril(ones_matrix)  # 因果掩码：下三角矩阵
        attention_mask_pad = (mask != 0).to(self.dev)  # padding掩码：非padding位置为True
        attention_mask = attention_mask_tril.unsqueeze(0) & attention_mask_pad.unsqueeze(1)

        # 6. 多层Transformer编码器处理
        for i in range(len(self.attention_layers)):
            if self.norm_first:
                # Pre-LN：先LayerNorm再注意力/前馈网络
                # 注意力层
                x = self.attention_layernorms[i](seqs)
                mha_outputs, _ = self.attention_layers[i](x, x, x, attn_mask=attention_mask)
                seqs = seqs + mha_outputs  # 残差连接
                
                # 前馈网络层
                seqs = seqs + self.forward_layers[i](self.forward_layernorms[i](seqs))
            else:
                # Post-LN：先注意力/前馈网络再LayerNorm
                # 注意力层
                mha_outputs, _ = self.attention_layers[i](seqs, seqs, seqs, attn_mask=attention_mask)
                seqs = self.attention_layernorms[i](seqs + mha_outputs)  # 残差连接后LayerNorm
                
                # 前馈网络层
                seqs = self.forward_layernorms[i](seqs + self.forward_layers[i](seqs))

        # 7. 输出LayerNorm归一化
        log_feats = self.last_layernorm(seqs)

        return log_feats

    def forward(
        self, user_item, pos_seqs, neg_seqs, mask, next_mask, next_action_type, seq_feature, pos_feature, neg_feature
    ):
        """
        训练前向传播 - 计算正负样本的logits用于对比损失
        
        功能说明：
        该方法实现了训练时的前向传播流程，通过对比学习的方式
        计算用户序列与正负样本的相似度，用于后续的损失计算。
        
        核心算法：
        1. 序列编码：将用户行为序列编码为特征表示
        2. 样本嵌入：将正负样本转换为特征表示
        3. 相似度计算：通过点积计算用户序列与样本的相似度
        4. 损失掩码：只对物品token计算损失，忽略用户token
        
        数学表示：
        给定用户序列特征表示 H ∈ [batch_size, seq_len, hidden_units]
        正样本特征表示 P ∈ [batch_size, seq_len, hidden_units]
        负样本特征表示 N ∈ [batch_size, seq_len, hidden_units]
        
        正样本相似度：S_pos = sum(H * P, dim=-1) ∈ [batch_size, seq_len]
        负样本相似度：S_neg = sum(H * N, dim=-1) ∈ [batch_size, seq_len]
        
        损失掩码：M = (next_mask == 1) ∈ [batch_size, seq_len]
        最终logits：L_pos = S_pos * M, L_neg = S_neg * M
        
        损失函数：
        通常使用BCEWithLogitsLoss：
        Loss = -[M * log(sigmoid(L_pos)) + M * log(1 - sigmoid(L_neg))]
        
        训练策略：
        1. 对比学习：通过正负样本的对比学习用户偏好
        2. 序列预测：预测用户下一个可能交互的物品
        3. 多任务：同时考虑曝光和点击两种行为类型
        
        Args:
            user_item: 用户序列ID，形状为[batch_size, seq_len]
            pos_seqs: 正样本序列ID，形状为[batch_size, seq_len]
            neg_seqs: 负样本序列ID，形状为[batch_size, seq_len]
            mask: token类型掩码，形状为[batch_size, seq_len]，
                  1表示item token，2表示user token，0表示padding
            next_mask: 下一个token类型掩码，形状为[batch_size, seq_len]，
                       1表示item token，2表示user token，0表示padding
            next_action_type: 下一个token动作类型，形状为[batch_size, seq_len]，
                              0表示曝光，1表示点击
            seq_feature: 序列特征列表，形状为[batch_size, seq_len]，
                         每个元素为特征字典{feature_id: feature_value}
            pos_feature: 正样本特征列表，形状为[batch_size, seq_len]，
                        每个元素为特征字典{feature_id: feature_value}
            neg_feature: 负样本特征列表，形状为[batch_size, seq_len]，
                        每个元素为特征字典{feature_id: feature_value}

        Returns:
            pos_logits: 正样本logits，形状为[batch_size, seq_len]，
                       表示用户序列与正样本的相似度
            neg_logits: 负样本logits，形状为[batch_size, seq_len]，
                       表示用户序列与负样本的相似度
        """
        # 1. 序列编码：将用户行为序列编码为特征表示
        log_feats = self.log2feats(user_item, mask, seq_feature)
        
        # 2. 构建损失掩码：只对物品token计算损失，忽略用户token
        loss_mask = (next_mask == 1).to(self.dev)

        # 3. 样本嵌入：将正负样本转换为特征表示
        # 注意：正负样本都是物品，不需要包含用户特征
        pos_embs = self.feat2emb(pos_seqs, pos_feature, include_user=False)
        neg_embs = self.feat2emb(neg_seqs, neg_feature, include_user=False)

        # 4. 相似度计算：通过点积计算用户序列与样本的相似度
        # 点积相似度：logits[i,j] = sum(log_feats[i,j] * embs[i,j])
        pos_logits = (log_feats * pos_embs).sum(dim=-1)
        neg_logits = (log_feats * neg_embs).sum(dim=-1)
        
        # 5. 应用损失掩码：只对物品位置计算损失
        pos_logits = pos_logits * loss_mask
        neg_logits = neg_logits * loss_mask

        return pos_logits, neg_logits

    def predict(self, log_seqs, seq_feature, mask):
        """
        推理阶段预测 - 计算用户序列的综合表征
        
        功能说明：
        该方法实现了推理阶段的用户表征计算，用于生成用户推荐。
        与训练阶段的主要区别是：
        1. 不需要正负样本：只计算用户序列的表征
        2. 使用最后一个位置：使用序列最后一个位置的表征作为用户综合表征
        3. 简化流程：不需要计算相似度和损失
        
        应用场景：
        1. 候选生成：使用用户表征计算与候选物品的相似度
        2. 向量检索：将用户表征存入向量数据库用于相似度检索
        3. 在线推理：实时计算用户表征用于推荐
        
        核心算法：
        1. 序列编码：使用与训练相同的Transformer编码器
        2. 位置选择：使用序列最后一个有效位置的表征
           - 最后一个位置包含了整个序列的汇总信息
           - 通过自注意力机制，最后一个位置能够捕获
             序列中所有位置的重要信息
        3. 用户表征：将选定的位置表征作为用户综合表征
        
        维度变化：
        输入序列: [batch_size, seq_len]
        编码后: [batch_size, seq_len, hidden_units]
        输出表征: [batch_size, hidden_units]
        
        推理优化：
        1. 批处理：支持批量计算多个用户的表征
        2. 内存高效：不需要存储正负样本的中间结果
        3. 计算复用：复用训练时的特征处理和Transformer编码逻辑
        
        Args:
            log_seqs: 用户序列ID，形状为[batch_size, seq_len]
            seq_feature: 序列特征列表，形状为[batch_size, seq_len]，
                         每个元素为特征字典{feature_id: feature_value}
            mask: token类型掩码，形状为[batch_size, seq_len]，
                  1表示item token，2表示user token，0表示padding

        Returns:
            final_feat: 用户序列的综合表征，形状为[batch_size, hidden_units]，
                       用于后续的推荐计算
        """
        # 1. 序列编码：使用与训练相同的Transformer编码器
        log_feats = self.log2feats(log_seqs, mask, seq_feature)

        # 2. 位置选择：选择序列最后一个位置的表征作为用户综合表征
        # 最后一个位置包含了整个序列的汇总信息，通过自注意力机制
        # 能够捕获序列中所有位置的重要信息
        final_feat = log_feats[:, -1, :]

        return final_feat

    def save_item_emb(self, item_ids, retrieval_ids, feat_dict, save_path, batch_size=1024):
        """
        生成候选库物品embedding - 用于向量检索和推荐
        
        功能说明：
        该方法实现了物品embedding的批量生成和保存，用于构建
        向量检索系统。这是推荐系统中离线处理的关键步骤。
        
        应用场景：
        1. 向量检索：将物品embedding存入向量数据库（如FAISS）
        2. 候选生成：通过向量相似度快速筛选候选物品
        3. 近似检索：支持大规模物品库的高效近似最近邻搜索
        
        处理流程：
        1. 批量处理：将大量物品分成小批次处理，避免内存溢出
        2. 特征嵌入：使用与训练相同的特征处理逻辑
        3. 格式转换：将PyTorch张量转换为numpy数组
        4. 文件保存：保存为二进制格式，支持高效读取
        
        文件格式：
        1. embedding.fbin：物品embedding的二进制文件
           - 格式：[num_items, embedding_dim]的float32数组
           - 用途：向量相似度计算和检索
           
        2. id.u64bin：物品ID的二进制文件
           - 格式：[num_items, 1]的uint64数组
           - 用途：embedding与原始ID的映射
        
        性能优化：
        1. 批处理：通过batch_size控制内存使用
        2. GPU加速：在GPU上进行特征嵌入计算
        3. 内存管理：及时释放不需要的中间变量
        4. 并行处理：使用tqdm显示进度，支持长时间运行
        
        数学表示：
        对于每个物品i，其embedding计算为：
        emb_i = DNN(Concat(emb_id(i), emb_feat1(i), emb_feat2(i), ...))
        
        其中：
        - emb_id(i)：物品ID的embedding
        - emb_featk(i)：第k个特征的embedding
        - DNN：特征融合的全连接网络
        - Concat：特征拼接操作
        
        Args:
            item_ids: 候选物品ID列表（re-id形式），用于特征查找
            retrieval_ids: 候选物品ID列表（检索ID，从0开始编号），用于检索脚本
            feat_dict: 训练集所有物品特征字典，key为物品ID，value为特征字典
            save_path: 保存路径，embedding和ID文件将保存在此目录下
            batch_size: 批次大小，控制内存使用，默认为1024

        Returns:
            无返回值，结果直接保存到文件
        """
        all_embs = []

        # 批量处理所有物品，避免内存溢出
        for start_idx in tqdm(range(0, len(item_ids), batch_size), desc="Saving item embeddings"):
            end_idx = min(start_idx + batch_size, len(item_ids))

            # 准备当前批次的物品ID和特征
            item_seq = torch.tensor(item_ids[start_idx:end_idx], device=self.dev).unsqueeze(0)
            batch_feat = []
            for i in range(start_idx, end_idx):
                batch_feat.append(feat_dict[i])

            batch_feat = np.array(batch_feat, dtype=object)

            # 计算当前批次的物品embedding
            # 使用与训练相同的特征处理逻辑，但不包含用户特征
            batch_emb = self.feat2emb(item_seq, [batch_feat], include_user=False).squeeze(0)

            # 将结果转移到CPU并转换为numpy数组
            all_embs.append(batch_emb.detach().cpu().numpy().astype(np.float32))

        # 合并所有批次的结果
        final_ids = np.array(retrieval_ids, dtype=np.uint64).reshape(-1, 1)
        final_embs = np.concatenate(all_embs, axis=0)
        
        # 保存为二进制文件，支持高效读取
        save_emb(final_embs, Path(save_path, 'embedding.fbin'))
        save_emb(final_ids, Path(save_path, 'id.u64bin'))
```