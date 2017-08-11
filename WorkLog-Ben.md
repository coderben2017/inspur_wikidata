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
  - 文件目录
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