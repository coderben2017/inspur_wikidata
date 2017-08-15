# 项目进展

### 8月7日
1. 使用wampserver搭建PHP运行环境（Apache+PHP+MySQL）
   - 注：mediawiki使用PHP5.5，低于5.5版本无法加载此工程。
2. 搭建本地mediawiki运行数据库
   - 注：数据库名统一使用limingming_wiki
3. 梳理mediawiki项目结构，找到入口文件（index.php）及操作入口（specials类，在\extensions\Wikibase\repo\includes\Specials\目录下，进入路径如 “mediawiki1.29.0/index.php/特殊:创建项”）

### 8月8日
1. 确定\extensions\Wikibase\repo\Wikibase.php是wikibase的数据库类，有query、select、addQuotes等方法
2. 确定添加声明的input在用户输入时会发送一个携带action、search、language等字段的ajax请求给\extensions\Wikibase\repo\Wikibase.php，在该文件中，$wgAPIModules['wbsearchentities']会拦截请求，提取action值，调用\extensions\Wikibase\repo\includes\Api\SearchEntities.php文件中的SearchEntities方法进行具体处理。
3. 已搭建好ItelliJ IDEA + XDebug调试工具，可远程调试、单步调试。

### 8月9日
1. 确定item页面是由\includes\libs\rdbms\database\Database.php中的Database类中封装的select、addQuotes、update、delete等方法进行增删改查，其中update方法操作数据库page表，select方法操作数据库wb_term表。
2. 发现进入index.php页面时，初始化过程会调用addQuotes方法、delete方法、select方法，但没有调用update方法。初始化时，select方法操作l10n_cache表，delete方法操作objectcache表。
3. 在mediawiki文档中找到了wikibase的数据库架构，理解了每张表、每个字段的作用。
    - 链接：https://www.mediawiki.org/wiki/Wikibase/Schem
4. 解决问题：
    1. 不能用localhost替代127.0.0.1
        - 比如当XDebug和PHPStorm两个工具中，一个配置了localhost，另一个配置了127.0.0.1，在debug的时候就会引起跨域，被chrome拦截ajax，导致无法load数据。
    2. 添加在statements中的字段不是item而是property
        - 它的限定符也是property类型的数据
5. item与property关系
    - 创建术语（Sprcial:NewItem）与创建属性（Sprcial:NewProperty）均保存在wb_terms表中。
    - 术语和属性在term_full_entity_id、term_entity_type两列进行区分，前者这两列是“Q2、item”，后者这两列是“P2、property”
    - 术语和属性都有三种记录类型：label（标签名）、description（描述）、alias（别名），所以每条数据在terms表中都有3条记录
    - 术语和属性都有独立的html页面，数据存储在page表中。page表的page_title列对应wb_terms表term_full_entity_id列（都存储Q2、P2这种id），page_content_model列中存储“wikibase-item”或“wikibase-property”进行区分
    - 术语的声明必须是属性（property），而不是术语（term）
    - 属性有多种类型（在创建时下拉选择），属性的限定符必须符合该属性的类型（属性与限定符间逻辑可以挖很深！）

### 8月10日
1. SpecialAllPages
  - 地址：http://localhost/mediawiki-1.29.0/index.php/Special:AllPages?from=1&to=5&namespace=120
  - 文件：mediawiki-1.29.0\includes\specials\SpecialAllPages.php
  - 参数获取：execute方法（第68行）
  - 输出HTML：outputHTMLForm方法（第104行）
2. 国际化
  - 文件: mediawiki-1.29.0\languages\*
  - 语言代号：mediawiki-1.29.0\languages\data\Names.php
  - 语言类：mediawiki-1.29.0\languages\classes\*
  - 转换JSON：mediawiki-1.29.0\languages\i18n\*
  - 转换关键信息：mediawiki-1.29.0\languages\messages\*
  - 具体执行类：
    - mediawiki-1.29.0\languages\Language.php
       - 注：里面的factory方法获取Localsetting.php内配置的语言代号，然后提取、缓存，返回对应的语言类
    - mediawiki-1.29.0\languages\ConverterRule.php
    - mediawiki-1.29.0\languages\FakeConverter.php
    - mediawiki-1.29.0\languages\LanguageCode.php
    - mediawiki-1.29.0\languages\LanguageConverter.php

3. 数据库
   -  l10n_cache表的lc_lang、lc_value两列对应mediawiki1.29.0\languages\i18n目录下的json文件，lc_key列可能对应mediawiki\1.29.0\languages\messages目录下相应的PHP文件（文件内是关键字数组）
   -  wb_property_info表存放我们创建的property的信息，pi_type列是property的数据类型，pi_info列是property特定格式的数据类型。

