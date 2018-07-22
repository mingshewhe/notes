这里讲的是Spring与mybatis简单使用。
# 配置bean
```java
@Bean
public DataSource dataSource() {
    DruidDataSource dataSource = new DruidDataSource();
    dataSource.setDriverClassName(environment.getProperty("driverClass"));
    dataSource.setUrl(environment.getProperty("url"));
    dataSource.setUsername(environment.getProperty("d_username"));
    dataSource.setPassword(environment.getProperty("password"));
    return dataSource;
}

@Bean
public SqlSessionFactory sqlSessionFactory(DataSource dataSource) throws Exception {
    SqlSessionFactoryBean sqlSessionFactoryBean = new SqlSessionFactoryBean();
    sqlSessionFactoryBean.setDataSource(dataSource);
    sqlSessionFactoryBean.setTypeAliasesPackage("com.ming.web.domain");
    return sqlSessionFactoryBean.getObject();
}

public MapperFactoryBean<UserMapper> userMapper(SqlSessionFactory sqlSessionFactory) {
    MapperFactoryBean<UserMapper> factoryBean = new MapperFactoryBean<>(UserMapper.class);
    factoryBean.setSqlSessionFactory(sqlSessionFactory);
    return factoryBean;
}
```
# 创建UserMapper
```java
public interface UserMapper {
    User getUser(long userId);
}
```
# 创建UserMapper.xml
这里得注意的是，UserMapper.xml必须和UserMapper路径是一样的。比如UserMapper全名师com.ming.dao.UserMapper,那么UserMapper.xml必须放在com/ming/dao/UserMapper.xml资源路径下。当然这个路径可以在sqlSessionFactoryBean中配置修改。
```java
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<!--namespace是UserMapper类全路径-->
<mapper namespace="com.ming.web.dao.UserMapper">

    <select id="getUser" resultType="User">/*dialect*/
        SELECT * FROM t_user WHERE id = #{userId}
    </select>
</mapper>
```