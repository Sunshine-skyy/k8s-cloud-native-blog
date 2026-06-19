# k8s-cloud-native-blog

基于 Kubernetes 的云原生博客系统设计与实现：从单体博客功能到多节点集群部署实践

## 项目简介

`k8s-cloud-native-blog` 是一个基于 Kubernetes 部署的云原生博客系统，采用前端、后端、数据库三层架构，围绕博客系统的典型场景完成了服务编排、持久化存储、服务暴露、健康检查和基础可观测性能力的实现。

本项目运行在已搭建好的 Kubernetes 集群之上，适合作为 Kubernetes 综合实践项目进行学习、复现和扩展。

## 项目来源

本项目参考并扩展了 [0xReLogic/kubernetes-for-students](https://github.com/0xReLogic/kubernetes-for-students) 中 Chapter 10 的 `Personal Blog Platform` 示例。

原项目主要面向学生学习场景，基于 `minikube` 演示 Kubernetes 基础对象如何协同工作，包括：

- Deployment
- Service
- ConfigMap
- Secret
- PVC
- Ingress

在此基础上，本项目结合真实 Kubernetes 集群环境进行了进一步改造和扩展。

## 相比原项目的主要改进

相较于 `kubernetes-for-students` 中的博客示例，本项目主要做了这些扩展：

- 从 `minikube` 教学场景迁移到真实多节点 Kubernetes 集群
- 结合本地环境适配 `kubectl v1.18.20`
- 增加 MySQL 持久化存储的手动 PV/PVC 配置
- 增加分页、搜索、分类、删除等博客功能
- 增加 `/health`、`/ready`、`/metrics` 接口
- 增加前端状态面板和基础可观测性展示
- 增加 NodePort 对外访问方案
- 提供可选 Ingress 配置

## 系统架构

本项目采用三层架构：

- 前端：`Nginx`
- 后端：`Node.js + Express`
- 数据库：`MySQL 8.0.41`

整体访问链路如下：

`Browser -> Nginx Frontend -> Blog API -> MySQL`

Kubernetes 资源关系如下：

- `blog-frontend`：负责提供前端页面，并将 `/api/` 代理到后端服务
- `blog-api`：负责提供博客 REST API，并连接 MySQL
- `blog-mysql`：负责存储博客文章数据
- `PersistentVolume / PersistentVolumeClaim`：负责 MySQL 数据持久化
- `ConfigMap`：负责后端配置注入
- `Secret`：负责数据库账号密码注入
- `Service`：负责服务发现和集群内通信
- `NodePort`：负责前端对外访问
- `Ingress`：提供可选的七层路由入口

## 当前已实现功能

根据当前仓库中的实际实现，系统已具备以下能力：

- 发布文章
- 查看文章列表
- 分页查询文章
- 关键词搜索文章
- 分类筛选文章
- 删除文章
- 文章摘要与全文切换
- 后端健康检查
- 后端就绪检查
- Prometheus 风格指标暴露
- 前端可视化展示 API、Ready、Metrics 状态

## 技术实现细节

### 1. 数据库层

数据库使用 `MySQL 8.0.41`：

- 通过 `mysql-secret.yaml` 注入数据库初始化参数
- 通过 `mysql-pv.yaml` 配置 `PersistentVolume` 和 `PersistentVolumeClaim`
- 通过 `mysql-deployment.yaml` 部署 MySQL 和对应 `Service`
- 配置了 MySQL 的 `livenessProbe` 和 `readinessProbe`

### 2. 后端 API 层

后端使用 `node:16-alpine`，在容器启动时动态创建并运行 `Express` 服务。

后端已实现：

- 自动初始化 `posts` 表
- 兼容 `category` 分类字段
- `GET /posts` 分页查询
- `GET /posts/search` 搜索文章
- `POST /posts` 发布文章
- `DELETE /posts/:id` 删除文章
- `GET /health` 健康检查
- `GET /ready` 就绪检查
- `GET /metrics` 指标暴露

同时，Pod 注解中已加入：

- `prometheus.io/scrape: "true"`
- `prometheus.io/port: "3000"`
- `prometheus.io/path: "/metrics"`

### 3. 前端层

前端使用 `nginx:alpine`，容器启动时动态生成单页博客界面，并通过 Nginx 反向代理将 `/api/` 转发到 `blog-api:3000`。

前端当前包含：

- 首页概览
- 状态看板
- 博客管理页
- 搜索框
- 分类筛选
- 分页切换
- 删除确认弹窗
- 文章摘要与全文切换
- 健康状态与指标状态展示

### 4. 服务暴露

前端服务当前使用：

- `NodePort: 30080`

因此可以通过以下方式访问：

`http://<NodeIP>:30080`

仓库中同时提供了可选 Ingress 配置：

- 文件：`blog-ingress.yaml`
- API 版本：`networking.k8s.io/v1beta1`
- Host：`blog.local`

## 仓库文件说明

当前准备公开的核心文件如下：

- `api-configmap.yaml`
- `api-deployment.yaml`
- `blog-ingress.yaml`
- `cleanup.sh.yaml`
- `deploy-blog.sh.yaml`
- `frontend-deployment.yaml`
- `mysql-deployment.yaml`
- `mysql-pv.yaml`
- `mysql-secret.yaml`
- `status.sh.yaml`

说明：

- 当前目录中的辅助脚本内容仍以 `.yaml` 后缀保存，例如 `deploy-blog.sh.yaml`、`status.sh.yaml`
- 本仓库将以你当前整理后的真实文件状态公开

## 环境说明

当前项目主要面向以下环境：

- 已提前搭建好的 Kubernetes 集群
- `kubectl` 版本：`v1.18.20`

兼容性说明：

- 由于集群版本较旧，部分配置使用旧版 API 写法
- `Ingress` 使用 `networking.k8s.io/v1beta1`
- 本项目更适合学习、实验和复现场景，正式生产环境建议进一步加强安全性和配置管理

## 基本部署思路

建议部署顺序如下：

1. 创建命名空间 `blog-platform`
2. 部署 `mysql-pv.yaml`
3. 部署 `mysql-secret.yaml`
4. 部署 `mysql-deployment.yaml`
5. 部署 `api-configmap.yaml`
6. 部署 `api-deployment.yaml`
7. 部署 `frontend-deployment.yaml`
8. 视情况部署 `blog-ingress.yaml`

部署完成后可重点检查：

- Pod 状态
- Service 状态
- PV/PVC 绑定状态
- 前端是否可通过 `NodePort 30080` 访问
- `/api/health`、`/api/ready`、`/api/metrics` 是否正常

## Blog Post

详细实现过程与问题排查文章整理中，后续补充。

## 开源说明

如果你正在学习以下内容，这个项目会比较适合作为参考：

- Kubernetes 基础对象如何组合成完整应用
- 如何在集群中部署前后端分离系统
- 如何为 MySQL 配置持久化存储
- 如何实现健康检查与基础监控接口
- 如何把教学示例扩展成更接近真实场景的项目

## 致谢

Thanks to [0xReLogic/kubernetes-for-students](https://github.com/0xReLogic/kubernetes-for-students) for the original teaching-oriented Kubernetes learning project and the Personal Blog Platform capstone idea.

This repository extends that idea into a more complete cloud-native blog deployment practice on a real Kubernetes cluster.