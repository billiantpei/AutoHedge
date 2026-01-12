1、背景
项目背景：
目前在信贷业务上存在多个场景，比如现金贷，BNPL，信用卡等。每个业务场景下存在着不少的文本等相关信息，比如征信报告中地理位置，BNPL中的SKU等，同时依赖滴滴生态，我们也可以使用滴滴出行和外卖等地理位置数据。针对这些文本信息，目前可以开发的更多是相关统计维度的特征，比如同一geohash网格下的次数或者频率相关维度特征，但是无法捕抓更细粒度下的信息，比如机场，居民小区，学校，医院等信息。因此调研了业界的一些常见做法，同时结合已有的资源情况，开发基于预训练好的西语bert模型，并且走通了在整体的开发部署流程

项目收益：
1、减少特征依赖，提升特征表达能力
   模型不再依赖单独开发的文本或统计维度特征，而是通过大模型直接对文本信息进行语义理解与编码，实现更细粒度的特征利用。例如，可以自动识别与区分“学校”“商场”“餐饮”等场景类型，从而更准确地捕捉用户行为特征与潜在消费偏好。 
2、利用预训练语言模型与多模态融合，显著提升模型效果
   引入预训练的西语语言模型，能够更充分地理解文本语义信息，补充传统结构化数据中缺失的语义特征维度。同时，结合时序特征与多模态信息（如文本、时间序列、地理位置等），实现多源数据的联合建模，进一步提升模型的稳定性与预测精度。 
3、多维度预测能力，拓展模型业务价值
  当前模型可同时预测风险、年龄与收入等多维度标签。相比依赖征信数据的传统风险模型，本项目仅使用出行、外卖等行为数据即可覆盖全部用户，实现风险与资质的统一评分。这一能力不仅支持对出行与外卖用户群的全面打分，也为预授信-精准营销等业务场景提供了高价值支持。 


应用场景:
信贷业务线上涉及文本等信息的全部场景，比如授信反欺诈、ABC卡、营销推广等。

2、业界调研
文本模型演变史

文本模型选型维度，为什么选择bert模型？
1、维度1：模型效果。
   保持相同文本内容，对纯文本信息建模效果对比：
      

   结论：预训练的西语模型比纯文本模型，在更多大语料的训练下，保留了更多语义信息，更能带来效果上的增益。

2、维度2：模型部署难度

   背景：目前GPU资源有限，没法支持模型回溯。需要对几千万用户进行离线打分，并且回溯相关分区数据。因此离线部署方案调整为spark上利用cpu打分。

试验1：实战环境：生产1卡(L20)  -- (1GPU/45G/12CPU/64G)
                LAMM3-14B模型，微调时候OOM。

试验2：spark上 3000w用户 模型参数量140M， 打分耗时极高，超过24h，任务被kill
     
因此模型方案调整为：使用基础西语bert-base模型，然后对bert蒸馏，在接入下游模型结构。
    
Bert模型选型：
bert模型结构：
![alt text](image-1.png)

BERT整体框架包含pre-train和fine-tune两个阶段。pre-train阶段模型是在无标注的标签数据上进行训练，fine-tune阶段，BERT模型首先是被pre-train模型参数初始化，然后所有的参数会用下游的有标注的数据进行训练。

Transformer结构
![alt text](image-2.png)

BERT是用了Transformer的encoder侧的网络，如上图的transformer的Encoder部分

西语bert 选择： （参数不宜过大，同时下载量较高）
![alt text](image-3.png)
https://huggingface.co/dccuchile/bert-base-spanish-wwm-cased
训练西语语料信息：
![alt text](image.png)
Number of lines: 300904000 (300M)
Number of tokens: 2996016962 (3B)
Number of chars: 18431160978 (18.4B)

bert-base-spanish-wwm-cased 结构
![alt text]({537A7646-D2CA-41B8-9BD6-087093C32E0C}.png)
蒸馏方案选型：

1、蒸馏方案调研：
![alt text]({B60CC197-6B71-4F6E-9F83-F397F588BF00}.png)

最后选择了TinyBERT 的蒸馏方案：效果与相关文档上，Tinybert更多，而且效果显著。

“实验结果表明，TinyBERT在只有原始 BERT 模型 13%参数量的情况下，推断加速达到 9 倍，同时在自然语言处理标准评测 GLUE 上获得原始 BERT 模型 96%的效果；另一方面，对比当前其他主流蒸馏方法，TinyBERT 在只有 28%的参数和 31%的推断时间情况下，平均提升 3 个点以上”

