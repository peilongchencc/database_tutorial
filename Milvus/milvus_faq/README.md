# Milvus使用FAQ:
- [Milvus使用FAQ:](#milvus使用faq)
  - [如何提升检索速度:](#如何提升检索速度)
    - [问题描述:](#问题描述)
    - [解决方案:](#解决方案)
  - [pymilvus如何批量查询:](#pymilvus如何批量查询)
    - [问题描述:](#问题描述-1)
    - [解决方案:](#解决方案-1)
  - [flush()后create\_index()卡住:](#flush后create_index卡住)
  - [text字段的模糊匹配无法实现:](#text字段的模糊匹配无法实现)
  - [重启milvus后没有数据:](#重启milvus后没有数据)
    - [问题描述:](#问题描述-2)
    - [解决方案:](#解决方案-2)
    - [笔者实践:](#笔者实践)


## 如何提升检索速度:

### 问题描述:

我使用的是Ubuntu18.4，我使用的是CPU版本Milvus，Milvus中存储了50万条向量数据，我的向量都是使用Electra转化的。我检索一条数据的耗时竟然需要0.2s，是否有方法提升检索速度？使用维度更低的模型转换词向量如何？我的数据都是中文文本。<br>

### 解决方案:

Milvus 是一个开源的向量数据库，用于存储、检索和分析向量数据，特别适合处理大规模、高维度的数据。如果你在使用 Milvus 进行检索操作时遇到了性能瓶颈，以下是一些可能的优化策略：

1. **优化索引类型和参数**：
   Milvus 支持多种索引类型，如 IVF_FLAT、IVF_SQ8、HNSW 等。每种索引类型都有其优缺点，并且可以根据数据特性进行调整。例如，对于高维数据，HNSW 可能是一个更好的选择。另外，优化索引的构建参数，如 nlist 或 M，也可能提高检索效率。

2. **调整查询参数**：
   当执行查询时，可以调整 `top_k` 和 `nprobe` 参数。`nprobe` 是在索引中搜索的桶的数量，增加 `nprobe` 可以提高检索精度，但会减慢检索速度。根据需求调整 `nprobe` 可以在速度与准确性之间取得平衡。

3. **减少向量维度**：
   使用维度更低的模型来转换词向量可能会加快检索速度，因为低维向量占用的空间更少，计算距离更快。但是，这也可能会影响检索结果的质量。可以考虑使用一些降维技术，如 PCA 或 t-SNE，但这应该在确保不会过度损害向量表示的情况下进行。

4. **优化硬件使用**：
   即便你使用的是CPU版本的Milvus，但确保你的服务器或者你的环境具有足够的CPU资源来处理这些请求。如果你的CPU资源紧张，这可能会导致查询速度下降。

5. **批量查询**：
   如果可能的话，将多个查询批量化可以显著提高吞吐量，因为这样可以在单个查询操作中复用更多的资源和计算。

6. **更新Milvus版本**：
   如果你使用的是旧版本的Milvus，考虑升级到最新版本。随着项目的发展，新版本通常会包含性能改进和新特性。

7. **分析瓶颈**：
   使用工具（如 Milvus 的监控系统、Linux 的 perf 工具等）来识别瓶颈所在。这可能是 CPU、内存、磁盘 I/O 或者网络。

8. **合理设置预加载数据量**：
   通过 `load_collection` 接口预加载数据到内存中可以提高检索速度。但是，预加载过多的数据可能会导致内存不足，从而影响性能。

最后，具体优化措施需要基于你的实际情况（如查询类型、系统资源、数据特性等）来决定。建议从监控工具和日志中收集信息，逐一尝试以上建议，并观察每次调整对性能的影响。


## pymilvus如何批量查询:

### 问题描述:

我使用的是Ubuntu18.4，我使用的是CPU版本Milvus，我使用的是pymilvus 2.x版本，我应该如何使用pymilvus进行批量查询？<br>

### 解决方案:

在使用`pymilvus`进行批量查询时，主要步骤是创建一个搜索参数字典，并使用`search`函数进行查询。以下是使用pymilvus 2.x版本进行批量查询的大致步骤：<br>

1. **连接到Milvus服务器**：首先，确保你已经启动了Milvus服务，然后使用`pymilvus`连接到服务。

2. **准备数据**：准备你想要查询的向量数据。

3. **构造查询**：构造一个搜索参数字典，指定你想要的搜索参数，例如查询向量、topK（返回最近邻的数量）和指标类型等。

4. **执行搜索**：使用集合（collection）对象的`search`方法执行查询。

5. **处理结果**：处理返回的结果，这通常包括距离分数和对应的ID。

以下是一个简单的示例代码：<br>

```python
from pymilvus import (
    connections,
    utility,
    FieldSchema, CollectionSchema, DataType,
    Collection,
)

# 第1步：连接到Milvus服务器
connections.connect("default", host='127.0.0.1', port='19530')

# 检查连接是否成功
print(f"Connected to Milvus: {utility.ping('default')}")

# 第2步：指定你的集合名称
collection_name = 'example_collection'

# 这里假设你的集合已经存在并且你已经知道字段名称和参数
# 如果需要，也可以创建新的集合

# 第3步：准备你的查询向量
# 这应该是一个二维数组，其中每个内部数组是一个查询向量
query_vectors = [[0.1, 0.2, ...], [0.2, 0.3, ...], ...]

# 第4步：构造查询
# 在这里指定你想要搜索的参数
search_params = {
    "metric_type": "L2",  # 或者其他距离度量类型，如 "IP"（内积）
    "params": {"nprobe": 10},
}

# 执行批量查询
# topK是返回结果的数量，len(query_vectors)是查询的向量数
results = collection.search(
    data=query_vectors,
    anns_field='embedding_field',  # 这里的'embedding_field'应该替换为你的向量字段名称
    param=search_params,
    limit=10,  # topK值
    expr=None,
    partition_names=None,
    output_fields=None,
    timeout=None,
    **kwargs
)

# 第5步：处理结果
# 打印查询结果
for result in results:
    for hit in result:
        print(f"ID: {hit.id}, Distance: {hit.distance}")

# 断开连接
connections.disconnect("default")
```

请根据你的集合和字段进行相应的替换。特别是，你需要替换`example_collection`为你的集合名称，`embedding_field`为包含向量的字段名称。<br>

还要注意，这只是一个示例，你可能需要根据你的集合配置和搜索需求调整代码中的参数。例如，你可能需要根据实际情况调整连接参数、搜索参数等。<br>


## flush()后create_index()卡住:

建议使用`top`指令查看CPU状态，查看内存情况，以笔者为例:<br>

| PID  | USER | PR | NI |    VIRT  |   RES  |  SHR | S | %CPU | %MEM |    TIME+   | COMMAND          |
|------|------|----|----|----------|--------|------|---|------|------|------------|------------------|
| 5374 | root | 20 |  0 |  973748  | 66088  | 3176 | R |100.3 |  0.4 | 78390:32   | node             |
| 5825 | root | 20 |  0 |  975028  | 65504  | 1640 | R |100.3 |  0.4 | 78385:25   | node             |
| 5677 | root | 20 |  0 |  833204  | 63768  | 1500 | R |100.0 |  0.4 | 78389:38   | node             |
|29422 | root | 20 |  0 | 8926256  | 846316 | 94696| S |  3.7 |  5.2 |  823:52.92 | milvus           |
|  843 | root | 10 |-10 |  154384  | 17160  |  816 | S |  1.3 |  0.1 |  1092:45   | AliYunDunMonito  |
|29018 | root | 20 |  0 |  720756  | 10196  | 7676 | S |  0.7 |  0.1 |    4:16.82 | containerd-shim  |

😡😡😡笔者在运行下列代码时，发现每次运行都卡在`create_index`不动，思索了好久，才发现是CPU占用的问题，我在运行下列代码的时候CPU占用为400%，而我租用的服务器一共只有4个核，所以会卡住，**处于等待状态**。<br>

```python
# ...省略上文
# 假设插入100条数据，维度为128
data = [
    [i for i in range(100)],
    [[random.random() for _ in range(128)] for _ in range(100)],
]
# 插入数据
demo_collection.insert(data)

# 刷新数据
demo_collection.flush()

# 建立索引
index_param = {
    "index_type": 'IVF_FLAT',
    "params": {"nlist": 10},
    "metric_type": 'L2'}
print(f"开始为向量建立索引，索引建立较慢，请稍等... ...")
demo_collection.create_index('float_vector_field', index_param)
print("\nCreated index:\n{}".format(demo_collection.index().params))

print("\n数据中的实体数量为:")
print(demo_collection.num_entities)
# ...省略下文
```

于是，我果断kill掉了前3个node进程，毕竟是租的服务器，而且我知道我的代码现在用不到nodejs的内容。<br>

如果你也有类似的情况，建议先使用`ps`命令查看这些进程(PID)的命令行参数，这可以提供一些关于进程是如何启动的信息。例如：<br>

```bash
ps -f -p 5374,5825,5677
```

这将显示每个进程的完整命令行，你可能能从中看到它们启动的脚本或是应用程序的名称。如果实在不知道，可能就需要问问同事了。<br>


## text字段的模糊匹配无法实现:

当前，笔者使用的是 `milvus v2.3.2`，该版本不支持模糊匹配，只支持前缀匹配。<br>

假设你的 `text`字段记录的是文本数据，如果你想要检索含有 "老师" 的数据("我的老师...")是无法做到的，只能支持检索到前缀为 "老师" 的文本，即 "老师很..."。<br>

代码为:<br>

```python
from config.server_settings import Milvus_Server_Config
from pymilvus import Collection, connections

def milvus_connection():
    """建立milvus连接(milvus默认为连接池形式)
    Ps: milvus的连接不需要返回值
    """
    connections.connect(host = Milvus_Server_Config['host'], port = Milvus_Server_Config['port'])

def delete_data_in_milvus_according_expr(expression, milvus_collection_name):
    """根据表达式删除milvus数据
    Args:
        expression(str): 布尔表达式, 请使用 milvus 支持的布尔表达式,例如: expr = "text == '货币三佳是t+1到账吗'"
        milvus_collection_name(str): milvus集合的名称,例如: 'standard_financial_question_collection'
    Return:
        无返回值
    """
    # 建立milvus连接,无返回值
    milvus_connection()
    # 连接milvus集合
    milvus_collection = Collection(name=milvus_collection_name)
    # 传入Expression,使用布尔表达式删除数据
    milvus_collection.delete(expression)
    # 提交更改
    milvus_collection.load()

if __name__ == "__main__":
    expr="text LIKE '老师%'"  # 只支持前缀匹配，不支持模糊匹配
    milvus_collection_name = 'standard_collection'
    delete_data_in_milvus_according_expr(expr, milvus_collection_name)
```


## 重启milvus后没有数据:

### 问题描述:

我的同事无意间重装了服务器的docker，我的milvus被迫停了，我重新启动了milvus，这时候milvus中的数据全都没有了，这是怎么回事？<br>

### 解决方案:

查看milvus的 `docker-compose.yml` 文件，查看其中 `volumes` 目录有没有数据，或者目录路径有没有变动。<br>

### 笔者实践:

笔者查看自己的 `volumes` 后发现，笔者之前自定义了 `volumes` 目录，但后来移动了 `volumes` 目录。<br>

同事重新安装了服务器的docker后，重启milvus时，连接的是之前的 `volumes` 目录，所以数据为空。笔者将 `docker-compose.yml` 文件中 `volumes` 目录修改正确，然后重新启动milvus就有数据了。<br>