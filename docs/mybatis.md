#### 添加依赖
```
<dependency>
	<groupId>com.jeesuite</groupId>
	<artifactId>jeesuite-mybatis</artifactId>
	<version>1.0.0</version>
</dependency>
```
#### 一些说明
##### 代码生成及其自动crud
说明：这一部分代码我实现了一个简单的版本，后面发现有几个比较优秀的mybatis增强框架专注做自动crud功能的，所以已经做了无缝对接推荐使用它们。
两个框架都提供了代码生成的方法，这里不多说。
* [Mapper](http://git.oschina.net/free/Mapper)
* [mybatis-plus](http://git.oschina.net/baomidou/mybatis-plus)

---
##### 其他功能说明
* 读写分离：已经线上稳定运行一年
* 自动缓存：已经线上稳定运行一年
* 分库路由：只适用基本的分库路由，不支持跨库join等，未上过生产系统。

#### 配置
##### mysql.properties配置，为了减少配置限定读取mysql.properties这个名字
```
#mysql global config
db.shard.size=1000
db.driverClass=com.mysql.jdbc.Driver
db.initialSize=2
db.minIdle=1
db.maxActive=10
db.maxWait=60000
db.timeBetweenEvictionRunsMillis=60000
db.minEvictableIdleTimeMillis=300000
db.testOnBorrow=false
db.testOnReturn=false

#master
master.db.url=jdbc:mysql://localhost:3306/demo_db?seUnicode=true&amp;characterEncoding=UTF-8
master.db.username=root
master.db.password=123456
master.db.initialSize=2
master.db.minIdle=2
master.db.maxActive=20

#slave ....
slave1.db.url=jdbc:mysql://localhost:3306/demo_db2?seUnicode=true&amp;characterEncoding=UTF-8
slave1.db.username=root
slave1.db.password=123456

#slave2 ....
slave2.db.url=jdbc:mysql://localhost:3306/demo_db3?seUnicode=true&amp;characterEncoding=UTF-8
slave2.db.username=root
slave2.db.password=123456

#如果你要使用分库功能
#master
group1.master.db.url=jdbc:mysql://localhost:3306/demo_db?seUnicode=true&amp;characterEncoding=UTF-8
group1.master.db.username=root
group1.master.db.password=123456
group1.master.db.initialSize=2
group1.master.db.minIdle=2
group1.master.db.maxActive=20

#slave ....
group1.slave1.db.url=jdbc:mysql://localhost:3306/demo_db2?seUnicode=true&amp;characterEncoding=UTF-8
group1.slave1.db.username=root
group1.slave1.db.password=123456
```
slave1~n,group1~n依次递增(第一组group0默认省略)
#####  spring配置
```
    <!--除了datasource配置外，其他都与原生配置一样-->
    <bean id="routeDataSource" class="com.jeesuite.mybatis.datasource.MutiRouteDataSource" />

	<bean id="transactionTemplate"
		class="org.springframework.transaction.support.TransactionTemplate">
		<property name="transactionManager" ref="transactionManager" />
		<property name="isolationLevelName" value="ISOLATION_DEFAULT" />
		<property name="propagationBehaviorName" value="PROPAGATION_REQUIRED" />
	</bean>

	<bean id="transactionManager"
		class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
		<property name="dataSource" ref="routeDataSource" />
	</bean>

	<bean id="routeSqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
		<property name="configLocation" value="classpath:mybatis-configuration.xml" />
		<property name="mapperLocations" value="classpath:mapper/*Mapper.xml" /> 
		<property name="dataSource" ref="routeDataSource" />
		<property name="typeAliasesPackage" value="com.jeesuite.demo.dao.entity" />
		<property name="plugins">
            <array>
                <bean class="com.jeesuite.mybatis.plugin.PluginsEntrypoint">
                    <property name="interceptorHandlers">
                       <list> 
                           <!--开启自动缓存-->
                           <bean class="com.jeesuite.mybatis.plugin.cache.CacheHandler" />
                           <!--开启读写分离-->
                           <bean class="com.jeesuite.mybatis.plugin.rwseparate.RwRouteHandler">
                              <!--CRUD框架驱动 mapper3，mybatis-plus,默认是mapper3-->
                              <property name="crudDriver" value="mapper3" />
                           </bean>
                           <!-- 
                            开启分库路由
                           <bean class="com.jeesuite.mybatis.plugin.shard.DatabaseRouteHandler" /> 
                           -->
                       </list>
                    </property>
                </bean>
            </array>
        </property>
	</bean>
	<!--集成Mapper框架的配置-->
	<bean class="tk.mybatis.spring.mapper.MapperScannerConfigurer">
        <property name="sqlSessionFactoryBeanName" value="routeSqlSessionFactory" />
		<property name="basePackage" value="com.jeesuite.demo.dao.mapper" />
    </bean>
```
#### 如何使用自动缓存
只需要在mapper接口增加@com.jeesuite.mybatis.plugin.cache.annotation.Cache标注
```
public interface UserEntityMapper extends BaseMapper<UserEntity> {
	
	@Cache
	public UserEntity findByMobile(@Param("mobile") String mobile);
	
	@Cache(uniqueResult = false,keyPattern = "UserEntity.type:%s")
	List<String> findByType(@Param("type") int type);
}
```
配置完成即可，查询会自动缓存、更新会自动更新。