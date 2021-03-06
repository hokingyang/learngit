\#读书《Python自然语言处理实战》，作者涂铭, 刘祥, 刘树春。

从自然语言的角度出发，NLP基本可以分为两个部分：自然语言处理以及自然语言生成，演化为语义理解和自然语言生成文本。其中自然语言生成包括三个阶段：文本规划（完成结构化数据中的基础内容规划）、语句规划（从结构化数据中组合语句来表达信息流）、实现（产生语法通顺的语句来表达文本）。

NLP是一门杂学，其构成知识基础包罗万象，这里简单罗列其知识体系：

- 句法语义分析：针对目标句子，进行各种句法分析，如分词、词性标记、命名实体识别及链接、句法分析、语义角色识别和多义词消歧等。
- 关键词抽取：抽取目标文本中的主要信息，比如从一条新闻中抽取关键信息。主要是了解是谁、于何时、为何、对谁、做了何事、产生了有什么结果。涉及实体识别、时间抽取、因果关系抽取等多项关键技术。 
- 文本挖掘：主要包含了对文本的聚类、分类、信息抽取、摘要、情感分析以及对挖掘的信息和知识的可视化、交互式的呈现界面。
- 机器翻译：将输入的源语言文本通过自动翻译转化为另一种语言的文本。根据输入数据类型的不同，可细分为文本翻译、语音翻译、手语翻译、图形翻译等。机器翻译从最早的基于规则到二十年前的基于统计的方法，再到今天的基于深度学习（编解码）的方法，逐渐形成了一套比较严谨的方法体系。
- 信息检索：对大规模的文档进行索引。可简单对文档中的词汇，赋以不同的权重来建立索引，也可使用算法模型来建立更加深层的索引。查询时，首先对输入比进行分析，然后在索引里面查找匹配的候选文档，再根据一个排序机制把候选文档排序，最后输出排序得分最高的文档。
- 问答系统：针对某个自然语言表达的问题，由问答系统给出一个精准的答案。需要对自然语言查询语句进行语义分析，包括实体链接、关系识别，形成逻辑表达式，然后到知识库中查找可能的候选答案并通过一个排序机制找出最佳的答案。
- 对话系统：系统通过多回合对话，跟用户进行聊天、回答、完成某项任务。主要涉及用户意图理解、通用聊天引擎、问答引擎、对话管理等技术。此外，为了体现上下文相关，要具备多轮对话能力。同时，为了体现个性化，对话系统还需要基于用户画像做个性化

NLP可以被应用于很多领域，这里大概总结出以下几种通用的应用：

- 机器翻译：计算机具备将一种语言翻译成另一种语言的能力。
- 情感分析：计算机能够判断用户评论是否积极。
- 智能问答：计算机能够正确回答输入的问题。
- 文摘生成：计算机能够准确归纳、总结并产生文本摘要。
- 文本分类：计算机能够采集各种文章，进行主题分析，从而进行自动分类。
- 舆论分析：计算机能够判断目前舆论的导向。
- 知识图谱：知识点相互连接而成的语义网络。

本书基于Python，分别介绍了正则表达式、Numpy应用、中文分词（着重于jieba）、词性标注、关键词提取、句法分析、文章相似度分析、情感分析、机器学习建模、基于深度学习的NLP算法等十章内容，对NLP从入门到高手成长过程大有裨益。书中内容都配有github上的代码可供使用，但是因为这一领域这两年发展太快，后面机器学习部分都是针对TF1的编码，用在最新的TF2上就不能正常通过，需要花费一定的时间进行移植。如果不细抠代码实现，这本书对了解NLP的内容和如何学习还是很有全局观的。