---
layout: post
title:  "solr创建core以及整合ik分词器"
date:   2019-04-09
excerpt: "想整合IK中文分词器到solr，得先创建个core，然后复制文件改改配置就是了。"
tag:
- 分词器
- 搜索
- Solr
- Web后端
comments: true
---

## Solr中创建core

1. 首先创建一个目录，这个前面的文章说过了。P.S.最好别放在solr目录下，免得后面solr索引文件太大导致tomcat运行不顺利。示例目录名为“solrHome”。

2. 把下载的solr文件中server\solr里的目录和文件都复制到上一步中创建的solrHome里。

3. 在solrHome里创建目录，这个就是核心core的目录。示例目录名为“mycore”。

4. 将solr-5.5.0\server\solr\configsets\basic_configs里的文件都复制到上面创建的mycore文件夹中。

5. 打开admin后台，Core Admin -> Add Core，填写之前创建的目录名mycore。

![Smithsonian Image](https://yawwq.github.io/assets/img/solr创建core以及整合ik分词器/1.png)

## 配置IK中文分词器

下载地址：http://files.cnblogs.com/files/qinxuanyu/ik%E5%88%86%E8%AF%8Dsolr5.x.rar

这个适用于solr5.5，其他版本的大家自己搜索一哈。一定要好好查一查，有时配文件很多问题都是出在版本不匹配上面。

1. 将IKAnalyzer2012FF_u2.jar文件复制到tomcat目录webapps\solr5.5\WEB-INF\lib下

2. 将IKAnalyzer.cfg.xml和stopword.dic复制到tomcat目录webapps\solr5.5\WEB-INF\classes下（classes文件前面文章说过，是由自己创建的）。

3. mycore\conf目录下managed-schema文件，在文件中增加如下节点：


	<field name="solrname" type="text_ik" indexed="true" stored="true" omitNorms="true"/>

	<fieldType name="text_ik" class="solr.TextField">
	    <analyzer class="org.wltea.analyzer.lucene.IKAnalyzer"/>
	</fieldType>

进核心中的Analysis选择text_ik测试一下，可以看到分词器已经成功配置了：

![Smithsonian Image](https://yawwq.github.io/assets/img/solr创建core以及整合ik分词器/2.png)