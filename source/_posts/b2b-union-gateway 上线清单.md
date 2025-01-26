---
title: union-gateway清单
tags:
  - java
  - gateway
thumbnail: "../images/scene.jpg"
categories:
  - 技术集

---
------

## **1. 依赖包上传到镜像仓库**

确保以下依赖包已上传至镜像仓库并可用：

```xml
<!-- 网关注册 starter -->
<dependency>
    <groupId>com.gigacloudtech.b2b</groupId>
    <artifactId>b2b-union-gateway-spring-boot-starter</artifactId>
    <version>0.1.0</version>
</dependency>

<!-- 网关 API -->
<dependency>
    <groupId>com.gigacloudtech.b2b</groupId>
    <artifactId>b2b-union-gateway-api</artifactId>
    <version>0.1.0</version>
</dependency>

<!-- manage-auth 服务 API -->
<dependency>
    <groupId>com.gigacloudtech.b2b</groupId>
    <artifactId>b2b-manage-auth</artifactId>
    <version>0.0.2</version>
</dependency>
```

------

## **2. 数据库搭建**

### **2.1 创建数据库和 Schema**

- 数据库名称：`b2b_gateway`

### **2.2 数据库表创建及初始化**

#### **建表语句**

```sql
-- 网关服务配置表
CREATE TABLE gw_application_config (
    id           BIGINT NOT NULL COMMENT '主键' PRIMARY KEY,
    route_id     VARCHAR(50) NOT NULL COMMENT '服务名',
    context_path VARCHAR(50) DEFAULT NULL COMMENT '服务路径前缀',
    create_time  DATETIME DEFAULT CURRENT_TIMESTAMP NOT NULL COMMENT '创建时间',
    create_by    VARCHAR(50) DEFAULT NULL COMMENT '创建人',
    update_time  DATETIME DEFAULT CURRENT_TIMESTAMP NOT NULL ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
    update_by    VARCHAR(50) DEFAULT NULL COMMENT '更新人'
);

-- 网关路由权限表
CREATE TABLE gw_route_auth (
    id            BIGINT NOT NULL COMMENT '主键' PRIMARY KEY,
    config_id     BIGINT NOT NULL COMMENT '配置表 ID',
    req_method    VARCHAR(10) DEFAULT NULL COMMENT '请求方式：GET, HEAD, POST, PUT, PATCH, DELETE, OPTIONS, TRACE',
    req_first_url VARCHAR(50) DEFAULT '' NOT NULL COMMENT '请求地址第一路径',
    req_url       VARCHAR(200) NOT NULL COMMENT '访问的 URL',
    auth_key      VARCHAR(500) DEFAULT '' NOT NULL COMMENT '权限 key',
    create_by     VARCHAR(100) NOT NULL COMMENT '创建人',
    create_time   DATETIME DEFAULT CURRENT_TIMESTAMP NOT NULL COMMENT '创建时间',
    update_by     VARCHAR(20) NOT NULL COMMENT '更新人',
    update_time   DATETIME DEFAULT CURRENT_TIMESTAMP NOT NULL ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间'
) COMMENT '按钮权限配置表';

-- 创建索引
CREATE INDEX idx_route_id_req_first_url ON gw_route_auth (config_id, req_first_url);
```

#### **初始化数据**

```sql
INSERT INTO gw_application_config (id, route_id, context_path, create_time, create_by, update_time, update_by)
VALUES (6058514649606144, 'b2b-manage', '/b2bmanage', NOW(), 'admin', NOW(), 'admin');
```

------

## **3. 配置文件**

确保以下配置文件已正确配置并上传至相应环境：

- **`b2b-union-gateway-core.yaml`**
- **`b2b-union-gateway-web.yaml`**
- **`b2b-manage-auth.yaml`**

------

## **4. 代码合并**

- 将 `feature` 特性分支合并到指定的目标环境分支。
- 检查代码冲突，确保代码合并后的功能完整且可运行。

------

## **5. 运维部署**

### **5.1 k8S 集群部署**

按以下实例数部署服务：

| **服务名称**           | **实例数** |
| ---------------------- | ---------- |
| b2b-union-gateway-core | 3          |
| b2b-union-gateway-web  | 3          |
| b2b-manage-auth        | 3          |

### **5.2 日志接入**

- 接入 **ELK** 日志监控，确保网关日志实时可见。

### **5.3 前端代理地址修改**

- 修改前端 **manage-web** 的后端代理地址：

  **原地址**：`https://manage.gigab2b.com`
   **新地址**：`https://<网关域名>/outer`

------

## **6. 上线后检查**

### **6.1 核心功能检查**

- 登录功能是否正常。
- 各接口的请求是否返回预期结果。

### **6.2 性能与日志监控**

- 使用 ELK 检查服务日志是否存在异常。
- 监控服务性能，确保无超时或高延迟问题。

### **6.3 配置验证**

- 验证数据库初始化是否成功，数据是否完整。

------

## **7. 其他注意事项**

- 确保相关服务的环境变量、配置信息均已校验完毕。
- 业务团队确认测试通过后方可上线。