### 8月11日
1. wikibase的namespace
  - 参考文献
    - https://www.mediawiki.org/wiki/Extension:Wikibase_Repository/zh#Setting_up_items_in_the_main_namespace
  - 文件路径
    - mediawiki-1.29.0\extensions\Wikibase\repo\config\ Wikibase.example.php
  - 涉及变量
    1. $baseNs —— 主命名空间（值 = 120）
    2. \$WB_NS_ITEM —— 术语命名空间（值 = $baseNs = 120）
    3. \$WB_NS_PROPERTY —— 属性命名空间（值 = $baseNs+2 = 122）
    4. $wgExtraNamespaces —— 全局数组变量，存放命名空间
       - 例：$wgExtraNamespaces[WB_NS_ITEM] = 'Item'
    5. $wgWBRepoSettings —— 全局二维数组变量，存放实体与命名空间的对应关系，“由它告诉wikibase哪个命名空间使用哪种实体”
       - 例：$wgWBRepoSettings\['entityNamespaces']['item'] = WB_NS_ITEM;
  - 数据库
    - page表的page_namespace列存储namespace的id（120、122）
2. 实体操作涉及的JS
  - 文件路径
    - mediawiki-1.29.0\extensions\Wikibase\vendor\wikibase\javascript-api\src\RepoApi.js
  - 说明
    - 该文件定义了一个api类——WbApiRepoApi，并在它的原型上扩展了createEntity、editEntity、getEntities、searchEntities等方法，每个方法均以ajax的形式向后端提交执行该操作所需要的字段信息，如action、search、id
3. GitHub Wikidata工具库——SQID工具
  1. 概要
    - SQID是一个用来解析、浏览、查询维基数据的工具。
    - 它是一个前端应用，基于AngularJS、jQuery和Bootstrap构建，使用SQID/src/data/exampleData目录下的classes.json、properties.json和statistics.json来充当“数据库”（只是wikidata库的一部分）。
    - 关于JSON格式的说明记录在SQID/src/data/format.md。
    - 它的JSON库并不是自动更新的，而是由wiki的项目组每一两周更新维护一次（基于wikidata的库），也提供了基于Python的更新脚本（即爬虫，文件路径是SQID/helpers/python/dataUpdate.py），供工具使用者自己更新。
  2. 用途
    - 快速查询维基百科数据，轻量级wikidata浏览工具（可能是类似教学、示范的作用。。）

### 8月14日
1. 整理“创建属性”功能的SQL模板
  - wikibase创建新属性时使用的是后端通信，没有用Ajax。
  - 在mediawiki-1.29.0\includes\libs\rdbms\database\Database.php中的query方法内加入输出语句，这样正常创建属性时就把执行的所有SQL中的INSERT和UPDATE语句写入TXT文本文件，然后我们越过系统，直接在新数据库上执行这些INSERT和UPDATE，模拟创建属性，然后测试系统能否找到并使用这个属性，最终测试通过。
  - 该模板已整理，换库可用（只需要修改id、rev_id、page_id等字段来适配本地库）
2. 梳理创建属性property
  - 分类（13种）
    1. Commons media file    共享资源媒体文件
    2. External identifier    外部标识符
    3. Geographic coordinates    地理坐标
    4. Geographic shape    地理形状
    5. Item    项
    6. Monolingual text    单语文本
    7. Point in time    时间点
    8. Peroperty    属性
    9. Quantity    数量
    10. String    字符串
    11. Tabular data     表格数据
    12. URL     统一资源定位符
    13. Mathematical expression      数学表达式
  - 对应数据库 —— wb_property_info表
    - pi_property_id存储property的id，pi_type字段存储property类型，pi_info存储json格式的property类型

### 8月15日
1. 解析wikibase的html渲染方式
  - html涉及文件：
    - mediawiki-1.29.0\extensions\Wikibase\view\src\ItemView.php
    - mediawiki-1.29.0\extensions\Wikibase\view\src\PropertyView.php
    - mediawiki-1.29.0\extensions\Wikibase\view\src\EntityView.php
    - mediawiki-1.29.0\extensions\Wikibase\view\src\Template\ TemplateFactory.php
    - 注：可以通过EntityView.php中的getHTML()方法截取新增页面时的渲染的html，同时我们验证到，查询item时查询结果页面的生成没有走这个函数。
  - php => html DOM 渲染模板
    - mediawiki-1.29.0\extensions\Wikibase\view\resources\templates.php
  - jquery、ui资源索引文件
    -  mediawiki-1.29.0\extensions\Wikibase\view\resources\jquery\ resources.php
    -  mediawiki-1.29.0\extensions\Wikibase\view\resources\jquery\ui\ resources.php
    -  mediawiki-1.29.0\extensions\Wikibase\view\resources\wikibase\ resources.php
  - jquery、ui资源目录
     -  mediawiki-1.29.0\extensions\Wikibase\view\resources\jquery\*
2. 整理查询item、查询property涉及的sql语句
3. 整理移除声明（property）涉及的sql
  - 移除一条声明即解除这个property与当前item的关系，在数据库wb_changes表的change_type字段中显示为wikibase-item~update，在change_info字段中存储一个json，记录该操作的详细信息
  - 移除声明、添加声明都会在recentchanges表中插入一条记录，操作类型体现在rc_comment字段中。
    - 移除时值为'/* wbsetclaim-create:2||1 */ [[Property:P3]]: [[Item:Q2]]'
    - 添加时值为'/* wbremoveclaims-remove:1| */ [[Property:P3]]: [[Item:Q2]]'
  - site_stats表的ss_total_edits字段值加1，记录编辑次数
  - searchindex表中新增了一条记录（但字段值是一种特殊编码，没看懂）