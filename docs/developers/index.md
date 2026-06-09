---
id: index
title: 跑通上云API Demo
sidebar_label: 跑通上云API Demo
sidebar_position: 1
description: SBIM 系统设备适配上云API的完整开发指南
---
# 跑通上云API Demo

> 这是一份**从 0 到 1**的动手指南：带你在本机/云主机上拉起依赖服务，导入示例数据，跑通上云API Demo，并为下一步的添加 **SuperDock** 设备的适配做好准备。

:::tip 你将获得

* 一套可直接运行的 Docker 化基础环境（EMQX、MySQL、Redis、MinIO、SRS）。
* 已导入的示例数据库，便于本地联调与回归。
* 明确的版本要求与常见坑位提示（端口、网络、权限）。
* SuperDock 对接的最佳实践入口（下文有直达链接）。

:::

## 为什么选择 DJI 上云 API

* **降低迁移成本**：接口统一、生态成熟；在兼容该接口后，客户可复用现有系统与经验，无需重新学习，大幅减少开发与培训成本。
* **提升系统兼容性**：通过标准化接口，SuperDock 可与 DJI 设备统一管理，支持混合部署，简化集成流程，保障稳定性与灵活性。

---

## 免责声明与使用须知

> 本文基于 **上云API Demo** 项目编写，仅供技术参考与学习。示例并不等于生产方案，请务必在上线前完成**安全评估与审计**。

* **技术参考**：文中示例代码与配置均为参考实现，来自并扩展自开源 Demo，不构成生产级方案。
* **安全提示**：Demo 可能存在安全隐患（如数据暴露、鉴权不足等）。不要直接暴露在公网；如需对外，请进行加固（网络隔离、最小权限、WAF、审计日志等）。
* **兼容性**：SuperDock 与 上云 API **Demo** 的兼容性基于当前版本验证；设备固件与 API 版本变更可能影响结果，正式部署前请做充分回归。
* **免责**：因参考本文造成的业务中断、数据损失、安全事件、第三方索赔、设备损坏等，**草莓创新及相关人员不承担责任**。

> 生产建议：隔离环境开发→专业安全审计→备份与恢复方案→监控与告警→持续更新补丁。

---

## 准备工作（Prerequisites）

