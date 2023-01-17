---
id: 781
title: 利用python进行自然语言处理（spaCy模块的简单使用）
date: '2019-10-11T23:51:49+08:00'
author: Janvn
layout: post
guid: 'http://codercoder.cn/?p=781'
permalink: /index.php/2019/10/nlpinpython/
views:
    - '855'
bigfa_ding:
    - '1'
categories:
    - Tech
tags:
    - nlp
    - python
    - spaCy
---

# 前言：

自然语言处理在数据科学领域中是一个大问题。在这篇文章中，我们来看一下对那些利用Python进行自然语言处理(NLP)的人可以使用的库。

# 正文：

自然语言处理（NLP）是数据科学中最有趣的子领域之一，数据科学家越来越期望能够制定涉及利用非结构化文本数据的解决方案。尽管如此，许多应用数据科学家（来自STEM和社会科学背景）都缺乏NLP经验。  
在这篇文章中，我将探讨一些基本的NLP概念，并展示如何使用Python中日益流行的spaCy包来实现它们。这篇文章是针对绝对的NLP初学者，但是假设其具备Python的知识。

## spaCy, You Say?

spaCy是由Matt Honnibal在Explosion AI开发用于“Python中的工业强度NLP”的相对较新的软件包。它的设计考虑了应用数据科学家，这意味着它不会影响用户决定使用什么深奥的算法来执行常见任务，而且速度快 – 速度极快(因为它在Cython中实现)。如果您熟悉Python数据科学堆栈，那么spaCy相当于用于NLP任务中的numpy – 尽管它相当低级但非常直观且高效。

## So, What Can It Do?

```
spaCy为任何NLP项目中常用的任务提供一站式服务，包括：
    Tokenization（分词）
    Lemmatization（词形还原）
    Part-of-speech tagging（词性标注）
    Entity recognition（实体识别）
    Dependency parsing（依赖解析）
    Sentence recognition（句子识别）
    Word-to-vector transformations（词向量转换）
    Many convenient methods for cleaning and normalizing text（许多方便的方法来清理和规范化文本）

```

我将提供其中一些功能的高级概述，并展示如何使用spaCy访问它们。

## Let’s Get Started!

首先，我们加载spaCy的管道，按照惯例，它存储在一个名为nlp的变量中。声明此变量将需要几秒钟，因为spaCy需将其模型和数据提前加载以节省之后处理的时间。实际上，这会使得前期处理方式变得非常繁重以致于之后每次将nlp解析器应用到数据时都不会产生成本。请注意，在这里，我使用的是英语模型，但也有一个功能齐全的德语模型，在几种语言中实现分词（如下所述）。  
我们在示例文本上调用NLP来创建Doc对象。 Doc对象现在是用于NLP任务文本本身的容器，文本的切片（Span对象）和元素（Token对象）。值得注意的是，Token和Span对象实际上不包含任何数据。取而代之，它们包含指向Doc对象中包含的数据的指针，并且被懒惰地评估（即根据请求）。 spaCy的大部分核心功能是通过Doc（n = 33），Span（n = 29）和Token（n = 78）对象上的方法访问的。

Spacy安装：

```shell
    pip install spacy

```

下载数据模型：

```shell
    python -m spacy download en_core_web_sm

```

```python
In[1]: import spacy
...: nlp = spacy.load("en_core_web_sm")
...: doc = nlp("The big grey dog ate all of the chocolate,
                but fortunately he wasn't sick!")

```

## Tokenization

分词是许多NLP任务的基础步骤。对文本分词是将一段文本拆分为单词，符号，标点符号，空格和其他元素的过程，从而创建标记。一种天真的方法是简单地以空格拆分字符串：

```python
In[2]: doc.text.split()
 ...:
Out[2]:
['The',
 'big',
 'grey',
 'dog',
 'ate',
 'all',
 'of',
 'the',
 'chocolate,',
 'but',
 'fortunately',
 'he',
 "wasn't",
 'sick!']

```

