title:  "使用关系型数据库表示树形结构"
date: 2016-11-29 18:20:30
tags:
    - MySQL
    - 技术
---

树形结构是很自然、常用的一种数据结构，比如公司的组织架构、目录树、某些分类系统。但是我们目前的数据系统大多是关系型的与树形结构天然异构，所以用关系型数据库存储树状结构往往很复杂。
要想在关系数据库中存树形结构主要有以下四种方案
1. 邻接表(Adjacency list)
2. 路径枚举(Path enumeration)
3. 嵌套集合(Nested sets)
4. 闭包表(Closure table)

以以下树形结构为例，说下这四种树形结构存储方式的利弊
![一颗树](/img/mysql_tree.jpg)
先考虑实际使用场景，常见的树形结构的操作有以下7种

1. 给定节点求直接子节点
2. 给定节点求其父节点
3. 给定节点求节点及其节点子树
4. 给定节点求节点到根的路径
5. 移动节点及所在的子树
6. 删除节点及所在子树
7. 插入一个新节点

操作6跟操作3实际是相同的，只不过一个是SELECT，一个是DELETE。操作5移动节点，实际是选取节点及子树进行删除，然后按照规则在目标节点下进行INSERT的复合操作，也就是6+7。
所以实际就是下面5中操作

1. 给定节点求直接子节点
2. 给定节点求其父节点
3. 给定节点求节点子树
4. 给定节点求节点到根的路径
5. 插入一个新节点


### 邻接表
邻接表是最简单的表示树形结构的方法，因为子节点与其父节点是一对一的关系，邻接表在节点表中存储每个节点的父节点。当然这种做法的问题就是，如果你想求一个节点的子节点，相当麻烦。。。

```sql
CREATE TABLE `Node_al` (
    `id` INT(10) NOT NULL,
    `name` VARCHAR(10) NOT NULL,
    `parent_id` INT(10) NOT NULL,
    PRIMARY KEY(id)
);

INSERT INTO `Node_al` VALUES (1, 'N1', 0), (2, 'N2', 1), (3, 'N3', 1), (4, 'N4', 2), (5, 'N5', 3), (6, 'N6', 3), (7, 'N7', 4), (8, 'N8', 4);
```

最后的表如下图
![邻接表](/img/aj_list_basic.jpg)


考察以上5种操作：

1. 求节点1的所有直接子节点
因为有parent_id， 所以只需SELECT所有parent_id=目标ID的节点就能够获得其直接子节点
![邻接表求直接子节点](/img/aj_direct_children.jpg)

2. 求节点2的父节点
![邻接表求直接子节点](/img/aj_direct_parent.jpg)

3. 求节点1的所有子孙节点
求一个节点的子节点节点需要左连接一次，我们要从根节点到叶子节点有多少层就需要左连接多少次。为了清晰，我只取id
![邻接表求所有子节点](/img/aj_sub_children.jpg)

4. 求节点8到根的路径
求到跟的路径需要一个循环依次找到路径上每个节点的父节点，直到根

5. 在节点5上插入节点9
```sql
INSERT INTO `Node_al` VALUES (9, 'N9', 5);
```

对于移动操作，邻接表无需进行删除重建，而只需要更新parent_id即可

### 路径枚举
路径枚举指的是将节点的路径作为一个字段存到数据表中，比如下面这样

```sql
CREATE TABLE `Node_pe` (
    `id` INT(10) NOT NULL,
    `name` VARCHAR(10) NOT NULL,
    `path` VARCHAR(128) NOT NULL,
    PRIMARY KEY(id)
);

INSERT INTO `Node_pe` VALUES (1, 'N1', '/1'), (2, 'N2', '/1/2'), (3, 'N3', '/1/3'), (4, 'N4', '/1/2/4'), (5, 'N5', '/1/2/5'), (6, 'N6', '/1/3/6'), (7, 'N7', '/1/3/7'), (8, 'N8', '/1/2/4/8');
```

最后的表如下图
![路径枚举](/img/pe_basic.jpg)

这样做的好处就是无需计算一个节点所在的路径，因为已经存在表中了

考察以上5种操作
1. 求节点2的所有直接子节点
![求路径下所有直接子节点 ](/img/pe_direct_children.jpg)

2. 求节点2的父节点
SELECT n1.* FROM Node n1 JOIN Node n2 ON n1.id = n2.parent_id WHERE n2.id = 2;

3. 求节点2及其所有子孙节点
![求路径下所有子节点 ](/img/pe_sub_children.jpg)

4. 求节点8到根的路径
![求节点到根的路径](/img/pe_path.jpg)

5. 在节点5上插入节点9，要先插入后更新
```sql
    INSERT INTO `Node_pe` VALUES (9, 'N9', '');
    SELECT path INTO @parent_path FROM `Node_pe` WHERE id = 5;
    UPDATE `Node_pe` SET path = @parent_path||LASTER_INSERT_ID()||'/' WHERE id = LAST_INSERT_ID();
```


### 嵌套集

嵌套集合将树形结构想象成一系列的集合，比如下面这样
![一个嵌套集合](/img/nest_set.jpg)
```sql
CREATE TABLE `Node_ne` (
    `id` INT(10) NOT NULL,
    `name` VARCHAR(10) NOT NULL,
    `lft` INT(10) NOT NULL,
    `rgh` INT(10) NOT NULL,
    PRIMARY KEY(id)
);

INSERT INTO `Node_ne` VALUES (1, 'N1', 1, 16), (2, 'N2', 2, 9), (3, 'N3', 10, 15), (4, 'N4', 3, 8), (5, 'N5', 11, 12), (6, 'N6', 13, 14), (7, 'N7', 4, 5), (8, 'N8', 6, 7);
```

考察以上5种操作：

1. 求节点2的所有直接子节点
似乎没有一个方法来求取，集合的强项是求解包含关系。所以可以辅助邻接表求取

2. 求节点2的父节点
![节点2的父节点](/img/ne_direct_parent.jpg)

3. 求节点2的所有子孙节点
![节点2的所有子孙节点](/img/ne_sub_children.jpg)

4. 求节点8到根的路径
![节点8到根的路径](/img/ne_path.jpg)

5. 插入节点
```sql
    呵呵
```
是的，嵌套集并不是稳定的结构，如果你要插入一个需要改变这个节点右边所有节点的权重，于是有个种改变权重配置的方法，但是如果你的程序足够复杂，请不要花心事在这个上面

### 闭包表

```sql
CREATE TABLE `Node_ct` (
    `id` INT(10) NOT NULL,
    `name` VARCHAR(10) NOT NULL,
    PRIMARY KEY(id)
);

CREATE TABLE `Node_ct_relation` (
    `node_id` INT(10) NOT NULL,
    `ancestor` INT(10) NOT NULL,
);

INSERT INTO `Node_ct` VALUES (1, 'N1'), (2, 'N2'), (3, 'N3'), (4, 'N4'), (5, 'N5'), (6, 'N6'), (7, 'N7'), (8, 'N8');
INSERT INTO `Node_ct_relation` VALUES (1, 1), (2, 1), (2, 2), (4, 1), (4, 2), (4, 4), (7, 1), (7, 2), (7, 4), (7, 7), \
(8, 1), (8, 2), (8, 4), (8, 8), (3, 1), (3, 3), (5, 1), (5, 3), (5, 5), (6, 1), (6, 3), (6, 6);
```

考察以上5种操作：

1. 求节点2的所有直接子节点

2. 求节点2的父节点

3. 求节点2的所有子孙节点

4. 求节点8到根的路径

5. 插入节点