* 一台 Linux 云服务器（建议 Ubuntu 20.04+，16.04 亦可）或本地 Linux/Mac 环境。
* **Docker** 与 **Docker Compose**

    * 安装教程：

        * Docker: [https://docs.docker.com/engine/install/ubuntu/](https://docs.docker.com/engine/install/ubuntu/)
        * Compose: [https://docs.docker.com/compose/install/](https://docs.docker.com/compose/install/)
* （源码编译场景）**Java 11+**、**Node.js 17.8**、**Nginx 1.20.2** 等。

:::caution 端口与网络

* 需占用端口：`1883/8883/18083`（EMQX）、`3306`（MySQL）、`6379`（Redis）、`9000/9001`（MinIO）、`1935/1985/8080/8000`（SRS）。
* Compose 默认创建 `192.168.6.0/24` 的 bridge 网络，请确保与现网无冲突。
  :::

---

## 获取示例包

* 直接下载打包资源（含镜像与示例文件）：

  **[下载源码包](https://terra-sz-hc1pro-cloudapi.oss-cn-shenzhen.aliyuncs.com/c0af9fe0d7eb4f35a8fe5b695e4d0b96/docker/cloud_api_sample_docker.zip)**

解压 `cloud_api_sample_docker.zip` 后，可见类似目录结构：

```text
root@server:~/djicloud/cloud_api_sample# ls
cloud_api_sample_docker_v1.10.0.tar  docker-compose.yml  update_backend.sh
data                                 source              update_front.sh
```

> 说明：目录名可能随版本略有差异；核心文件包括 `docker-compose.yml`、`data/`（数据持久化）、`source/`（源码与 SQL 位置）及预打包镜像 tar。

---

## 启动依赖服务（Docker Compose）

1）将 `docker-compose.yml` 替换为下述内容：

```yaml
version: "3"
services:
  emqx:
    image: emqx:4.4
    ports:
      - "18083:18083"
      - "1883:1883"
      - "8083:8083"
      - "8883:8883"
      - "8084:8084"
    environment:
      - EMQX_ALLOW_ANONYMOUS=true # 仅限本地/演示环境！
    hostname: emqx-broker
    networks:
      - cloud_service_bridge

  mysql:
    image: mysql:latest
    networks:
      - cloud_service_bridge
    ports:
      - "3306:3306"
    volumes:
      - /etc/localtime:/etc/localtime
      - ./data/mysql:/var/lib/mysql
    environment:
      - MYSQL_ROOT_PASSWORD=root
    hostname: cloud_api_sample_mysql

  redis:
    image: redis:6.2
    restart: always
    hostname: cloud_api_sample_redis
    ports:
      - "6379:6379"
    networks:
      - cloud_service_bridge
    command: ["redis-server"]

  minio:
    image: minio/minio:latest
    container_name: minio
    environment:
      - MINIO_ROOT_USER=minioadmin
      - MINIO_ROOT_PASSWORD=minioadmin
      - TZ=Asia/Shanghai
    ports:
      - "9000:9000"
      - "9001:9001"
    volumes:
      - ./data/minio:/data
      - ./data/minio/logs:/var/log/minio
      - /etc/localtime:/etc/localtime
    command: ["server", "/data", "--console-address", ":9001"]
    restart: always

  srs:
    image: ossrs/srs
    container_name: sd.srsdemo
    hostname: srsdemo
    ports:
      - "1935:1935"
      - "1985:1985"
      - "8080:8080"
      - "8000:8000"
      - "8000:8000/udp"
    environment:
      - CANDIDATE=【替换为服务器对外可达IP】
    command: ["objs/srs", "-c", "conf/rtmp2rtc.conf"]
    restart: always

networks:
  cloud_service_bridge:
    driver: bridge
    ipam:
      config:
        - subnet: 192.168.6.0/24
```

2）启动：

```bash
sudo docker compose up -d
```

3）验证服务是否就绪：

```bash
sudo docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}"
```

应能看到 `emqx` / `mysql` / `redis` / `minio` / `sd.srsdemo` 均为 `Up` 状态。

:::tip 常见检查点

* `CANDIDATE` 必须是 SRS 可被外网/设备访问到的 IP（或内网联调环境的真实可达 IP）。
* `EMQX_ALLOW_ANONYMOUS=true` 仅供开发阶段使用；生产务必关闭匿名并接入鉴权。
* 若端口被占用，先排查宿主机已有服务或调整映射端口。
  :::

---

## 导入示例数据库（cloud_sample.sql）

1）定位 SQL 文件：

```text
source/backend_service/sql/cloud_sample.sql
```

2）将数据导入到容器内的 MySQL：

```bash
sudo docker exec -i cloud_api_sample-mysql-1 \
  mysql -uroot -proot < source/backend_service/sql/cloud_sample.sql
```

> 如果当前目录不在项目根目录，请将 `cloud_sample.sql` 的路径替换为你的实际路径。

3）验证数据库是否创建成功：

```bash
sudo docker exec -it cloud_api_sample-mysql-1 \
  mysql -uroot -proot -e "SHOW DATABASES LIKE 'cloud_sample';"
```

正常输出示例：

```text
+-------------------------+
| Database (cloud_sample) |
+-------------------------+
| cloud_sample            |
+-------------------------+
```

---

## 源码获取与二次开发

Demo 采用前后端分离：前端 **TS + Vue3**，后端 **Java 11+ / Spring Boot**。建议先按上文跑通依赖，再拉取源码进行二次开发。

**前端需要**：TypeScript / HTML / CSS、Vue 3.x、Node.js（npm/yarn）、Ant Design Vue v2、HTTP/WebSocket、Nginx（用于部署）。

**后端需要**：Java 11+、Spring Boot、MQTT、MySQL、WebSocket、Redis。

**参考版本**：

* Linux：Ubuntu 20.04+
* Java：OpenJDK 11+
* MySQL：8.0.26
* EMQX：4.4.0
* Redis：6.2
* Nginx：1.20.2
* Vue：3.0.5
* Node.js：17.8

**技术支持与资源**：

* **翼飞鸿天官网**：https://www.sz-yfht.com/
* **上云 API 文档**：[https://developer.dji.com/doc/cloud-api-tutorial/cn/](https://developer.dji.com/doc/cloud-api-tutorial/cn/)
* **开源后端**：[https://github.com/dji-sdk/DJI-Cloud-API-Demo](https://github.com/dji-sdk/DJI-Cloud-API-Demo)
* **开源前端**：[https://github.com/dji-sdk/Cloud-API-Demo-Web](https://github.com/dji-sdk/Cloud-API-Demo-Web)

> 若需要构建自定义镜像，可参考仓库内脚本 `update_backend.sh` / `update_front.sh`。

---

## 日志查看

**常用日志位置与示例命令**：

```bash
# 应用日志
tail -f logs/cloud-api-demo.log

# MQTT 连接日志
grep "MQTT" logs/cloud-api-demo.log

# 设备注册日志
grep "thing.register" logs/cloud-api-demo.log

# 错误日志
grep "ERROR" logs/cloud-api-demo.log
```

---

## 下一步：对接 SuperDock 设备

* 如果你只想先跑通 Demo，到这里基本完成 ✅。
* 如果你要让 Demo 正确识别、管理 **SuperDock 系列机场**，请继续阅读下一篇：

👉 **[快速对接 SuperDock 设备](./superdock)**

---