从表面上看，这看起来很好。但请注意，它忽略了标点符号，并且不会分割动词和副词（“是”，“不是”）。换句话说，它是天真的，它无法帮助我们（和机器）识别文本元素去理解其结构和意义。让我们看看spaCy如何处理这个问题：

```python
In[3]: [token.orth_ for token in doc]
 ...:
Out[3]:
['The',
 'big',
 'grey',
 'dog',
 'ate',
 'all',
 'of',
 'the',
 'chocolate',
 ',',
 'but',
 'fortunately',
 'he',
 'was',
 "n't",
 'sick',
 '!']

```

在这里，我们访问每个词的.orth\_方法，该方法返回词的字符串表示，而不是SpaCy词对象。这可能并不总是可取的，但值得注意。 SpaCy识别标点符号，并能够从单词标记中分割这些标点符号。许多SpaCy的词方法都提供了已处理文本的字符串和整数表示：带有下划线后缀的方法返回字符串和没有下划线后缀的方法返回整数。例如：

```python
In[4]: [(token, token.orth_, token.orth) for token in doc]
 ...:
Out[4]: [(The, 'The', 5059648917813135842),
 (big, 'big', 15511632813958231649),
 (grey, 'grey', 10475807793332549289),
 (dog, 'dog', 7562983679033046312),
 (ate, 'ate', 10806788082624814911),
 (all, 'all', 13409319323822384369),
 (of, 'of', 886050111519832510),
 (the, 'the', 7425985699627899538),
 (chocolate, 'chocolate', 10946593968795032542),
 (,, ',', 2593208677638477497),
 (but, 'but', 14560795576765492085),
 (fortunately, 'fortunately', 13851269277375979931),
 (he, 'he', 1655312771067108281),
 (was, 'was', 9921686513378912864),
 (n't, "n't", 2043519015752540944),
 (sick, 'sick', 14841597609857081305),
 (!, '!', 17494803046312582752)]

```

去除所有标点符号和空格

```python
In[5]: [token.orth_ for token in doc if not token.is_punct | token.is_space]
 ...:
Out[5]:
['The',
 'big',
 'grey',
 'dog',
 'ate',
 'all',
 'of',
 'the',
 'chocolate',
 'but',
 'fortunately',
 'he',
 'was',
 "n't",
 'sick']

```

很酷，对吗？

## Lemmatization

分词的相关任务是词形还原。词形还原是将单词缩减为基本形式的过程 – 如果你愿意的话，它的母语单词。单词的不同用法通常具有相同的根含义。例如，practice，practiced和practising基本上都是指同一件事。通常希望标准化与其基本形式具有相似含义的单词。使用SpaCy，我们可以使用词的.lemma\_方法访问每个单词的基本形式：

```python
In[6]: practice = "practice practiced practicing"
 ...:  nlp_practice = nlp(practice)
 ...: [word.lemma_ for word in nlp_practice]
 ...:
Out[6]:
['practice', 'practice', 'practice']

```

为什么这有用？一个直接的用例是机器学习，特别是文本分类。例如，在创建“词袋”之前对文本进行词形还原可以避免单词重复，因此，允许模型在多个文档中构建更清晰的单词使用模式图像。

## POS Tagging

词性标注是将语法属性（即名词，动词，副词，形容词等）分配给单词的过程。共享相同POS标签的单词倾向于遵循类似的句法结构，并且在基于规则的处理中很有用。  
例如，在事件的给定描述中，我们可能希望确定谁拥有什么。通过利用所有格，我们可以做到这一点（提供文本在语法上是合理的！）。SpaCy使用流行的Penn Treebank POS标签（见这里，https://www.ling.upenn.edu/courses/Fall\_2003/ling001/penn\_treebank\_pos.html  
）。使用SpaCy，您可以分别使用.pos\_和.tag\_方法访问粗粒度和细粒度POS标签。在这里，我访问细粒度的POS标签：

