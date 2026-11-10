# 多作者博客系统（Sakura Blog）

一个前后端分离的多作者博客系统。核心不是"能发文章"，而是**内容在公开之前先过审核**，以及**点赞数可被关系表验证**。

[在线演示](https://example.com) · [问题反馈](../../issues)

---

## 功能特性

### 内容与创作
- **Markdown 创作**：编辑/预览切换，支持标题、代码块、表格、引用、图片
- **草稿与发布**：草稿仅本人可见，不进社区时间线
- **自动目录**：文章详情页按 h1–h3 生成锚点目录
- **封面与标签**：上传封面（魔数校验）、多标签归类

### 发布即审核
- 审核**嵌在发布请求内**，不是独立后台功能
- 三态判定：`Pass` 直接上线 / `Review` 转待复核 / `Block` 拒绝
- 拒绝时返回**风险类别 + 字符区间**，前端高亮命中位置
- 审核服务超时、限流、未配密钥 → 统一降级为待复核，**不阻断创作**

### 互动
- **点赞幂等**：独立 `post_likes` 关系表 + `UNIQUE(post_id, user_id)`，重复点击不产生新记录
- **计数可校验**：`posts.likes` 冗余计数，与关系表同事务写入
- **两级评论**：回复挂载在父评论下，作者可删自己的评论，文章作者可删全部

### 社区与体验
- 时间线按「今天 / 昨天 / 本周 / 更早」分组
- 近 30 天高赞文章封面轮播
- 三形态检索：标题模糊 / 文章编号 / 粘贴链接自动解析
- 首页 3D 场景（Three.js，iframe 隔离渲染上下文）
- 环境音由 Web Audio 实时合成，不加载音频文件

---

## 技术栈

| 层 | 技术 |
|---|---|
| 前端 | Vue 3（组合式 API）、Vite、Vue Router 4、Axios |
| 渲染 | marked、highlight.js、Three.js |
| 后端 | Spring Boot 3.2、MyBatis-Plus、JJWT 0.12 |
| 数据库 | MySQL 5.7（utf8mb4）、InnoDB |
| 内容安全 | 腾讯云内容安全（TMS），TC3-HMAC-SHA256 手写签名 |

---

## 项目结构

```
.
├── blog-backend/                # Spring Boot 后端
│   └── src/main/java/com/blog/
│       ├── controller/          # AuthController、PostController、CommentController…
│       ├── service/             # 业务逻辑
│       ├── mapper/              # MyBatis-Plus Mapper
│       ├── config/              # JwtInterceptor、WebConfig、异常处理
│       ├── common/              # JwtUtil、MarkdownUtil、统一返回结构
│       └── entity/ dto/ vo/
├── blog-frontend/               # Vue 3 前端
│   └── src/
│       ├── views/               # 首页、文章、创作、社区、个人中心
│       ├── components/          # 通用组件（含 3D 场景、音频合成）
│       ├── api/                 # Axios 封装 + 请求拦截
│       ├── store/               # 登录态（authState）
│       └── router/              # 路由与权限守卫
└── blogdb.sql                   # 数据库结构与示例数据
```

---

## 快速开始

### 环境要求

- JDK 17+（Spring Boot 3.x 要求）
- Node.js 18+
- MySQL 5.7+ / 8.0
- Maven 3.8+

### 1. 初始化数据库

```bash
mysql -u root -p < blogdb.sql
```

会创建 `blogdb` 库和 4 张表（`users`、`posts`、`comments`、`post_likes`），并写入示例数据。

### 2. 配置后端

编辑 `blog-backend/src/main/resources/application.yml`：

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/blogdb?useUnicode=true&characterEncoding=utf8
    username: root
    password: 你的密码

jwt:
  secret: 至少32位的随机字符串
  expiration: 86400000        # 24 小时（毫秒）

tencent:
  tms:
    secret-id: ${TENCENT_TMS_SECRET_ID:}
    secret-key: ${TENCENT_TMS_SECRET_KEY:}
```

> 腾讯云密钥建议用环境变量注入，不要提交到仓库。未配置时审核走降级分支，文章进入待复核状态，系统仍可正常使用。

### 3. 启动后端

```bash
cd blog-backend
mvn spring-boot:run
```

默认监听 `8091`。

### 4. 启动前端

```bash
cd blog-frontend
npm install
npm run dev
```

访问 `http://localhost:3000`。开发环境通过 Vite 的 `proxy` 把 `/api` 转发到后端，生产环境由 Nginx 收敛到同一域名。

---

## 核心设计

### 为什么审核要嵌进发布请求

把审核做成独立后台功能，意味着数据库里会先出现"未经判定就公开"的内容，漏检只能事后追溯。本系统在写入之前完成判定：

```
提交 → 拼装待检文本 → 云端判定 → 映射状态 → 落库
                                  ├─ Pass     → published
                                  ├─ Review   → pending（待复核）
                                  └─ Block    → rejected（附类别与区间）
```

### 点赞为什么需要关系表

如果只有一个 `likes` 整型字段自增，会有两个问题：同一用户重复点击会重复计数；并发下读改写会丢更新。

本系统的做法：

- `post_likes` 表记录"谁赞了哪篇"，并在 `(post_id, user_id)` 上建联合唯一索引
- 重复插入由**数据库**拒绝，应用层捕获后按"已点赞"处理
- 计数用 `UPDATE ... SET likes = likes + 1`，由数据库保证原子性

---

## 主要接口

| 方法 | 路径 | 说明 | 需登录 |
|---|---|---|---|
| POST | `/api/auth/register` | 注册（成功即登录） | 否 |
| POST | `/api/auth/login` | 登录，返回 token | 否 |
| GET | `/api/posts` | 文章列表（分页） | 否 |
| GET | `/api/posts/{id}` | 文章详情 | 否 |
| POST | `/api/posts` | 发布文章（触发审核） | 是 |
| PUT | `/api/posts/{id}` | 编辑文章 | 是 |
| GET | `/api/posts/mine` | 我的文章 | 是 |
| POST | `/api/posts/{id}/like` | 点赞（幂等） | 是 |
| POST | `/api/comments` | 发表评论 | 是 |
| DELETE | `/api/comments/{id}` | 删除评论 | 是 |
| POST | `/api/upload` | 上传图片 | 是 |

统一返回结构：

```json
{
  "code": 0,
  "message": "ok",
  "data": { },
  "timestamp": 1762387200000
}
```

错误码按模块分段：`10xxx` 账号、`20xxx` 文章、`30xxx` 审核、`40xxx` 互动。业务失败一律返回 HTTP 200 并在 `code` 中体现，仅认证失败（401）与资源不存在（404）使用对应状态码。

---

## 已知限制

- **无人工复核界面**：`Review` 状态的内容需通过数据库操作处理
- **检索为子串匹配**：在前端已加载列表内过滤，分页外的内容搜不到
- **审核为同步调用**：响应慢时会拉长发布耗时（实测最慢 1.31s）
- **令牌无法主动撤销**：JWT 无状态，登出仅清前端存储，有效期内服务端仍认可
- **未做缓存层**：单机部署，高并发下无热点数据缓存

---

## 许可证

本项目为毕业设计作品，仅供学习参考。示例图片来自 picsum.photos，版权归原作者所有。

如需引用或二次开发，请注明出处。
