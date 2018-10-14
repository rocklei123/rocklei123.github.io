---
title: Hibernate缓存策略
date: 2018-09-02 16:29:56
tags: [Hibernate, 缓存]
categories: Hibernate
---

##   一、什么是Hibernate一级缓存
###   1、 一级缓存范围
（1）Hibernate 一级缓存又称之为"Session 缓存"、“会话级缓存”
（2）通过Session从数据库查询时会吧实体在内存中存储起来，下一次查询同一实体时不再从数据库获取，而是从内存中获取，这就是缓存。
（3）一级缓存的生命周期和session相同;Session 销毁它也会销毁
（4）一级缓存中的数据可适用范围在当前会话之内

测试用例（1）：
![test1](markdown-img-paste-20180902171836974.png)

<!--more-->
###   2、 清理一级缓存
一级缓存无法取消，用两个方法管理。
（1）session.evict(obj) ：会把指定的缓冲对象进行清除。
（2）  session.clear() ：把缓冲区内的全部对象清除，但不包括操作中的对象。

测试用例1session.evict(obj)  清理当前对象后，再次查询需要查询数据库：
![hibernate_evict_test](markdown-img-paste-2018090217192062.png)

测试用例2 session.clear 后（需要从数据库中查询）：
![hibernate_clear_test](markdown-img-paste-2018090217195599.png)


###  3、一级缓存注意问题：

（1）query.list()是不会使用一级缓存的
（2）query.iterate()会使用一级缓存，当缓存中有数据的时候，query.iterate()将所有对象的id查询出来然后到缓存中将所有对象都查询出来，如果缓存中没有数据，query.iterate()则把对象从数据库中一条一条的将数据查出来
（3）一级缓存也有些时候会对程序的性能产生影响，因为在对数据库进行增删改的时候同时也要更新缓存

测试用例1 query.list 不会使用缓存：
![hibernate_query_list](markdown-img-paste-20180902172130128.png)


测试用例2 query.iterate()会使用一级缓存：
![hibernate_query_iterate](markdown-img-paste-20180902172229735.png)
我们看到，当如果通过iterator()方法来获得我们对象的时候，hibernate首先会发出1条sql去查询出所有对象的 id 值，当我们如果需要查询到某个对象的具体信息的时候，hibernate此时会根据查询出来的 id 值再发sql语句去从数据库中查询对象的信息，这就是典型的 N+1 的问题。

##   二、Hibernate二级缓存

###  1、 二级缓存简介
二级缓存的生命周期是SessionFactory,当SessionFactory关闭时,缓存才会清空.
二级缓存是每个session共用的缓存,并不是默认开启的,需要手动去配置.

![introduce_hibernate_cache2](markdown-img-paste-20180902172334211.png)

###   2 、二级缓存配置步骤
1.添加二级缓存对应的jar包.
jar包:commons-logging-1.1.3.jar、ehcache.jar

2.在Hibernate的配置文件中添加Provider类的描述(即添加二级缓存接口对应外部的实现类).
<property name="cache.provider_class">net.sf.ehcache.hibernate.EhCacheProvider</property>


```xml
　　　　 <!-- 开启二级缓存 -->
        <property name="hibernate.cache.use_second_level_cache">true</property>
        <!-- 二级缓存的提供类 在hibernate4.0版本以后我们都是配置这个属性来指定二级缓存的提供类-->
        <property name="hibernate.cache.region.factory_class">org.hibernate.cache.ehcache.EhCacheRegionFactory</property>
        <!-- 二级缓存配置文件的位置 -->
        <property name="hibernate.cache.provider_configuration_file_resource_path">ehcache.xml</property>
```
3.添加二级缓存的属性配置文件,直接放在src根目录即可.
ehcache.xml

