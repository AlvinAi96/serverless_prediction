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
|1|直接清除数据缺失，不处理重复样本，特征工程3-4(单5阶)|41.70/2.40|0.29023|2020.11.12|v3|
|2|直接清除数据缺失，不处理重复样本，特征工程3-4(2-5阶)|42.01/2.41|**0.29074**|2020.11.13|v4|

注：“时间格式转换”，"排序后去除时间特征DOTTING_TIME", "特征工程1"和“类别型数据独热编码”是全部实验都必须使用的。

### 2020.11.12
1. 预处理实验1主要是想研究**特征工程3-特征工程4-行内特征(单5阶)**：

|编号|预处理手段|线下表现(MSE/测评分数)|线上分数|提交时间|数据版本|
|:--:|:--:|:--:|:--:|:--:|--:|
|1|直接清除数据缺失，不处理重复样本，特征工程3-4(单5阶)|41.70/2.40|0.29023|2020.11.13|v3|

注：这里不进行特征工程2<br>
**总结**：说明行内描述性统计特征很有帮助。

### 2020.11.13
1. 预处理实验2主要是想研究**特征工程4-行内特征(2-5阶)**：

|编号|预处理手段|线下表现(MSE/测评分数)|线上分数|提交时间|数据版本|
|:--:|:--:|:--:|:--:|:--:|--:|
|1|直接清除数据缺失，不处理重复样本，特征工程3-4(单5阶)|41.70/2.40|0.29023|2020.11.13|v3|
|2|直接清除数据缺失，不处理重复样本，特征工程3-4(2-5阶)|42.01/2.41|**0.29074**|2020.11.13|v4|

注：这里不进行特征工程2<br>
**总结**：说明行内描述性统计特征-多阶提升不大。

### 2020.11.14
1. 预处理实验3主要是想研究**特征工程2(单5阶)-特征工程3-特征工程4-行内特征(单5阶)**：

|编号|预处理手段|线下表现(MSE/测评分数)|线上分数|提交时间|数据版本|
|:--:|:--:|:--:|:--:|:--:|--:|
|1|直接清除数据缺失，不处理重复样本，特征工程3-4(单5阶)|41.85/2.40|0.29009|2020.11.14|v|
|3|直接清除数据缺失，不处理重复样本，特征工程2(单5阶)，特征工程3-4(单5阶)|37.35/2.29|0.14259|2020.11.14|v|

注：这里进行特征工程2<br>
**总结**：引入差分五阶特征后，应该是产生了过拟合现象。

### 2020.11.15
1. 听完[华为官方赛题讲解](华为云-大数据时代的Serverless工作负载预测)，有几点需要关注下：
- “云环境下采集数据存在噪声，虽然发生比较少，但噪声可能比较大”。这种噪声很难衡量，后续可以考虑训练时引入额外噪声增加模型通用性看看。
- “CU绝大部分是静态信息，但不排斥云服务提供方对用户队列进行扩容”。这点跟EDA的结果是一样的。
- “B榜的数据被包含于现有训练集中”。这样我们后续切B榜，以队列-level的训练方式是可以应对的。
- “采集异常导致DOTTING_TIME一样，其他不一样”。
- “采集的状态是指一个周期下相同状态的数量之后，任务数是根据前后时间戳内的变化量统计得到的”。所以时间间隔特征重要，它反映了数据是在多长时间下统计得到的。
2. 听完鱼佬分享直播，有几点需要关注下：
- “CPU使用率提高，内存使用率下降，磁盘使用率上升”。这个跟我们EDA相关性热图分析结果是一致的，我认为的业务解释是CPU使用率变高，内存会溢出，所以内存使用率下降了，然后原内存溢出的部分内容被释放（移除或存入）磁盘，因此最后磁盘使用率上升。
- “时间序列特征常见有四点特征需要注意：周期性、趋势性、强相关性及异常点。从周期性考虑，像日期变量可以拆分为年/月/周/日/小时/分钟，像距离某天的时间差。从异常点考虑，是否某个特殊日期，时间组合。但该赛题关注趋势性和强相关，要结合时序相关特征（历史平移，划窗特征）”。这块跟我们前期的思路是一致的。
- “划窗特征统计时使用了mean,std,max,min,median,np.ptp。可以考虑些交叉统计”。交叉统计特征？这块我们或许可以深挖下。
- “模型融合分三块考虑：<br>
理论分析：特征差异 + 样本差异 + 模型差异。样本差异可以行/列采样。更细的分为5个：样本扰动，不同特征组，输出转换，参数调整，loss函数选择。<br>
训练过程融合：Bagging + Boosting<br>
训练结果融合：投票法 + 平均法 + Stacking”。
- Datawhale开源代码中有对xgb,lgb和cat模型有训练代码，具有参考价值。

3. 实验4主要是想研究**保留含缺失值的特征**：

|编号|预处理手段|线下表现(MSE/测评分数)|线上分数|提交时间|数据版本|
|:--:|:--:|:--:|:--:|:--:|--:|
|2|直接清除数据缺失，不处理重复样本，特征工程3-4(2-5阶)|42.01/2.41|0.29074|2020.11.13|v4|
|4|不处理数据缺失，不处理重复样本，特征工程3-4(2-5阶)|41.99/2.41|**0.29279**|2020.11.13|v5|

注：实验4的数据缺失处理：直接删除'RESOURCE_TYPE'，用0填补'DISK_USAGE'的缺失值。<br>
**总结**：做实验4的目的在于：之前eda.ipynb发现了'DISK_USAGE'与CPU使用率的相关性，而之前实验卻因它存在缺失值而直接删除它，这有些得不偿失。由于eda.ipynb得到含缺失值特征的队列（[297, 298, 20889, 21487, 21671, 21673, 81221, 82695, 82697, 82929, 83109, 83609]），为什么训练时我们不让：**无缺失值队列模型使用全部特征进行训练，而含缺失值队列模型不使用缺失值特征训练**呢？实验4实验的该想法后证实实验获得不少提升，说明对DISK_USAGE的缺失值处理能有效提高模型表现，后续建议考虑合理的缺失值填补方式去将所有队列模型都能加入DISK_USAGE进入训练中。有预感这会是个涨分点。
