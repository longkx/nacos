# Nacos 国产化数据库适配说明文档

## 概述

本适配工作于2026年1月29日由 longkaixiang 完成，主要实现了 Nacos 对国产数据库 **达梦(DM)** 和 **人大金仓(Kingbase)** 的支持。适配基于 Nacos 3.1.1 版本，遵循了原有的插件化数据源架构。

### 适配的主要改动

#### 1. 依赖添加
在 `console/pom.xml` 和 `naming/pom.xml` 中添加了两种国产数据库的 JDBC 驱动依赖：

```xml
<!-- 达梦数据库驱动 -->
<dependency>
    <groupId>com.dameng</groupId>
    <artifactId>DmJdbcDriver18</artifactId>
    <version>8.1.3.140</version>
</dependency>

<!-- 人大金仓数据库驱动 -->
<dependency>
    <groupId>cn.com.kingbase</groupId>
    <artifactId>kingbase8</artifactId>
    <version>9.0.1</version>
</dependency>
```

#### 2. 数据源常量定义
在 `DataSourceConstant.java` 中添加了新的数据库类型常量：

```java
public class DataSourceConstant {
    public static final String MYSQL = "mysql";
    public static final String DERBY = "derby";
    public static final String DM = "dm";          // 达梦
    public static final String KINGBASE = "kingbase"; // 人大金仓
}
```

#### 3. 外部数据源属性增强
修改了 `ExternalDataSourceProperties.java`，支持从配置中动态获取 JDBC 驱动类名，而不是硬编码为 MySQL 驱动：

```java
// 原代码（硬编码 MySQL 驱动）
poolProperties.setDriverClassName(JDBC_DRIVER_NAME);

// 新代码（从配置获取）
poolProperties.setDriverClassName(getOrDefault(jdbcDriverName, index, jdbcDriverName.get(index)).trim());
```

#### 4. Mapper 实现类创建
为两种数据库分别创建了完整的 Mapper 实现类：

**达梦数据库 (dm包下)**：
- `AbstractMapperByDm.java` - 抽象基类
- `ConfigInfoMapperByDm.java` - 配置信息 Mapper
- `ConfigInfoBetaMapperByDm.java` - Beta配置 Mapper
- `ConfigInfoTagMapperByDm.java` - 配置标签 Mapper
- 等其他 10+ 个 Mapper 实现

**人大金仓数据库 (kingbase包下)**：
- `AbstractMapperByKingbase.java` - 抽象基类
- `ConfigInfoMapperByKingbase.java` - 配置信息 Mapper
- `ConfigInfoBetaMapperByKingbase.java` - Beta配置 Mapper
- `ConfigInfoTagMapperByKingbase.java` - 配置标签 Mapper
- 等其他 10+ 个 Mapper 实现

#### 5. 可信函数枚举
为两种数据库分别创建了函数枚举类，用于安全地处理 SQL 函数调用：

```java
// TrustedDmFunctionEnum.java - 达梦函数枚举
public enum TrustedDmFunctionEnum {
    NOW("NOW()", "NOW()");  // 目前只定义了 NOW() 函数
    // ... 其他代码
}

// TrustedKingbaseFunctionEnum.java - 人大金仓函数枚举
public enum TrustedKingbaseFunctionEnum {
    NOW("NOW()", "NOW()");  // 目前只定义了 NOW() 函数
    // ... 其他代码
}
```

## 关键代码段分析

### 1. 抽象基类实现
两种数据库的抽象基类结构一致，主要重写了 `getFunction()` 方法：

**AbstractMapperByDm.java:27-33**
```java
public abstract class AbstractMapperByDm extends AbstractMapper {
    @Override
    public String getFunction(String functionName) {
        return TrustedDmFunctionEnum.getFunctionByName(functionName);
    }
}
```

**AbstractMapperByKingbase.java:27-33**
```java
public abstract class AbstractMapperByKingbase extends AbstractMapper {
    @Override
    public String getFunction(String functionName) {
        return TrustedKingbaseFunctionEnum.getFunctionByName(functionName);
    }
}
```

### 2. Mapper 实现示例
以 `ConfigInfoMapperByDm.java` 为例，展示了达梦数据库的 SQL 实现：

**ConfigInfoMapperByDm.java:41-307**
```java
public class ConfigInfoMapperByDm extends AbstractMapperByDm implements ConfigInfoMapper {

    @Override
    public MapperResult findConfigInfoByAppFetchRows(MapperContext context) {
        final String appName = (String) context.getWhereParameter(FieldConstant.APP_NAME);
        final String tenantId = (String) context.getWhereParameter(FieldConstant.TENANT_ID);
        String sql = "SELECT id,data_id,group_id,tenant_id,app_name,content FROM config_info"
                + " WHERE tenant_id LIKE ? AND app_name= ?" + " LIMIT " + context.getStartRow() + ","
                + context.getPageSize();
        return new MapperResult(sql, CollectionUtils.list(tenantId, appName));
    }

    @Override
    public String getDataSource() {
        return DataSourceConstant.DM;  // 返回数据源标识
    }
}
```

