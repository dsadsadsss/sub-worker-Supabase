# 节点自动上传聚合订阅管理系统 - 部署教程

## 📋 目录

- [系统简介](#系统简介)
- [功能特性](#功能特性)
- [准备工作](#准备工作)
- [部署步骤](#部署步骤)
- [配置说明](#配置说明)
- [使用指南](#使用指南)
- [常见问题](#常见问题)

---

## 系统简介

这是一个基于 Cloudflare Workers 和 Supabase 的节点订阅管理系统，支持节点自动上传、聚合管理、订阅生成等功能。

### 主要优势

- ✅ 完全免费部署（使用 Cloudflare Workers 免费套餐）
- ✅ 自动节点管理和过期清理
- ✅ 支持自定义节点和订阅链接聚合
- ✅ 关键词过滤功能
- ✅ 优选 IP 和端口自动替换
- ✅ 响应式 Web 管理界面

---

## 功能特性

### 1. 节点自动上传管理
- 支持节点自动上传和更新
- 自动清理过期节点
- 节点列表查看和单独删除

### 2. 自定义节点管理
- 手动添加自定义节点
- 支持订阅链接聚合
- 支持 v2ray 格式节点

### 3. 关键词过滤
- 支持添加关键词过滤规则
- 自动过滤包含关键词的节点
- 支持批量管理过滤规则

### 4. 订阅生成
- 自动生成订阅链接
- 支持优选 IP 和端口替换
- Base64 编码输出

---

## 准备工作

### 1. 注册 Cloudflare 账号
- 访问 [Cloudflare](https://dash.cloudflare.com/)
- 注册并登录账号

### 2. 注册 Supabase 账号
- 访问 [Supabase](https://supabase.com/)
- 注册并登录账号
- 创建新项目

### 3. 创建数据库表

在 Supabase 项目的 SQL Editor 中执行以下 SQL：

```
-- 创建URLs表（自动上传的节点）
CREATE TABLE IF NOT EXISTS urls (
    id BIGSERIAL PRIMARY KEY,
    url_name TEXT NOT NULL,
    url TEXT NOT NULL,
    last_update BIGINT NOT NULL,
    expiration_ttl INTEGER NOT NULL
);

-- 创建自定义节点表
CREATE TABLE IF NOT EXISTS sub2_urls (
    id BIGSERIAL PRIMARY KEY,
    url TEXT NOT NULL
);

-- 创建关键词过滤表
CREATE TABLE IF NOT EXISTS exclude_keywords (
    id BIGSERIAL PRIMARY KEY,
    keyword TEXT NOT NULL
);

-- 创建背景图片表
CREATE TABLE IF NOT EXISTS background_images (
    id BIGSERIAL PRIMARY KEY,
    image_url TEXT NOT NULL,
    created_at BIGINT DEFAULT EXTRACT(EPOCH FROM NOW())::BIGINT
);

-- 初始化所有表的函数
CREATE OR REPLACE FUNCTION init_tables()
RETURNS void AS $$
BEGIN
    CREATE TABLE IF NOT EXISTS urls (
        id BIGSERIAL PRIMARY KEY,
        url_name TEXT NOT NULL,
        url TEXT NOT NULL,
        last_update BIGINT NOT NULL,
        expiration_ttl INTEGER NOT NULL
    );

    CREATE TABLE IF NOT EXISTS sub2_urls (
        id BIGSERIAL PRIMARY KEY,
        url TEXT NOT NULL
    );

    CREATE TABLE IF NOT EXISTS exclude_keywords (
        id BIGSERIAL PRIMARY KEY,
        keyword TEXT NOT NULL
    );
    
    CREATE TABLE IF NOT EXISTS background_images (
        id BIGSERIAL PRIMARY KEY,
        image_url TEXT NOT NULL,
        created_at BIGINT DEFAULT EXTRACT(EPOCH FROM NOW())::BIGINT
    );
END;
$$ LANGUAGE plpgsql;

-- 删除过期URLs的函数
CREATE OR REPLACE FUNCTION delete_expired_urls(expired_time BIGINT)
RETURNS void AS $$
BEGIN
    DELETE FROM urls WHERE last_update < expired_time;
END;
$$ LANGUAGE plpgsql;

-- 删除所有URLs的函数
CREATE OR REPLACE FUNCTION delete_all_urls()
RETURNS void AS $$
BEGIN
    DELETE FROM urls;
END;
$$ LANGUAGE plpgsql;

-- 删除所有SUB2 URLs的函数
CREATE OR REPLACE FUNCTION delete_all_sub2_urls()
RETURNS void AS $$
BEGIN
    DELETE FROM sub2_urls;
END;
$$ LANGUAGE plpgsql;

-- 删除所有关键词的函数
CREATE OR REPLACE FUNCTION delete_all_keywords()
RETURNS void AS $$
BEGIN
    DELETE FROM exclude_keywords;
END;
$$ LANGUAGE plpgsql;

-- 删除所有背景图片的函数
CREATE OR REPLACE FUNCTION delete_all_background_images()
RETURNS void AS $$
BEGIN
    DELETE FROM background_images;
END;
$$ LANGUAGE plpgsql;
```
## 部署步骤

### 步骤 1: 获取 Supabase 配置信息

1. 登录 Supabase 控制台
2. 进入项目设置（Settings）
3. 点击 API 选项
4. 复制以下信息：
   - `Project URL`（SUPABASE_URL）
   - `anon public key`（SUPABASE_ANON_KEY）

### 步骤 2: 创建 Cloudflare Worker

1. 登录 Cloudflare 控制台
2. 进入 Workers & Pages
3. 点击「创建应用程序」
4. 选择「创建 Worker」
5. 给 Worker 命名（例如：sub-manager）
6. 点击「部署」

### 步骤 3: 配置 Worker 代码

1. 点击「编辑代码」
2. 删除默认代码
3. 复制完整的 worker.js 代码并粘贴
4. 点击「保存并部署」

### 步骤 4: 配置环境变量

1. 返回 Worker 设置页面
2. 点击「设置」标签
3. 找到「变量」部分
4. 添加以下环境变量：

| 变量名 | 类型 | 值 | 说明 |
|--------|------|-----|------|
| `PSWD` | 加密 | 你的登录密码 | 登录密码，建议使用强密码 |
| `SUPABASE_URL` | 文本 | 你的项目URL | 从 Supabase 复制 |
| `SUPABASE_ANON_KEY` | 加密 | 你的匿名密钥 | 从 Supabase 复制 |

5. 点击「保存」

### 步骤 5: 修改代码配置

在代码顶部的 `CONFIG` 对象中修改以下参数：

```javascript
const CONFIG = {
    UUID: 'your-uuid',              // 管理页面访问路径
    APITOKEN: 'your-api-token',     // API路径密钥（建议使用随机字符串）
    UP: 'your-upload-key',          // 节点上传密钥
    TOKEN: 'your-sub-token',        // 订阅地址密钥
    USER: 'admin',                   // 登录用户名
    DEFAULT_DOMAIN: '',              // 默认即可
    EXPIRATION_TIME: 1200,           // 链接有效期(秒)
    MAX_LOGIN_ATTEMPTS: 3,           // 最大登录尝试次数
    BACKGROUND_IMAGE: 'https://example.com/bg1.jpg;https://example.com/bg2.jpg' // 背景图片URL，多个用分号隔开
};
```

**安全建议：**
- `UUID`: 使用随机字符串，例如：`a1b2c3d4`
- `APITOKEN`: 使用长随机字符串，例如：`sk_live_1234567890abcdef`
- `UP`: 使用随机字符串，例如：`upload_xyz789`
- `TOKEN`: 使用随机字符串，例如：`sub_abc123`

### 步骤 6: 保存并部署

1. 修改完配置后，点击「保存并部署」
2. 等待部署完成

---

## 配置说明

### CONFIG 参数详解

| 参数 | 说明 | 示例 |
|------|------|------|
| `UUID` | 管理页面访问路径 | 访问地址为：`https://your-worker.workers.dev/sub-your-uuid` |
| `APITOKEN` | API路径密钥 | 用于保护管理接口，不会暴露给前端 |
| `UP` | 节点上传密钥 | 上传地址为：`https://your-worker.workers.dev/upload-your-upload-key` |
| `TOKEN` | 订阅地址密钥 | 订阅地址为：`https://your-worker.workers.dev/token=your-sub-token?cf_ip=...&cf_port=...` |
| `USER` | 登录用户名 | 管理页面登录用户名 |
| `EXPIRATION_TIME` | 节点有效期 | 单位：秒，超过此时间未更新的节点会被自动删除 |
| `MAX_LOGIN_ATTEMPTS` | 最大登录尝试次数 | 超过次数后需要等待一段时间 |
| `BACKGROUND_IMAGE` | 背景图片 | 多个图片用分号`;`隔开，系统会随机选择 |

## 使用指南

### 1. 访问管理页面

访问地址：`https://your-worker.workers.dev/sub-{你的UUID}`

例如：`https://my-sub.workers.dev/sub-a1b2c3d4`

### 2. 登录系统

- 用户名：CONFIG 中设置的 `USER` 值
- 密码：环境变量中设置的 `PSWD` 值

### 3. 首页信息

登录后可以看到：

#### 节点上传地址
```
https://your-worker.workers.dev/upload-{你的UP值}
```

**使用方法：**
```json
POST https://your-worker.workers.dev/upload-your-upload-key
Content-Type: application/json

{
  "URL_NAME": "node-name",
  "URL": "vmess://xxx..."
}
```

#### 订阅地址
```
https://your-worker.workers.dev/token={你的TOKEN值}?cf_ip=ip.sb&cf_port=443
```

**参数说明：**
- `cf_ip`: 优选域名（必填），会自动替换节点中的 `ip.sb` 或 `YOUXUAN_IP`
- `cf_port`: 优选端口（必填），会自动替换节点中的 `443` 或 `8443`

**使用示例：**
```
https://my-sub.workers.dev/token=sub_abc123?cf_ip=cf.example.com&cf_port=443
```

### 4. 自动上传节点管理

- **查看节点列表**：显示所有已上传的节点
- **刷新列表**：重新加载节点列表
- **删除单个节点**：点击节点右侧的「删除」按钮
- **全部删除**：清空所有上传的节点

### 5. 自定义节点管理

#### 添加节点
1. 在文本框中输入节点，每行一个
2. 支持格式：`协议://内容`
3. 点击「添加更新」

**示例：**
```
vmess://eyJhZGQiOiIxMjcuMC4w...
vless://uuid@domain:443?encryption=none...
https://example.com/subscription
```

#### 管理节点
- **查看列表**：显示所有自定义节点
- **刷新列表**：重新加载列表
- **删除单个**：点击节点右侧的「删除」按钮
- **全部删除**：清空所有自定义节点

### 6. 关键词过滤

#### 添加过滤规则
1. 在文本框中输入关键词，每行一个
2. 点击「添加更新」

**示例：**
```
过期
失效
测试
```

#### 管理规则
- **查看列表**：显示所有过滤关键词
- **刷新列表**：重新加载列表
- **删除单个**：点击关键词右侧的「删除」按钮
- **全部删除**：清空所有关键词

---

## 常见问题

### Q1: 为什么登录后看不到节点？

**A:** 请检查：
1. 是否成功上传过节点
2. 节点是否过期（超过 `EXPIRATION_TIME` 设置的时间）
3. 点击「刷新列表」按钮重新加载

### Q2: 订阅地址无法使用？

**A:** 请确保：
1. 订阅地址中包含 `cf_ip` 和 `cf_port` 参数
2. TOKEN 值正确
3. 至少有一个有效节点

### Q3: 如何上传节点？

**A:** 有两种方式：

**方式 1：通过 API 上传**
```bash
curl -X POST https://your-worker.workers.dev/upload-your-upload-key \
  -H "Content-Type: application/json" \
  -d '{
    "URL_NAME": "node1",
    "URL": "vmess://xxx..."
  }'
```

**方式 2：通过管理页面手动添加**
- 在「自定义节点或链接」标签页中添加

### Q4: 节点格式要求？

**A:** 支持以下格式：
- `vmess://` 开头的 VMess 节点
- `vless://` 开头的 VLESS 节点
- `http://` 或 `https://` 开头的订阅链接（会自动解析）

### Q5: 如何更换背景图片？

**A:** 修改 CONFIG 中的 `BACKGROUND_IMAGE`：
```javascript
BACKGROUND_IMAGE: 'https://example.com/bg1.jpg;https://example.com/bg2.jpg;https://example.com/bg3.jpg'
```

多个图片用分号 `;` 隔开，系统会随机选择一张显示。

### Q6: 忘记密码怎么办？

**A:** 
1. 登录 Cloudflare Workers 控制台
2. 进入你的 Worker 设置
3. 在「变量」部分修改 `PSWD` 的值
4. 保存后即可使用新密码登录

### Q7: 如何修改节点有效期？

**A:** 修改 CONFIG 中的 `EXPIRATION_TIME`：
```javascript
EXPIRATION_TIME: 1200,  // 单位：秒（1200秒 = 20分钟）
```

### Q8: 优选 IP 替换是如何工作的？

**A:** 
- 上传节点时，将优选域名设置为 `ip.sb`，端口设置为 `443` 或 `8443`
- 订阅时通过 `cf_ip` 和 `cf_port` 参数指定要替换的域名和端口
- 系统会自动替换节点中的优选域名和端口

**示例：**
```
原节点: vmess://...@ip.sb:443?...
订阅地址: ?cf_ip=cf.example.com&cf_port=2053
结果: vmess://...@cf.example.com:2053?...
```

### Q9: 如何备份数据？

**A:** 
1. 登录 Supabase 控制台
2. 进入你的项目
3. 使用 SQL Editor 导出数据：

```sql
-- 导出上传节点
SELECT * FROM urls;

-- 导出自定义节点
SELECT * FROM sub2_urls;

-- 导出过滤关键词
SELECT * FROM exclude_keywords;
```

### Q10: 如何迁移到新的 Worker？

**A:** 
1. 在新 Worker 中部署相同的代码
2. 配置相同的环境变量
3. 使用相同的 Supabase 项目
4. 数据会自动同步

---

## 高级配置

### 自定义域名

1. 在 Cloudflare Workers 设置中点击「触发器」
2. 点击「添加自定义域」
3. 输入你的域名（需要在 Cloudflare 托管）
4. 配置 DNS 记录

访问地址变为：`https://your-domain.com/sub-{UUID}`

### 速率限制

如果需要添加访问限制，可以在代码中添加：

```javascript
// 在 handleRequest 函数开头添加
const rateLimiter = {
  // 实现你的速率限制逻辑
};
```

### 日志记录

在 Supabase 中创建日志表：

```sql
CREATE TABLE access_logs (
    id BIGSERIAL PRIMARY KEY,
    path TEXT,
    ip TEXT,
    timestamp TIMESTAMP DEFAULT NOW()
);
```

---

## 安全建议

1. **使用强密码**：`PSWD` 应该使用复杂的随机字符串
2. **定期更换密钥**：定期更换 `APITOKEN`、`UP`、`TOKEN` 等密钥
3. **限制访问**：可以使用 Cloudflare 的访问规则限制访问来源
4. **启用日志**：记录访问日志以便审计
5. **备份数据**：定期备份 Supabase 数据库

---

## 技术支持

如果遇到问题：

1. 检查 Cloudflare Workers 的日志
2. 检查 Supabase 的数据库连接
3. 确认环境变量配置正确
4. 查看浏览器控制台的错误信息

---

## 更新日志

### v1.0.0
- ✅ 初始版本发布
- ✅ 支持节点自动上传管理
- ✅ 支持自定义节点管理
- ✅ 支持关键词过滤
- ✅ 支持优选 IP 替换
- ✅ 响应式管理界面
- ✅ 背景图片支持

---

## 许可证

版权所有 © 2025 FsMs

本项目仅供学习交流使用，请勿用于非法用途。

---

**祝你使用愉快！** 🎉