Tinybert结构图：https://www.infoq.cn/article/egwerqdg6l5zuaexash3
![alt text](image-4.png)


![alt text](image-5.png)

2、蒸馏试验结果
![alt text]({A684C9BB-A043-407D-860D-6112CEDC28F1}.png)
  参数对比
参数量占比：14500634/109852418 = 13.2%

6.8%/ 7.3% = 93.2% 模型性能损失很小，保留了原始模型93%的效果

3、模型设计
建模标签
      1、标签分析：
  年龄与 风险& 收入关系分析：
[图片: https://cooper.didichuxing.com/uploader/f/gZBfEjh6lszqD2Vp.png?accessToken=eyJhbGciOiJIUzI1NiIsImtpZCI6ImRlZmF1bHQiLCJ0eXAiOiJKV1QifQ.eyJleHAiOjE3NjgyMTgxMzIsImZpbGVHVUlEIjoiTVFWOTJlZ3JkVmdWRzF5eiIsImlhdCI6MTc2ODIxNzUzMiwiaXNzIjoidXBsb2FkZXJfYWNjZXNzX3Jlc291cmNlIiwicGFhIjoic2RrOnRoaXJkOmUwNjhhMDcxZTQ3MzQyMTI5NmE4ZDRmODE2MWY4MjU5IiwidXNlcklkIjoxMDAwMDM0OTY1OX0.EOwmqVInd5bAFfMsqxL0ua5CF9umCWQwUlNtnDtHgTs](https://cooper.didichuxing.com/uploader/f/gZBfEjh6lszqD2Vp.png?accessToken=eyJhbGciOiJIUzI1NiIsImtpZCI6ImRlZmF1bHQiLCJ0eXAiOiJKV1QifQ.eyJleHAiOjE3NjgyMTg3NTksImZpbGVHVUlEIjoiTVFWOTJlZ3JkVmdWRzF5eiIsImlhdCI6MTc2ODIxODE1OSwiaXNzIjoidXBsb2FkZXJfYWNjZXNzX3Jlc291cmNlIiwicGFhIjoic2RrOnRoaXJkOmUwNjhhMDcxZTQ3MzQyMTI5NmE4ZDRmODE2MWY4MjU5IiwidXNlcklkIjoxMDAwMDM0OTY1OX0.KEvGLYsQLa8Rlo4ZMWE2UqhwvtP5pg89IoHwkodTp7s)

eva收入与风险关系分析：
[图片: https://cooper.didichuxing.com/uploader/f/sDZWp7EKeR6q3qqA.png?accessToken=eyJhbGciOiJIUzI1NiIsImtpZCI6ImRlZmF1bHQiLCJ0eXAiOiJKV1QifQ.eyJleHAiOjE3NjgyMTgxMzIsImZpbGVHVUlEIjoiTVFWOTJlZ3JkVmdWRzF5eiIsImlhdCI6MTc2ODIxNzUzMiwiaXNzIjoidXBsb2FkZXJfYWNjZXNzX3Jlc291cmNlIiwicGFhIjoic2RrOnRoaXJkOmUwNjhhMDcxZTQ3MzQyMTI5NmE4ZDRmODE2MWY4MjU5IiwidXNlcklkIjoxMDAwMDM0OTY1OX0.EOwmqVInd5bAFfMsqxL0ua5CF9umCWQwUlNtnDtHgTs](https://cooper.didichuxing.com/uploader/f/sDZWp7EKeR6q3qqA.png?accessToken=eyJhbGciOiJIUzI1NiIsImtpZCI6ImRlZmF1bHQiLCJ0eXAiOiJKV1QifQ.eyJleHAiOjE3NjgyMTg3NTksImZpbGVHVUlEIjoiTVFWOTJlZ3JkVmdWRzF5eiIsImlhdCI6MTc2ODIxODE1OSwiaXNzIjoidXBsb2FkZXJfYWNjZXNzX3Jlc291cmNlIiwicGFhIjoic2RrOnRoaXJkOmUwNjhhMDcxZTQ3MzQyMTI5NmE4ZDRmODE2MWY4MjU5IiwidXNlcklkIjoxMDAwMDM0OTY1OX0.KEvGLYsQLa8Rlo4ZMWE2UqhwvtP5pg89IoHwkodTp7s)
     
结论：最终考虑到资质与风险是对于业务来讲比较重要的两个方向，同时 资质越好的用户风险是越低的，风险 与资质(年龄与收入)是有相关性的，因此考虑使用了 多任务建模，同时预估 风险，年龄与收入

年龄标签：0,1,2,3 分别对应不同的年龄区间
风险标签：0,1 对应是否会逾期
收入标签: 对高收入截断 大于等于3w 就直接等于3w，同时只选取收入大于7000以上的用户，然后对收入进行对数变换, 然后归一化 在 -1到1 之间
[图片: https://cooper.didichuxing.com/uploader/f/YQ3YYfHyOV8TSxJ6.png?accessToken=eyJhbGciOiJIUzI1NiIsImtpZCI6ImRlZmF1bHQiLCJ0eXAiOiJKV1QifQ.eyJleHAiOjE3NjgyMTgxMzIsImZpbGVHVUlEIjoiTVFWOTJlZ3JkVmdWRzF5eiIsImlhdCI6MTc2ODIxNzUzMiwiaXNzIjoidXBsb2FkZXJfYWNjZXNzX3Jlc291cmNlIiwicGFhIjoic2RrOnRoaXJkOmUwNjhhMDcxZTQ3MzQyMTI5NmE4ZDRmODE2MWY4MjU5IiwidXNlcklkIjoxMDAwMDM0OTY1OX0.EOwmqVInd5bAFfMsqxL0ua5CF9umCWQwUlNtnDtHgTs](https://cooper.didichuxing.com/uploader/f/YQ3YYfHyOV8TSxJ6.png?accessToken=eyJhbGciOiJIUzI1NiIsImtpZCI6ImRlZmF1bHQiLCJ0eXAiOiJKV1QifQ.eyJleHAiOjE3NjgyMTg3NTksImZpbGVHVUlEIjoiTVFWOTJlZ3JkVmdWRzF5eiIsImlhdCI6MTc2ODIxODE1OSwiaXNzIjoidXBsb2FkZXJfYWNjZXNzX3Jlc291cmNlIiwicGFhIjoic2RrOnRoaXJkOmUwNjhhMDcxZTQ3MzQyMTI5NmE4ZDRmODE2MWY4MjU5IiwidXNlcklkIjoxMDAwMDM0OTY1OX0.KEvGLYsQLa8Rlo4ZMWE2UqhwvtP5pg89IoHwkodTp7s)
模型特征筛选&处理方式
 特征筛选主要包含两部分，第一部分是出行外卖的数据基础信息数据，另一部分是基于统计维度计算的特征, 

 1、特征稳定性 PSI 筛选：
[图片: https://cooper.didichuxing.com/uploader/f/QOjq2ReTxMRJ718i.png?accessToken=eyJhbGciOiJIUzI1NiIsImtpZCI6ImRlZmF1bHQiLCJ0eXAiOiJKV1QifQ.eyJleHAiOjE3NjgyMTgxMzIsImZpbGVHVUlEIjoiTVFWOTJlZ3JkVmdWRzF5eiIsImlhdCI6MTc2ODIxNzUzMiwiaXNzIjoidXBsb2FkZXJfYWNjZXNzX3Jlc291cmNlIiwicGFhIjoic2RrOnRoaXJkOmUwNjhhMDcxZTQ3MzQyMTI5NmE4ZDRmODE2MWY4MjU5IiwidXNlcklkIjoxMDAwMDM0OTY1OX0.EOwmqVInd5bAFfMsqxL0ua5CF9umCWQwUlNtnDtHgTs](https://cooper.didichuxing.com/uploader/f/QOjq2ReTxMRJ718i.png?accessToken=eyJhbGciOiJIUzI1NiIsImtpZCI6ImRlZmF1bHQiLCJ0eXAiOiJKV1QifQ.eyJleHAiOjE3NjgyMTg3NTksImZpbGVHVUlEIjoiTVFWOTJlZ3JkVmdWRzF5eiIsImlhdCI6MTc2ODIxODE1OSwiaXNzIjoidXBsb2FkZXJfYWNjZXNzX3Jlc291cmNlIiwicGFhIjoic2RrOnRoaXJkOmUwNjhhMDcxZTQ3MzQyMTI5NmE4ZDRmODE2MWY4MjU5IiwidXNlcklkIjoxMDAwMDM0OTY1OX0.KEvGLYsQLa8Rlo4ZMWE2UqhwvtP5pg89IoHwkodTp7s)

2、特征处理
     1） 空值填充 （均值填充）
     2）截断 （10%-90%之间）
     3)  离散特征 OrdinalEncoder 转换，后接embedding
     4）文本特征：保留最近20条 出行与外卖数据，超过截断与不足padding

模型结构
[图片: https://cooper.didichuxing.com/uploader/f/2LPFrWpc0t4UJ4Cr.png?accessToken=eyJhbGciOiJIUzI1NiIsImtpZCI6ImRlZmF1bHQiLCJ0eXAiOiJKV1QifQ.eyJleHAiOjE3NjgyMTgxMzIsImZpbGVHVUlEIjoiTVFWOTJlZ3JkVmdWRzF5eiIsImlhdCI6MTc2ODIxNzUzMiwiaXNzIjoidXBsb2FkZXJfYWNjZXNzX3Jlc291cmNlIiwicGFhIjoic2RrOnRoaXJkOmUwNjhhMDcxZTQ3MzQyMTI5NmE4ZDRmODE2MWY4MjU5IiwidXNlcklkIjoxMDAwMDM0OTY1OX0.EOwmqVInd5bAFfMsqxL0ua5CF9umCWQwUlNtnDtHgTs](https://cooper.didichuxing.com/uploader/f/2LPFrWpc0t4UJ4Cr.png?accessToken=eyJhbGciOiJIUzI1NiIsImtpZCI6ImRlZmF1bHQiLCJ0eXAiOiJKV1QifQ.eyJleHAiOjE3NjgyMTg3NTksImZpbGVHVUlEIjoiTVFWOTJlZ3JkVmdWRzF5eiIsImlhdCI6MTc2ODIxODE1OSwiaXNzIjoidXBsb2FkZXJfYWNjZXNzX3Jlc291cmNlIiwicGFhIjoic2RrOnRoaXJkOmUwNjhhMDcxZTQ3MzQyMTI5NmE4ZDRmODE2MWY4MjU5IiwidXNlcklkIjoxMDAwMDM0OTY1OX0.KEvGLYsQLa8Rlo4ZMWE2UqhwvtP5pg89IoHwkodTp7s)

模型效果

风险上效果：全部主端测试案上样本评估
![alt text]({15114687-8259-4E99-862D-8EED99CFEFA9}.png)

年龄上增益：（匹配的样本上全部测试案评估）
图片: https://cooper.didichuxing.com/uploader/f/qMhdDsFEivcUERW7.png?accessToken=eyJhbGciOiJIUzI1NiIsImtpZCI6ImRlZmF1bHQiLCJ0eXAiOiJKV1QifQ.eyJleHAiOjE3NjgyMTgxMzIsImZpbGVHVUlEIjoiTVFWOTJlZ3JkVmdWRzF5eiIsImlhdCI6MTc2ODIxNzUzMiwiaXNzIjoidXBsb2FkZXJfYWNjZXNzX3Jlc291cmNlIiwicGFhIjoic2RrOnRoaXJkOmUwNjhhMDcxZTQ3MzQyMTI5NmE4ZDRmODE2MWY4MjU5IiwidXNlcklkIjoxMDAwMDM0OTY1OX0.EOwmqVInd5bAFfMsqxL0ua5CF9umCWQwUlNtnDtHgTs

收入上增益：（匹配样本上全部测试案评估）
真实eva收入（只看有eve收入的客群，包括在职与失业）
图片: https://cooper.didichuxing.com/uploader/f/4xO3YewYKpQlM3Vm.png?accessToken=eyJhbGciOiJIUzI1NiIsImtpZCI6ImRlZmF1bHQiLCJ0eXAiOiJKV1QifQ.eyJleHAiOjE3NjgyMTgxMzIsImZpbGVHVUlEIjoiTVFWOTJlZ3JkVmdWRzF5eiIsImlhdCI6MTc2ODIxNzUzMiwiaXNzIjoidXBsb2FkZXJfYWNjZXNzX3Jlc291cmNlIiwicGFhIjoic2RrOnRoaXJkOmUwNjhhMDcxZTQ3MzQyMTI5NmE4ZDRmODE2MWY4MjU5IiwidXNlcklkIjoxMDAwMDM0OTY1OX0.EOwmqVInd5bAFfMsqxL0ua5CF9umCWQwUlNtnDtHgTs

多任务模型-收入分（匹配的样本上全部测试案评估） 优化也在这部分样本上 对比 eva收入情况 
图片: https://cooper.didichuxing.com/uploader/f/iBhJRiM73i5lgKHn.png?accessToken=eyJhbGciOiJIUzI1NiIsImtpZCI6ImRlZmF1bHQiLCJ0eXAiOiJKV1QifQ.eyJleHAiOjE3NjgyMTgxMzIsImZpbGVHVUlEIjoiTVFWOTJlZ3JkVmdWRzF5eiIsImlhdCI6MTc2ODIxNzUzMiwiaXNzIjoidXBsb2FkZXJfYWNjZXNzX3Jlc291cmNlIiwicGFhIjoic2RrOnRoaXJkOmUwNjhhMDcxZTQ3MzQyMTI5NmE4ZDRmODE2MWY4MjU5IiwidXNlcklkIjoxMDAwMDM0OTY1OX0.EOwmqVInd5bAFfMsqxL0ua5CF9umCWQwUlNtnDtHgTs

![alt text]({15ED7A62-233A-4FFE-BBD0-E47D8BA08969}.png)
4、模型部署

cpu计算下的加速方案调研
![alt text]({7FEE2B67-1B59-4EAF-858A-5907D8B6A4E4}.png)
网上测试结果-- 非实际测试
ONNX 与 JIT加速方案对比文档：
https://zhuanlan.zhihu.com/p/349770691

“ONNX Runtime 在 CPU 上更快的本质，是因为它是一个“静态图编译 + 高度算子融合 + 针对 CPU 内核优化”的 推理引擎；
而 PyTorch JIT 本质仍是一个“动态解释器 + 局部优化器”，并非专为推理部署而设计
”
最终选择了基于onnx的加速方案。同时使用 使用 int 8 的精度优化
![alt text]({9CC41106-9889-4465-A955-C85C4147BD1A}.png)
onnx 加速后效果对比：
(优化前) ：数量： 7800  batch_size：32， 1 cpu，耗时41分钟，单case耗时：0.315s
(优化后)： 数量： 22242 batch_size：32， 5 cpu, 耗时16分钟，单case耗时：0.215s 
提升运行速度 1.5倍


spark 一计算方案优化
![alt text]({B7466B2B-C165-4CA0-BAD5-D4F0872C1A1D}.png)
结论：目前通过上面的一些优化技术，整体千万的用户量，在200分钟左右完成。


5、总结-创新点
 整体架构创新：
BERT 表征能力 + 蒸馏压缩 + 多任务结构融合（PLE）+ 训练与推理一体化优化

一、创新点 1：BERT 与结构化特征融合的语义增强
	•	引入 BERT 模型 对高语义复杂度特征（如出行地址、外卖地址等）进行嵌入式表征；
	•	将文本化嵌入与数值类统计特征拼接，通过统一 Transformer 层融合，显著提升跨模态特征的表达能力；
	•	对比传统 树 模型（XGBoost），语义特征的利用率和模型的区分度均明显提高。


二、创新点 2：蒸馏 BERT（TinyBERT）实现轻量化部署
	•	使用 TinyBERT 蒸馏策略，通过 隐藏层匹配（Hidden-State Loss） 与 注意力矩阵匹配（Attention Loss） 同步优化，使学生模型（Student）在保持 90% 表现的同时，参数量仅为教师模型（Teacher）的 1/6；
	•	支持实时推理（online scoring），有效控制资源消耗（GPU/CPU）；
	•	离线训练采用 分阶段蒸馏（layer-wise distillation） 与 任务蒸馏（task-specific distillation），确保特征层与任务层均能保留语义映射。

三、创新点 3：融合 PLE (Progressive Layered Extraction) 的多任务学习框架
	•	在蒸馏后的 BERT 表征上构建 PLE 多任务结构，共享底层通用特征，分层提取任务特定特征；
	•	多任务设置覆盖：
	•	年龄预测 
	•	违约风险识别
	•	收入预测 
	•	通过 PLE 的专家门控结构（Shared Expert + Task-specific Expert）实现任务间信息解耦，减轻负迁移问题。

四、创新点 4：训练与推理一体化优化
	•	支持 Spark + ONNXRuntime 部署；
	•	蒸馏后模型可在分布式环境中以批推理或实时方式执行；
	•	启用图优化（ORT_ENABLE_ALL）与 mapInPandas 并行推理，提升推理效率。


6、未来规划