### 3. 数据源标识方法
每个 Mapper 实现类都重写了 `getDataSource()` 方法，返回对应的数据源常量：

**达梦数据库返回：**
```java
@Override
public String getDataSource() {
    return DataSourceConstant.DM;
}
```

**人大金仓数据库返回：**
```java
@Override
public String getDataSource() {
    return DataSourceConstant.KINGBASE;
}
```

## 配置和使用方法

### 1. 数据库驱动配置
在 `application.properties` 中配置国产数据库连接：

```properties
# 达梦数据库配置示例
spring.datasource.driver-class-name=dm.jdbc.driver.DmDriver
spring.datasource.url=jdbc:dm://localhost:5236/nacos
spring.datasource.username=nacos
spring.datasource.password=nacos

# 人大金仓数据库配置示例
spring.datasource.driver-class-name=com.kingbase8.Driver
spring.datasource.url=jdbc:kingbase8://localhost:54321/nacos
spring.datasource.username=nacos
spring.datasource.password=nacos
```

### 2. 多数据源配置
通过 `db` 配置节支持多数据源：

```properties
db.num=1
db.url.0=jdbc:dm://localhost:5236/nacos
db.user.0=nacos
db.password.0=nacos
db.jdbcDriverName.0=dm.jdbc.driver.DmDriver
```

## 适配原理总结

### 1. 插件化架构利用
Nacos 原有的插件化数据源架构为国产化适配提供了良好基础：
- 通过 `Mapper` 接口定义统一的数据库操作规范
- 每种数据库实现自己的 Mapper 类
- 运行时根据配置动态选择对应的 Mapper 实现

### 2. SQL 兼容性处理
适配过程中主要处理了以下 SQL 差异：
- **分页语法**：使用 `LIMIT offset, size` 语法（与 MySQL 兼容）
- **函数调用**：通过 `TrustedFunctionEnum` 统一管理数据库函数
- **数据类型**：保持与原有 MySQL 表结构一致

### 3. 安全考虑
- 使用 `TrustedFunctionEnum` 枚举可信 SQL 函数，防止 SQL 注入
- 所有 SQL 语句通过参数化查询构建
- 继承现有的安全校验机制

### 4. 扩展性设计
- 新增的 `jdbcDriverName` 配置属性支持任意 JDBC 驱动
- 抽象基类设计便于未来支持更多国产数据库
- 常量定义集中管理，便于维护

## Docker 镜像打包和使用

### 1. Docker 镜像构建
基于提交 `a83d19890` 的改动，Nacos 现在支持通过 Docker 镜像方式部署。主要新增文件包括：

- **Dockerfile**: 重新设计的 Docker 镜像构建文件，支持环境变量配置
- **docker/docker-startup.sh**: Docker 容器启动脚本，支持集群和单机模式
- **docker/application.properties**: Docker 环境专用的配置文件
- **distribution/conf/application.properties**: 更新了数据库配置示例，包含达梦和人大金仓配置

### 2. Dockerfile 关键改动
新的 Dockerfile 主要改动包括：

```dockerfile
# 设置环境变量
ENV MODE="cluster" \
    PREFER_HOST_MODE="ip"\
    BASE_DIR="/home/nacos" \
    CLASSPATH=".:/home/nacos/conf:$CLASSPATH" \
    CLUSTER_CONF="/home/nacos/conf/cluster.conf" \
    FUNCTION_MODE="all" \
    JAVA_HOME="/usr/java/openjdk-17" \
    NACOS_USER="nacos" \
    JAVA="/usr/java/openjdk-17/bin/java" \
    JVM_XMS="1g" \
    JVM_XMX="1g" \
    JVM_XMN="512m" \
    JVM_MS="128m" \
    JVM_MMS="320m" \
    NACOS_DEBUG="n" \
    TOMCAT_ACCESSLOG_ENABLED="false" \
    TIME_ZONE="Asia/Shanghai"

# 下载并安装 Nacos
RUN set -x \
    && rm -rf /home/nacos/bin/* /home/nacos/conf/*.properties /home/nacos/conf/*.example /home/nacos/conf/nacos-mysql.sql \
    && ln -snf /usr/share/zoneinfo/$TIME_ZONE /etc/localtime && echo $TIME_ZONE > /etc/timezone

ADD docker/docker-startup.sh bin/docker-startup.sh
ADD docker/application.properties conf/application.properties

# 设置启动日志目录
RUN mkdir -p logs \
	&& touch logs/start.out \
	&& ln -sf /dev/stdout logs/start.out \
	&& ln -sf /dev/stderr logs/start.out \
    && chmod +x bin/docker-startup.sh

EXPOSE 8848
EXPOSE 9848 8080
ENTRYPOINT ["sh","bin/docker-startup.sh"]
```

### 3. 数据库配置支持
`distribution/conf/application.properties` 文件已更新，包含国产数据库配置示例：

```properties
### Connect URL of MySQL:
#db.url.0=
#db.user=
#db.password=
#db.pool.config.driverClassName=com.mysql.cj.jdbc.Driver

### Connect URL of Dm:
#db.url.0=
#db.user=
#db.password=
##db.pool.config.driverClassName=dm.jdbc.driver.DmDriver

### Connect URL of Kingbase:
#db.url.0=
#db.user=
#db.password=
#db.pool.config.driverClassName=com.kingbase8.Driver
```

