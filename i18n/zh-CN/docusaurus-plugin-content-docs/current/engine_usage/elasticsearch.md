---
title: ElasticSearch 引擎
sidebar_position: 11
---

本文主要介绍在 Linkis1.X 中，ElasticSearch 引擎的配置、部署和使用。

## 1. 环境准备

如果您希望在您的服务器上使用 ElasticSearch 引擎，您需要准备 ElasticSearch 服务并提供连接信息，如 ElasticSearch 集群的连接地址、用户名和密码等

## 2. 部署和配置

### 2.1 版本的选择和编译
注意: 编译 ElasticSearch 引擎之前需要进行 Linkis 项目全量编译  
发布的安装部署包中默认不包含此引擎插件，
你可以按[Linkis引擎安装指引](https://linkis.apache.org/zh-CN/blog/2022/04/15/how-to-download-engineconn-plugin)部署安装 ，或者按以下流程，手动编译部署

单独编译 ElasticSearch 引擎 

```
${linkis_code_dir}/linkis-enginepconn-lugins/engineconn-plugins/elasticsearch/
mvn clean install
```

### 2.2 物料的部署和加载

将 2.1 步编译出来的引擎包,位于
```bash
${linkis_code_dir}/linkis-engineconn-plugins/engineconn-plugins/jdbc/target/out/elasticsearch
```
上传到服务器的引擎目录下
```bash 
${LINKIS_HOME}/lib/linkis-engineplugins
```
并重启linkis-engineplugin（或者通过引擎接口进行刷新）
```bash
cd ${LINKIS_HOME}/sbin
sh linkis-daemon.sh restart cg-engineplugin
```
### 2.3 引擎的标签

Linkis1.X是通过标签来进行的，所以需要在我们数据库中插入数据，插入的方式如下文所示。

管理台的配置是按照引擎标签来进行管理的，如果新增的引擎，有配置参数需要配置的话，需要修改对应的表的元数据

```
linkis_ps_configuration_config_key:  插入引擎的配置参数的key和默认values
linkis_cg_manager_label：插入引擎label如：hive-2.3.3
linkis_ps_configuration_category： 插入引擎的目录关联关系
linkis_ps_configuration_config_value： 插入引擎需要展示的配置
linkis_ps_configuration_key_engine_relation:配置项和引擎的关联关系
```

```sql

SET @ELASTICSEARCHENG_LABEL="elasticsearch-7.6.2";
SET @ELASTICSEARCHENG_ALL=CONCAT('*-*,',@ELASTICSEARCHENG_LABEL);
SET @ELASTICSEARCHENG_IDE=CONCAT('*-IDE,',@ELASTICSEARCHENG_LABEL);

insert into `linkis_cg_manager_label` (`label_key`, `label_value`, `label_feature`, `label_value_size`, `update_time`, `create_time`) VALUES ('combined_userCreator_engineType',@ELASTICSEARCHENG_ALL, 'OPTIONAL', 2, now(), now());
insert into `linkis_cg_manager_label` (`label_key`, `label_value`, `label_feature`, `label_value_size`, `update_time`, `create_time`) VALUES ('combined_userCreator_engineType',@ELASTICSEARCHENG_IDE, 'OPTIONAL', 2, now(), now());

select @label_id := id from linkis_cg_manager_label where `label_value` = @ELASTICSEARCHENG_IDE;
insert into linkis_ps_configuration_category (`label_id`, `level`) VALUES (@label_id, 2);


INSERT INTO `linkis_ps_configuration_config_key` (`key`, `description`, `name`, `default_value`, `validate_type`, `validate_range`, `engine_conn_type`, `is_hidden`, `is_advanced`, `level`, `treeName`) VALUES ('linkis.elasticsearcheng.url', 'such as:http://127.0.0.1:8080', 'conn address', 'http://127.0.0.1:8080', 'Regex', '^\\s*http://([^:]+)(:\\d+)(/[^\\?]+)?(\\?\\S*)?$', 'elasticsearcheng', 0, 0, 1, '数据源配置');
INSERT INTO `linkis_ps_configuration_config_key` (`key`, `description`, `name`, `default_value`, `validate_type`, `validate_range`, `engine_conn_type`, `is_hidden`, `is_advanced`, `level`, `treeName`) VALUES ('linkis.elasticsearcheng.catalog', 'catalog', 'catalog', 'system', 'None', '', 'elasticsearcheng', 0, 0, 1, '数据源配置');
INSERT INTO `linkis_ps_configuration_config_key` (`key`, `description`, `name`, `default_value`, `validate_type`, `validate_range`, `engine_conn_type`, `is_hidden`, `is_advanced`, `level`, `treeName`) VALUES ('linkis.elasticsearcheng.source', 'source', 'source', 'global', 'None', '', 'elasticsearcheng', 0, 0, 1, '数据源配置');


-- elasticsearcheng-*
insert into `linkis_ps_configuration_key_engine_relation` (`config_key_id`, `engine_type_label_id`)
(select config.id as `config_key_id`, label.id AS `engine_type_label_id` FROM linkis_ps_configuration_config_key config
INNER JOIN linkis_cg_manager_label label ON config.engine_conn_type = 'elasticsearcheng' and label_value = @ELASTICSEARCHENG_ALL);

-- elasticsearcheng default configuration
insert into `linkis_ps_configuration_config_value` (`config_key_id`, `config_value`, `config_label_id`)
(select `relation`.`config_key_id` AS `config_key_id`, '' AS `config_value`, `relation`.`engine_type_label_id` AS `config_label_id` FROM linkis_ps_configuration_key_engine_relation relation
INNER JOIN linkis_cg_manager_label label ON relation.engine_type_label_id = label.id AND label.label_value = @ELASTICSEARCHENG_ALL);

```

### 2.4 ElasticSearch 引擎相关配置

| 配置                     | 默认值          |是否必须    | 说明                                     |
| ------------------------ | ------------------- | ---|---------------------------------------- |
| linkis.es.cluster        | 127.0.0.1:9200    |是  | ElasticSearch 集群，多个节点使用逗号分隔 |
| linkis.es.username       | 无    |否              | ElasticSearch 集群用户名                 |
| linkis.es.password       | 无       |否           | ElasticSearch 集群密码                   |
| linkis.es.auth.cache     | false       |否        | 客户端是否缓存认证                       |
| linkis.es.sniffer.enable | false          |否     | 客户端是否开启 sniffer                   |
| linkis.es.http.method    | GET               |否  | 调用方式                                 |
| linkis.es.http.endpoint  | /_search           |否 | JSON 脚本调用的 Endpoint                 |
| linkis.es.sql.endpoint   | /_sql             |否  | SQL 脚本调用的 Endpoint                  |
| linkis.es.sql.format     | {"query":"%s"} |否| SQL 脚本调用的模板，%s 替换成 SQL 作为请求体请求Es 集群 |
| linkis.es.headers.* | 无 |否| 客户端 Headers 配置 |
| linkis.engineconn.concurrent.limit | 100|否 | 引擎最大并发 |

## 3. ElasticSearch引擎使用
### 3.1 准备操作
您需要配置ElasticSearch的连接信息，包括连接地址信息或用户名密码(如果启用)等信息。

![ElasticSearch](https://user-images.githubusercontent.com/22620332/182787195-8051bf25-1e1e-47e5-ad88-4896278857f2.png)  

图3-1 ElasticSearch配置信息

您也可以再提交任务接口中的params.configuration.runtime进行修改即可
```shell
linkis.es.cluster
linkis.es.username               |
linkis.es.password
```

### 3.2 通过Linkis SDK进行使用

Linkis提供了Java和Scala 的SDK向Linkis服务端提交任务. 具体可以参考 [JAVA SDK Manual](../user_guide/sdk-manual.md).
对于ElasticSearch任务您只需要修改Demo中的EngineConnType和CodeType参数即可:

```java
        Map<String, Object> labels = new HashMap<String, Object>();
        labels.put(LabelKeyConstant.ENGINE_TYPE_KEY, "elasticsearch-7.6.2"); // required engineType Label
        labels.put(LabelKeyConstant.USER_CREATOR_TYPE_KEY, "hadoop-IDE");// required execute user and creator
        labels.put(LabelKeyConstant.CODE_TYPE_KEY, "elasticsearch"); // required codeType
```

### 3.2 通过Linkis-cli进行任务提交
**-codeType参数说明**
- sql/essql：
- json/esjson：设置请求参数通过json格式的方式提交任务

**使用示例**

Linkis 1.0后提供了cli的方式提交任务，我们只需要指定对应的EngineConn和CodeType标签类型即可，ElasticSearch的使用如下：
```shell
sh ./bin/linkis-cli -submitUser hadoop -engineType elasticsearch-7.6.2 -codeType json -code '{"query": {"match": {"order_id": "584677"}}}' -runtimeMap linkis.es.http.method=GET -runtimeMap linkis.es.http.endpoint=/kibana_sample_data_ecommerce/_search
```
具体使用可以参考： [Linkis CLI Manual](../user_guide/linkiscli-manual.md).


## 4. ElasticSearch引擎的用户设置

ElasticSearch的用户设置主要是设置ElasticSearch的连接信息，但是建议用户将此密码等信息进行加密管理。