```python
In[7]: doc2 = nlp("Conor's dog's toy was hidden under the man's sofa in the woman's house")
...: pos_tags = [(i, i.tag_) for i in doc2]
...: pos_tags
...:
Out[7]:
[(Conor, 'NNP'),
('s, 'POS'),
(dog, 'NN'),
('s, 'POS'),
(toy, 'NN'),
(was, 'VBD'),
(hidden, 'VBN'),
(under, 'IN'),
(the, 'DT'),
(man, 'NN'),
('s, 'POS'),
(sofa, 'NN'),
(in, 'IN'),
(the, 'DT'),
(woman, 'NN'),
('s, 'POS'),
(house, 'NN')]

```

我们可以看到’s这些词被标记为POS。我们可以利用此标记来提取所有者及其拥有的东西：

```python
In[8]: owners_possessions = []
...: for i in pos_tags:
...:     if i[1] == "POS":
...:         owner = i[0].nbor(-1)
...:         possession = i[0].nbor(1)
...:         owners_possessions.append((owner, possession))
...:
...: owners_possessions
...:
Out[8]:
[(Conor, dog), (dog, toy), (man, sofa), (woman, house)]

```

这将返回所有者拥有元组的列表。如果你想成为关于它的超级Pythonic，你可以在列表理解（使用python的列表推导式）中做到这一点（我认为这是更好的！）：

```python
In[9]: [(i[0].nbor(-1), i[0].nbor(+1)) for i in pos_tags if i[1] == "POS"]
...:
Out[9]:
[(Conor, dog), (dog, toy), (man, sofa), (woman, house)]

```

在这里，我们使用每个词的.nbor方法，该方法返回词的相邻的词。

## Entity Recognition

实体识别是将文本中找到的命名实体分类为预定义类别（如人员，地点，组织，日期等）的过程.spaCy使用统计模型对广泛的实体进行分类，包括人员，事件，艺术作品和国籍/宗教（参见完整清单的文件, https://spacy.io/usage/entity-recognition%29.）。  
例如，让我们从巴拉克奥巴马的维基百科条目中获取前两句话。我们将解析此文本，然后使用Doc对象的.ents方法访问已识别的实体。通过在Doc上调用此方法，我们可以访问其他词方法，特别是.label\_和.label：

```python
In[10]: wiki_obama = """Barack Obama is an American politician who served as
...: the 44th President of the United States from 2009 to 2017. He is the first
...: African American to have served as president,
...: as well as the first born outside the contiguous United States."""
...:
...: nlp_obama = nlp(wiki_obama)
...: [(i, i.label_, i.label) for i in nlp_obama.ents]
...:
Out[10]:
[(Barack Obama, 'PERSON', 380),
 (American, 'NORP', 381),
 (44th, 'ORDINAL', 396),
 (the United States, 'GPE', 384),
 (2009, 'DATE', 391),
 (2017, 'DATE', 391),
 (first, 'ORDINAL', 396),
 (African, 'NORP', 381),
 (American, 'NORP', 381),
 (first, 'ORDINAL', 396),
 (United States, 'GPE', 384)]

```

您可以看到模型已识别的实体以及它们的准确程度（在本例中）。 PERSON不用多说，指的是人名，NORP 是国籍或宗教团体，GPE识别位置（城市，国家等），DATE识别特定日期或日期范围，ORDINAL识别代表一些顺序类型的单词或数字。  
虽然我们讨论的是Doc方法的主题，但值得一提的是spaCy的句子识别器。 NLP任务想要将文档拆分成句子 并不罕见。通过访问Doc’s.sents方法，可以很容易地使用SpaCy执行此操作：

```python
In[11]: for ix, sent in enumerate(nlp_obama.sents, 1):
...:       print("Sentence number {}: {}".format(ix, sent))
...:
Sentence number 1: Barack Obama is an American politician who served as
the 44th President of the United States from 2009 to 2017.
Sentence number 2: He is the first
African American to have served as president,
as well as the first born outside the contiguous United States.

```

就这样。在后面的文章中，我将展示如何在复杂的数据挖掘和ML任务中使用spaCy。

译自：https://dzone.com/articles/nlp-in-python (by Jayesh Bapu Ahire Mar.13.18)  
翻译：by Janvn in Aug.17.2019

测试环境：win10  
Python环境：Python 3.7.3 | Anaconda  
spaCy使用方法：https://github.com/explosion/spaCy