```xml
<ehcache>

    <!-- Sets the path to the directory where cache .data files are created.

         If the path is a Java System Property it is replaced by
         its value in the running VM.

         The following properties are translated:
         user.home - User's home directory
         user.dir - User's current working directory
         java.io.tmpdir - Default temp file path -->
　　
　　<!--指定二级缓存存放在磁盘上的位置-->
    <diskStore path="user.dir"/>　　

　　<!--我们可以给每个实体类指定一个对应的缓存，如果没有匹配到该类，则使用这个默认的缓存配置-->
    <defaultCache
        maxElementsInMemory="10000"　　//在内存中存放的最大对象数
        eternal="false"　　　　　　　　　//是否永久保存缓存，设置成false
        timeToIdleSeconds="120"　　　　
        timeToLiveSeconds="120"　　　　
        overflowToDisk="true"　　　　　//如果对象数量超过内存中最大的数，是否将其保存到磁盘中，设置成true
        />
　　
　　<!--
　　　　1、timeToLiveSeconds的定义是：以创建时间为基准开始计算的超时时长；
　　　　2、timeToIdleSeconds的定义是：在创建时间和最近访问时间中取出离现在最近的时间作为基准计算的超时时长；
　　　　3、如果仅设置了timeToLiveSeconds，则该对象的超时时间=创建时间+timeToLiveSeconds，假设为A；
　　　　4、如果没设置timeToLiveSeconds，则该对象的超时时间=max(创建时间，最近访问时间)+timeToIdleSeconds，假设为B；
　　　　5、如果两者都设置了，则取出A、B最少的值，即min(A,B)，表示只要有一个超时成立即算超时。
　　-->

　　<!--可以给每个实体类指定一个配置文件，通过name属性指定，要使用类的全名-->
    <cache name="com.xiaoluo.bean.Student"
        maxElementsInMemory="10000"
        eternal="false"
        timeToIdleSeconds="300"
        timeToLiveSeconds="600"
        overflowToDisk="true"
        />

    <cache name="sampleCache2"
        maxElementsInMemory="1000"
        eternal="true"
        timeToIdleSeconds="0"
        timeToLiveSeconds="0"
        overflowToDisk="false"
        /> -->


</ehcache>
```
4.如果使用xml配置，我们需要在 Student.hbm.xml 中加上一下配置。在需要被缓存的表所对应的映射文件中添加<cache/>标签.
####  配置文件配置
在<class>标签下添加<cache usage="read-only"/>


```xml
<hibernate-mapping package="com.xiaoluo.bean">
    <class name="Student" table="t_student">
        <!-- 二级缓存一般设置为只读的 -->
        <cache usage="read-only"/>
        <id name="id" type="int" column="id">
            <generator class="native"/>
        </id>
        <property name="name" column="name" type="string"></property>
        <property name="sex" column="sex" type="string"></property>
        <many-to-one name="room" column="rid" fetch="join"></many-to-one>
    </class>
</hibernate-mapping>

```

####   注解方式
如果使用annotation配置，我们需要在Student这个类上加上这样一个注解：
```java
@Entity
@Table(name="t_student")
@Cache(usage=CacheConcurrencyStrategy.READ_ONLY)　　//　　表示开启二级缓存，并使用read-only策略
public class Student
{
    private int id;
    private String name;
    private String sex;
    private Classroom room;
    .......
}
```

###  3、二级缓存注意事项
（1）二级缓存的使用策略一般有这几种：read-only、nonstrict-read-write、read-write、transactional。注意：我们通常使用二级缓存都是将其配置成 read-only ，即我们应当在那些不需要进行修改的实体类上使用二级缓存，否则如果对缓存进行读写的话，性能会变差，这样设置缓存就失去了意义。
（2）二级缓存缓存的仅仅是对象，如果查询出来的是对象的一些属性，则不会被加到缓存中去
（3）当我们如果通过 list() 去查询两次对象时，二级缓存虽然会缓存查询出来的对象，但是我们看到发出了两条相同的查询语句，这是因为二级缓存不会缓存我们的hql查询语句，要想解决这个问题，我们就要配置我们的查询缓存了。
（4）查询缓存(sessionFactory级别)

我们如果要配置查询缓存，只需要在hibernate.cfg.xml中加入一条配置即可：
```xml
　　　　　<!-- 开启查询缓存 -->
        <property name="hibernate.cache.use_query_cache">true</property>
```
然后我们如果在查询hql语句时要使用查询缓存，就需要在查询语句后面设置这样一个方法:

```java
List<Student> ls = session.createQuery("from Student where name like ?")
                    .setCacheable(true)　　//开启查询缓存，查询缓存也是SessionFactory级别的缓存
                    .setParameter(0, "%王%")
                    .setFirstResult(0).setMaxResults(50).list();
```

如果是在annotation中，我们还需要在这个类上加上这样一个注解：@Cacheable


###   4、 二级缓存使用场景

![hibernate_cache2_use](markdown-img-paste-20180902172551857.png)

##   三、Hibernate一级、二级缓存的对比
![hibernate_cache_compare](markdown-img-paste-20180902172656770.png)

## 参考
https://www.cnblogs.com/xiaoluo501395377/p/3377604.html
https://www.imooc.com/learn/465

## 欢迎关注米宝窝，持续更新中，谢谢！
 [米宝窝 https://rocklei123.github.io/ ](https://rocklei123.github.io/ "https://rocklei123.github.io/")
