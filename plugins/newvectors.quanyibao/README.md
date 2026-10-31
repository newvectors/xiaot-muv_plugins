# 权益宝（newvectors.quanyibao）

对接 [权益宝](https://www.quanyibao.com) 的资源搜索插件：登录后用积分解锁影视资源，把 115 网盘分享链接交给 Muvyo 转存。

## 能力

| 能力 | 说明 |
|---|---|
| `search` + `resolve` | 搜片名 → 找到该影片的资源帖 → 需要时用积分购买 → 返回 115 分享链接与提取码 |
| `tasks` | 「每日积分签到」，默认每天一次，执行时间在插件设置的「定时任务」里改 |
| `account` | 设置页顶部显示账号与**剩余积分** |

## 配置

| 项 | 说明 |
|---|---|
| 账号 | 手机号写成 `86 13800138000`（区号 + 空格 + 号码），或直接填邮箱 |
| 密码 | 只存本机，用于自动登录换令牌 |
| 每次最多查询影视数 | 关键词搜索时取前几部作品去查资源帖，默认 3 |
| 每部最多网盘条目 | 一部作品最多交出几个网盘条目，默认 2 |
| 允许扣积分购买 | 关闭后只给已解锁的资源，不会花积分 |
| 优先 4K/1080P | 同一作品多个资源时，分辨率高的排前面 |

## 不会重复扣分

站点自身的购买接口没有资源级幂等：对已解锁的资源再次调用会**再扣一次积分**。

本插件在每次取链前都会重新读一次该资源帖的服务端状态（`auth.pageData` 里的 `allow`）：

- `allow = true`（已解锁）→ 直接取链，**不调用购买接口**；
- 未解锁 → 先比余额，够了才买，且只买一次；
- 买了但没拿到链接（超时、5xx、响应异常）→ 按 `uncertain` 上报，提示到站点核对，**不会自动重试**。

另外本地还留了一份已解锁记录（`mv.storage`），作为服务端状态读不到时的兜底。

## 接口要点（开发备忘）

- 查询走 REST GET，写操作走统一 RPC `POST /api/request {action, data}`；业务失败同样返回 HTTP 200，要看 body 里的 `statusCode` / `success` / `errorCode`。
- 按影片取资源的正确链路是 **`/api/film/catalog?ids=tv:1399`（批量，返回 `postId`）→ `auth.pageData` 读 `/article/<postId>`**。
  `/api/film/resources` 是「资源库浏览」接口，**不按影片过滤**，别拿它当单片资源列表。
- 搜索结果条目的主键字段是 `id`（tmdbId），`catalog` 里才叫 `tmdbId`；单条详情包在 `item` 下。
- 购买参数 `args` 必须是**字符串化 JSON**，`payment` 固定 `film_credit`。
- `loginData.module` 的 key 是随机串，只能遍历 `values` 按结构识别。

## 本地校验

```sh
# 静态：manifest / 权限 / 能力与函数对应（脚本来自 muvyo-addon-dev 技能）
python3 ~/.workbuddy/skills/muvyo-addon-dev/scripts/check_addon.py . --namespace newvectors

# 动态：按 action 分发的假 mv，跑 21 组用例（仓库内）
node ../../tools/qyb-addon-test/run.mjs .
```
