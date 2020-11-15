# 实验记录
**整理者**：Ai<br>
项目分三大模块开展：EDA、数据预处理及模型。
## 一、EDA
此块后续补充。
## 二、数据预处理
截止到2020.11.11，data_preprocess.ipynb已更新至第4版，当前最新版本实现的预处理手段如下：
1. 时间格式转换：将DOTTING_TIME原格式转为UNIX时间格式(eda.ipynb已做)<br>
2. 处理数据缺失<br>
3. 处理重复样本<br>
4. 特征工程1：把‘CU’队列规格与'CPU_USAGE'和'MEM_USAGE'结合得到'USED_CPU'已用CPU量和‘USED_MEM’已用内存量<br>
5. 特征工程2：差分‘时间间隔’和‘其它连续型特征’<br>
6. 特征工程3：在每个时间点的样本特征中，加入历史5个时间间隔的特征值<br>
7. 特征工程4：加入行内统计特征，即考虑2/3/4/5阶的历史平均值,最大值,最小值,标准差<br>
8. 类别型变量独热编码

之前尝试过几个预处理手段，透过模型表现，得到以下一些结论：<br>
1. **LightGBM没必要对连续型变量使用标准化**。因为树模型不是利用SGD进行收敛优化，且LGBM利用特征直方图寻找最优特征及分裂点时，只关注取值顺序，值标准化不影响各样本的取值顺序，所以不需要。我试过训练集和测试集合并后，不分队列标准化和分队列标准化，结果都不是特别理想，其中分队列标准化得分更低，原因分队列标准化是在一个相对较小的样本空间内进行的，使得标准化结果不全面，比如内存使用率正常是0-100，而有些队列内存使用率最高是50，而其他队列有达到100的，这样前者队列标准化和后者标准化就不一样，可模型参数是队列共享的，这样的话，数据和模型参数没有保持好的一致性，导致最后模型表现较差。这是一个失败的尝试，但总的来说，树不需要标准化。而如果使用其他需要标准化数据的模型，一定要考虑数据和模型参数共享的一致性。<br>
2. **LightGBM没必要对类别型变量使用哑变量化，只要独热编码即可**。关于结论1和2，相见[知乎：用lightgbm建模时，标签编码后的类别型特征还需要归一化吗？还是只需要对连续型特征做归一化呢？](https://www.zhihu.com/question/389542211)。<br>
3. **正在提交数全置0的提交结果更好**。原因可能是由于正在提交数在测试集分布多为0，所以全部置零处可以取得较好的结果，如果再靠训练对ljn(指LAUNCHING_JOB_NUMS)得分可能性价比不是很高。所以当前可以暂时考虑将ljn直接置零，只进行cpu进行针对性的训练和预测。
4. **预测目标为T+5的值，没有使用T+1,T+2，...,T+5作为预测目标好**。原因前者是只通过一个模型，预测当前样本时间点下，未来第25分钟的值，其实时间跨度有些大，模型误差也因此变大。而通过5个模型，分别预测未来5/10/15/20/25分钟下的值，模型效果会更好，简单来说，就是说从近到远的时间点我都有专门的模型去预测，这样显而易见地将准确度提高了，但缺点是模型多了。

由于最新的data_preprocess4.ipynb把当前能考虑的预处理方法都放进去了，所以最后完整运行后，特征数会很多。个人建议从无到有去依次使用不同预处理手段下的数据去训练模型，若模型表现不错则进一步添加新的预处理手段。数据预处理消融实验汇总记录如下：

|编号|预处理手段|线下表现(MSE/测评分数)|线上分数|提交时间|数据版本|
|:--:|:--:|:--:|:--:|:--:|--:|
|1|直接清除数据缺失，不处理重复样本，特征工程4(单5阶)|41.70/2.40|0.29023|2020.11.12|v3|
|2|直接清除数据缺失，不处理重复样本，特征工程4(2-5阶)|42.01/2.41|**0.29074**|2020.11.13|v4|

注：“时间格式转换”，"排序后去除时间特征DOTTING_TIME", "特征工程1"和“类别型数据独热编码”是全部实验都必须使用的。

### 2020.11.12
1. 预处理实验1主要是想研究**特征工程4-行内特征(单5阶)**：

|编号|预处理手段|线下表现(MSE/测评分数)|线上分数|提交时间|数据版本|
|:--:|:--:|:--:|:--:|:--:|--:|
|1|直接清除数据缺失，不处理重复样本，特征工程4(单5阶)|41.70/2.40|0.29023|2020.11.13|v3|

注：这里不进行特征工程2<br>
**总结**：说明行内描述性统计特征很有帮助。

### 2020.11.13
1. 预处理实验2主要是想研究**特征工程4-行内特征(2-5阶)**：

|编号|预处理手段|线下表现(MSE/测评分数)|线上分数|提交时间|数据版本|
|:--:|:--:|:--:|:--:|:--:|--:|
|1|直接清除数据缺失，不处理重复样本，特征工程4(单5阶)|41.70/2.40|0.29023|2020.11.13|v3|
|2|直接清除数据缺失，不处理重复样本，特征工程4(2-5阶)|42.01/2.41|**0.29074**|2020.11.13|v4|

注：这里不进行特征工程2<br>
**总结**：说明行内描述性统计特征-多阶提升不大。

### 2020.11.14
1. 预处理实验2主要是想研究**特征工程3-特征工程4-行内特征(单5阶)**：

|编号|预处理手段|线下表现(MSE/测评分数)|线上分数|提交时间|数据版本|
|:--:|:--:|:--:|:--:|:--:|--:|
|1|直接清除数据缺失，不处理重复样本，特征工程3，特征工程4(单5阶)|41.85/2.40|0.29009|2020.11.14|v|

注：这里不进行特征工程2<br>

### 2020.11.14
1. 预处理实验2主要是想研究**特征工程2(单5阶)-特征工程3-特征工程4-行内特征(单5阶)**：

|编号|预处理手段|线下表现(MSE/测评分数)|线上分数|提交时间|数据版本|
|:--:|:--:|:--:|:--:|:--:|--:|
|1|直接清除数据缺失，不处理重复样本，特征工程2(单5阶)，特征工程3，特征工程4(单5阶)|37.35/2.29|0.14259|2020.11.14|v|

注：这里进行特征工程2<br>
**总结**：引入差分五阶特征后，应该是产生了过拟合现象。
