---
slug: advanced-database-select-usage-in-zf2
date: '2012-06-08 17:07:27'
title: Zend Framework 2.0 (zf2) 进阶数据库操作
id: 148
tags:
  - SQL
  - ZF2
  - Zend Framework 2
  - DB
---

[Zend Framework 2](http://avnpc.com/pages/zf2-summary)完全重写了Zend1的数据库组件，但是目前手册给的例子都弱爆了，只能自己整理一些[Zend\Db\TableGateway用例](http://avnpc.com/pages/advanced-database-select-usage-in-zf2)，详见下文。

连接数据库
===========

首选的操作DB方式还是继承TableGateway，比如现在要操作的数据库为test，数据表为mydbtable，新建一个类如下

    class MyDbTable extends Zend\Db\TableGateway\TableGateway
    {
    }

连接数据库驱动推荐用pdo


    $adapter = new Zend\Db\Adapter\Adapter(array(
        'driver' => 'pdo',
        'dsn' => 'mysql:dbname=test;hostname=localhost',
        'username' => 'root',
        'password' => 'passwoard',
        'driver_options' => array(
            PDO::MYSQL_ATTR_INIT_COMMAND => 'SET NAMES \'UTF8\''
        ),
    ));
    
    $myDbTable = new MyDbTable('mydbtable', $adapter);

这样子数据库的连接就完成了。

执行Transaction
===========

    $adapter->getDriver()->getConnection()->beginTransaction();

数据库会执行SQL语句：

    START TRANSACTION

不过不是所有的驱动都具备Transaction功能，还是推荐使用pdo

WHERE 链式操作
================

ZF2推荐的查询方式为链式操作。

    $select = $myDbTable->getSql()->select();
    $select->where('id > 1')->order('id DESC')->limit(10);
    $resultSet = $myDbTable->selectWith($select);
    $result = $resultSet->toArray();
    
执行的SQL语句为：

    SELECT `mydbtable`.* FROM `mydbtable` WHERE id > 1 ORDER BY `id` DESC LIMIT 10

WHERE 闭包操作
==============

zf2还支持闭包式的操作，上例可以以闭包方式改写成：

    $select->where(function($where){
        $where->lessThan('id', 10);
        $where->greaterThan('id', 5);
        return $where;
    })->order('id DESC')->limit(10);
    
SQL:

    SELECT `mydbtable`.* FROM `mydbtable` WHERE `id` < '10' AND `id` > '5' ORDER BY `id` DESC LIMIT 10

WHERE 非闭包操作
==============

如果不想用闭包，也可以在Zend\Db\Sql\Select中调用对象where从而在外部获得where的一个实例进行操作

    $select = $myDbTable->getSql()->select();
    $where = $select->where;
    $where->lessThan('id', 10);
    $where->greaterThan('id', 5);
    $select->order('id DESC')->limit(10);
    $myDbTable->selectWith($select);

SQL:

    SELECT `mydbtable`.* FROM `mydbtable` WHERE `id` < '10' AND `id` > '5' ORDER BY `id` DESC LIMIT 10

WHERE AND复合条件
================

    $select->where(
        array('id > 30')
    )->where(
        array('id < 10')
    );

SQL:

    SELECT `mydbtable`.* FROM `mydbtable` WHERE id > 30 AND id < 10
    

WHERE OR 复合条件
==================

    $select = $myDbTable->getSql()->select();
    $select->where(
        array('id > 30')
    )->where(
        array('id < 10'), \Zend\Db\Sql\Where::OP_OR
    );
    
SQL:

    SELECT `mydbtable`.* FROM `mydbtable` WHERE id > 30 OR id < 10

还可以这样写，效果是一样。这是通过魔术方法__call()实现的。

    $where = $select->where;
    $where->lessThan('id', 10);
    $where->or;
    $where->greaterThan('id', 30);
    
WHERE 复杂条件
===============

如果查询条件进一步复杂，比起链式操作来，闭包操作更具有灵活性，比如：

    $select->where(function($where){

        $subWhereForId = clone $where;
        $subWhereForTitle = clone $where;

        $subWhereForId->lessThan('id', 10);
        $subWhereForId->or;
        $subWhereForId->greaterThan('id', 20);

        $where->addPredicate($subWhereForId);

        $subWhereForTitle->equalTo('title', 'a');
        $subWhereForTitle->or;
        $subWhereForTitle->equalTo('title', 'b');
        $where->addPredicate($subWhereForTitle);

        return $where;
    });
    
等同于SQL：

    SELECT `mydbtable`.* FROM `mydbtable` WHERE (`id` < '10' OR `id` > '20') AND (`title` = 'a' OR `title` = 'b')
    
表达式
===============

当查询条件里有SQL函数时，需要使用Zend\Db\Sql\Expression生成SQL表达式。

例如

    $select->where(array(
        'id' => new Zend\Db\Sql\Expression('NOW()')
    ));

SQL：

    SELECT `mydbtable`.* FROM `mydbtable` WHERE `id` = NOW()

下面这个例子是常用的操作，查询指定的ID并使用指定的顺序排序。综合使用了上面提到的知识。

    $idArray = array('2', '1');
    $select->where(array(
        'id' => $idArray
    ));
    $order = sprintf('FIELD(id, %s)', implode(',', array_fill(0, count($idArray), Zend\Db\Sql\Expression::PLACEHOLDER)));
    $select->order(array(new Zend\Db\Sql\Expression($order, $idArray)));

SQL

    SELECT `mydbtable`.* FROM `mydbtable` WHERE `id` IN ('2', '1') ORDER BY FIELD(id, '2','1')
    
EvaEngine的改进
================

在[EvaEngine](http://avnpc.com/pages/eva-engine)中，可以使用完整的链操作，最开始的例子在EvaEngine里可以这样写：

    $result = $myDbTable->where('id > 5')->order('id DESC')->limit(10)->find();

有兴趣的话不妨参看[Eva\Db\TableGateway\TableGateway](https://github.com/AlloVince/eva-engine/blob/master/vendor/Eva/Db/TableGateway/TableGateway.php)的实现。