### 4. Docker 容器启动命令
支持单机模式和集群模式部署：

```bash
# 单机模式启动示例
docker run --name nacos-standalone-derby \
    -e MODE=standalone \
    -e NACOS_AUTH_TOKEN=${your_nacos_auth_secret_token} \
    -e NACOS_AUTH_IDENTITY_KEY=${your_nacos_server_identity_key} \
    -e NACOS_AUTH_IDENTITY_VALUE=${your_nacos_server_identity_value} \
    -p 8080:8080 \
    -p 8848:8848 \
    -p 9848:9848 \
    -d nacos/nacos-server:latest

# 集群模式启动示例（需要配置数据库）
docker run --name nacos-cluster-mysql \
    -e MODE=cluster \
    -e SPRING_DATASOURCE_PLATFORM=mysql \
    -e MYSQL_SERVICE_HOST=your_mysql_host \
    -e MYSQL_SERVICE_PORT=3306 \
    -e MYSQL_SERVICE_DB_NAME=nacos \
    -e MYSQL_SERVICE_USER=nacos \
    -e MYSQL_SERVICE_PASSWORD=nacos \
    -e NACOS_AUTH_TOKEN=${your_nacos_auth_secret_token} \
    -e NACOS_AUTH_IDENTITY_KEY=${your_nacos_server_identity_key} \
    -e NACOS_AUTH_IDENTITY_VALUE=${your_nacos_server_identity_value} \
    -p 8848:8848 \
    -p 9848:9848 \
    -p 9555:9555 \
    -d nacos/nacos-server:latest
```

### 5. 环境变量配置
Docker 镜像支持以下关键环境变量：

| 环境变量 | 默认值 | 说明 |
|---------|--------|------|
| MODE | cluster | 运行模式：standalone（单机）或 cluster（集群） |
| NACOS_AUTH_TOKEN | 必填 | Base64 编码的认证密钥 |
| NACOS_AUTH_IDENTITY_KEY | 必填 | 服务器身份标识键 |
| NACOS_AUTH_IDENTITY_VALUE | 必填 | 服务器身份标识值 |
| JVM_XMS | 1g | JVM 初始堆大小 |
| JVM_XMX | 1g | JVM 最大堆大小 |
| TIME_ZONE | Asia/Shanghai | 容器时区 |

### 6. 构建自定义镜像
要构建包含国产数据库驱动的自定义镜像：

```bash
# 1. 构建 Nacos 项目
mvn -Prelease-nacos -DskipTests clean install -U

# 2. 构建 Docker 镜像
docker build -t nacos/nacos-server:custom .

# 3. 运行容器（使用达梦数据库示例）
docker run --name nacos-dm \
    -e MODE=standalone \
    -e SPRING_DATASOURCE_PLATFORM=dm \
    -e db.num=1 \
    -e db.url.0=jdbc:dm://localhost:5236/nacos \
    -e db.user.0=nacos \
    -e db.password.0=nacos \
    -e db.pool.config.driverClassName=dm.jdbc.driver.DmDriver \
    -e NACOS_AUTH_TOKEN=${your_token} \
    -e NACOS_AUTH_IDENTITY_KEY=${your_key} \
    -e NACOS_AUTH_IDENTITY_VALUE=${your_value} \
    -p 8848:8848 \
    -p 9848:9848 \
    -p 8080:8080 \
    -d nacos/nacos-server:custom
```

### 7. 注意事项
1. **认证配置**：生产环境必须设置 `NACOS_AUTH_TOKEN`、`NACOS_AUTH_IDENTITY_KEY` 和 `NACOS_AUTH_IDENTITY_VALUE`
2. **数据库驱动**：使用国产数据库时需要确保相应 JDBC 驱动已包含在构建中
3. **数据持久化**：建议挂载数据卷持久化配置数据
4. **网络配置**：集群模式需要正确配置网络和节点发现

## 测试状态
人大金仓数据库已经通过初步测试，达梦数据库的测试状态未明确说明。

## 后续建议
1. **函数扩展**：当前只定义了 `NOW()` 函数，根据实际使用情况需要扩展更多数据库函数
2. **方言支持**：考虑实现更完整的数据源方言支持
3. **性能优化**：针对国产数据库特性进行 SQL 优化
4. **文档完善**：补充详细的使用文档和故障排查指南

## 文件位置参考
- 达梦数据库适配代码：`plugin/datasource/src/main/java/com/alibaba/nacos/plugin/datasource/impl/dm/`
- 人大金仓数据库适配代码：`plugin/datasource/src/main/java/com/alibaba/nacos/plugin/datasource/impl/kingbase/`
- 数据源常量定义：`plugin/datasource/src/main/java/com/alibaba/nacos/plugin/datasource/constants/DataSourceConstant.java`
- 外部数据源属性：`persistence/src/main/java/com/alibaba/nacos/persistence/datasource/ExternalDataSourceProperties.java`
