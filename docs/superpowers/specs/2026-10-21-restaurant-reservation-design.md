# 餐厅预约小程序设计

日期：2026-10-21  
状态：待审阅  
范围：第一版（一家餐厅的可装修首页 + 订桌）。点餐、支付、分店运营不在本版实现。

本文件描述一个新系统。yshop-drink 只作为业务参考，不在其 Java / Vue / UniApp 代码上继续开发。实现时新建独立工程。

## 1. 目标

顾客在微信小程序里上下滑动查看餐厅的长图和视频，并提交订桌。餐厅员工在电脑后台更换首页内容、处理预约。客户改图片、视频、文案、顺序和按钮时，只改后台数据，不改小程序代码，也不发版。

第一版只有一家店。所有业务数据带门店编号，以便以后增加同一品牌的分店。在线点餐和支付以后以模块方式加入，本版不实现。

## 2. 已确认的决定

- 采用方案 1：数据库保存页面 JSON，管理后台和小程序各自渲染。两端不共用界面组件。
- 装修是纵向排列的区块，不是自由画布，也不是低代码。
- 长图按原始比例撑满宽度，整页上下滑动，不裁成一屏。
- 员工登录支持微信扫码，并保留账号密码。登录方式可以继续增加。
- 员工权限使用标准 RBAC。顾客不进入这套角色。
- 后端 Rust（Axum + SQLx），管理后台 React + Vite + Ant Design，顾客端 Taro（React），数据库 MySQL 8，运行 MySQL 与 API 两个容器。

## 3. 系统边界

运行时两个容器：

| 容器 | 职责 |
|---|---|
| MySQL 8 | 全部业务数据 |
| Rust API | 接口、员工后台静态文件、素材文件 |

管理后台在开发时于本机由 Vite 启动，生产环境把构建结果交给 API 托管。微信小程序由 Taro 编译，在微信开发者工具中运行，不放入 Docker。

逻辑模块：

| 模块 | 职责 |
|---|---|
| 核心 | 配置、数据库、会话、RBAC、门店、素材、模块注册 |
| 页面 | 首页草稿、发布、历史版本 |
| 订桌 | 规则、餐次、时段名额、预约 |
| 管理后台 | 装修、预约处理、员工与角色 |
| 小程序 | 渲染已发布首页，提交和查询预约 |

一个业务模块向核心登记三件事：HTTP 路由、自有数据表、提供的区块类型。核心只挂载已启用模块。启用状态存在 `feature` 表。第一版启用 `page` 和 `reservation`。以后的点餐、支付用同样方式登记，并带来新的区块类型。这里没有安装包，也没有上传第三方代码的应用商店。

后台可以展示功能列表并切换已开发模块的启用状态。关闭模块后，对应路由不再挂载。页面模块自带的五种区块仍按文档渲染。订桌模块关闭时，订桌接口不存在，从入口进入固定订桌页后显示「订桌暂未开放」。以后由某个业务模块登记、并随该模块关闭而不再识别的区块，小程序跳过该区块。

## 4. 登录与权限

### 4.1 两类人

员工使用管理后台，走 RBAC。顾客使用小程序，第一版不登录，只在预约上留下姓名和电话。顾客以后的小程序微信登录使用另一个登录提供者，写入顾客表，不写入员工角色。

### 4.2 登录提供者

每种提供者只证明外部账号是谁。核心再用外部身份找到员工。

| 提供者编码 | 第一版 | 行为 |
|---|---|---|
| `password` | 启用 | 账号密码。用于初始化管理员，以及微信不可用时登录 |
| `wechat_qr` | 启用 | 微信开放平台「网站应用」扫码。管理后台的日常登录 |
| 以后的提供者 | 不实现 | 实现同一接口后即可注册 |

接口约定：

- `start` 返回短时票据和二维码地址，或返回密码登录不需要的空挑战。
- `complete` 用回调或密码换取 `provider + subject`。
- 微信的 `subject` 使用 unionid。开放平台必须绑定到能拿到 unionid 的应用。只拿到 openid 时登录失败，错误码 `WECHAT_UNBOUND` 不适用，使用 `VALIDATION`，说明「微信未返回统一标识」。

扫码流程：

1. `POST /admin/auth/wechat/start` 创建一次性票据，有效期 5 分钟，返回微信二维码地址。
2. 用户扫码后，微信回调 API。API 用授权码换取 unionid，票据标记已使用。
3. `external_identity` 中存在 `wechat_qr + unionid`，且员工在职，则创建会话。
4. 没有绑定时拒绝，不创建员工。错误码 `WECHAT_UNBOUND`。

密码登录校验密码摘要后创建同一张会话表中的会话。密码摘要使用 Argon2id。

会话令牌是随机 32 字节，数据库保存其哈希。管理后台用 `HttpOnly`、`SameSite=Lax` Cookie 携带。生产环境增加 `Secure`；本地 HTTP 不设置 `Secure`，否则浏览器不会带回 Cookie。每次需要权限的请求都读取会话和当前授权，不把权限写进令牌。退出、停用员工、超过有效期都使后续请求失败。会话有效期 12 小时。

密码登录的请求体是用户名和密码。初始化：员工表为空时，API 启动读取 `ADMIN_INITIAL_PASSWORD`。该变量缺失则进程退出。存在则创建用户名 `admin` 的管理员并授予系统角色「管理员」。该用户用密码登录后，可以在个人资料中绑定自己的微信。持有 `staff.manage` 的员工也可以为其他员工绑定或解除微信。微信回调成功或失败后，都重定向到同一 API 域名下的 `/login/result`。后台静态文件由该 API 托管，因此 Cookie 与后台同源。

### 4.3 RBAC

四类数据：

- 员工。
- 角色。一组权限的名字。
- 权限。固定编码，由模块在启动时写入，界面不能新建编码。
- 员工授权。员工、角色、门店。门店为空表示全品牌。

系统角色 `admin` 拥有全部已登记权限，不能删除，不能移除其权限。预设角色 `front_desk`（前台）拥有 `reservation.read`、`reservation.confirm`、`reservation.cancel`。

权限编码：

| 编码 | 能力 |
|---|---|
| `page.edit` | 修改首页草稿 |
| `page.publish` | 发布首页 |
| `media.upload` | 上传素材 |
| `media.delete` | 删除未被引用的素材 |
| `reservation.read` | 查看预约 |
| `reservation.confirm` | 确认、拒绝、标记到店 |
| `reservation.cancel` | 前台取消预约 |
| `booking_policy.manage` | 修改订桌规则和餐次 |
| `staff.manage` | 管理员工和微信绑定 |
| `role.manage` | 管理非系统角色及其权限 |
| `module.manage` | 启用或关闭模块 |

接口缺少会话时返回 `UNAUTHENTICATED`。会话有效但权限不足时返回 `FORBIDDEN`。

## 5. 页面装修

### 5.1 页面文档

第一版只有首页，页面键为 `home`。数据按多页面存放。时区为 `Asia/Shanghai`。

```json
{
  "schemaVersion": 1,
  "blocks": [
    {
      "id": "blk_01HZX3Y8QK9S0V7N2M4P6R8T1A",
      "type": "image",
      "visible": true,
      "props": {
        "mediaId": 12,
        "action": { "type": "reservation" }
      }
    }
  ]
}
```

`id` 由后台生成，为带 `blk_` 前缀的 ULID。`schemaVersion` 必须是 1。保存草稿时拒绝未知区块类型和缺失的必填属性，并且 `mediaId` 必须属于当前门店、类型必须与区块匹配。小程序渲染已发布页面时，跳过未知类型，其余区块照常显示。订桌模块关闭时，订桌入口区块仍按文档渲染；进入固定订桌页后，公开订桌接口不存在，页面显示「订桌暂未开放」。

区块类型：

| type | 必填 props | 顾客侧 |
|---|---|---|
| `image` | `mediaId`，素材必须是图片。`action` 可空 | 宽度 100%，高度按原始比例。可点按 |
| `video` | `mediaId`，素材必须是视频。`posterMediaId` 可空 | 宽度 100%，显示封面，点击播放 |
| `text` | `content`（1–500 字），`align` 为 `left` 或 `center` | 一段文字 |
| `button` | `label`（1–20 字），`action` | 一个按钮 |
| `reservationEntry` | `title`（1–40 字），`description`（0–200 字） | 入口，点按进入订桌流程 |

`action` 只有三种：`{ "type": "reservation" }`、`{ "type": "phone", "phone": "11位手机号" }`、`{ "type": "url", "url": "https 开头且长度不超过 500" }`。视频区块没有 `action`。

`visible` 为 false 的区块仍保存在文档中，小程序不渲染。

订桌的日期、时段、人数、姓名和电话是固定页面，不做成区块属性。

### 5.2 草稿与发布

`page` 一行同时保存 `draft_document` 和 `published_document`。`GET /public/home` 只返回已发布文档，并把 `mediaId` 展开成 `/media/{id}` 地址；尚未发布时返回空区块数组。后台读取草稿时返回原始文档，另附这些素材的地址，供预览使用。

`PUT /admin/pages/home` 只改草稿，需要 `page.edit`。`POST /admin/pages/home/publish` 在一个事务中把草稿复制到已发布列、写入 `published_at`，并向 `page_revision` 追加一份历史，需要 `page.publish`。发布空页面是允许的。

素材通过 `mediaId` 引用。草稿或已发布文档引用该素材时，删除返回 `MEDIA_IN_USE`。

### 5.3 素材

允许的图片：`image/jpeg`、`image/png`、`image/webp`，最大 10 MB。允许的视频：`video/mp4`，最大 50 MB。文件名由服务器生成，存放在 API 数据卷的 `media/` 目录。公开读取地址为 `/media/{id}`。

### 5.4 后台编辑器

编辑页上方是手机宽度的预览，使用与小程序相同的文档结构、独立的网页渲染器。下方是区块列表，支持添加、拖动排序、隐藏、删除和编辑选中区块的属性。图片和视频从已上传素材中选择。

## 6. 订桌

### 6.1 顾客流程

不登录。顺序为日期、时段、人数、姓名、电话、备注。成功后展示 6 位数字预约码。查询和顾客取消使用电话加预约码。

日期规则：

- 使用 `Asia/Shanghai` 的自然日。
- 只能选择今天到「今天 + advance_days」之间的日期。
- `closed_weekdays` 中的星期不可选。编码 1 为周一，7 为周日。

时段规则：

- 餐次给出当天的起止钟点。按 `slot_minutes` 切分，起点对齐餐次开始时间，末端不超过餐次结束时间。
- 开始时间早于当前时间的时段不返回。
- 开始时间距离现在小于 `min_lead_minutes` 的时段不返回。
- 剩余人数小于本次人数的时段仍返回，但标记为不可订，避免顾客误以为没有该时间。

人数必须在 `min_party` 与 `max_party` 之间。电话为中国大陆 11 位手机号。备注最长 200 字，可空。姓名 1–30 字。

预约码是密码学随机的 6 位数字，在同一门店内唯一，冲突时重新生成。同一 IP 对查询和取消合计每分钟最多 10 次，超过时返回 `VALIDATION`，说明「请稍后再试」。

### 6.2 名额

名额按人数，不记录桌号。`slot_capacity` 以门店、日期、开始时间唯一。提交预约的事务：

1. 按唯一键插入或锁住该时段行。
2. 再次核对日期、提前时间和人数规则。
3. `booked_covers + party_size <= max_covers_per_slot` 时，增加已占人数并插入预约。
4. 否则返回 `SLOT_FULL`，事务回滚。

待确认和已确认都占用人数。拒绝、前台取消、顾客取消时在事务中减少已占人数，且不得减成负数。确认和标记到店不改变已占人数。

`require_confirmation` 为 true 时新单状态为 `pending`，否则为 `confirmed`。

### 6.3 状态

| 状态 | 进入条件 | 顾客 | 前台 |
|---|---|---|---|
| `pending` | 需要人工确认时提交 | 开始前可取消 | 可确认、拒绝、取消 |
| `confirmed` | 自动确认，或前台确认待确认单 | 开始前可取消 | 可取消、标记到店 |
| `rejected` | 前台拒绝待确认单 | 只能查看 | 不能再改 |
| `cancelled` | 顾客或前台取消 | 只能查看 | 不能再改 |
| `arrived` | 前台把已确认标为到店 | 只能查看 | 不能再改 |

顾客取消的条件是状态为 `pending` 或 `confirmed`，且当前时间早于该档开始时间。前台取消不受开始时间限制，只要求状态仍是 `pending` 或 `confirmed`。拒绝只作用于 `pending`。标记到店只作用于 `confirmed`。非法迁移返回 `VALIDATION`。

### 6.4 默认规则

初始化的那一家店使用：可提前 14 天，最短提前 60 分钟，间隔 30 分钟，1–8 人，每档 40 人，需要人工确认，无固定休息日。餐次初始化为午市 11:30–14:00、晚市 17:30–21:00。没有启用餐次时，任何提交都返回 `SLOT_CLOSED`。

## 7. 数据表

业务表都有 `shop_id`，除非该表本身是全品牌的员工、角色或权限。预约状态的发生时间用 UTC。预约日期、时段开始时间、餐次钟点是 `Asia/Shanghai` 的民事时间，按日期和钟点分别存储，不做时区换算。

| 表 | 要点 |
|---|---|
| `shop` | 名称、电话、地址、是否营业。第一版插入一行 |
| `staff_user` | 用户名、姓名、手机、密码摘要、在职状态 |
| `external_identity` | `principal_type=staff`、员工编号、`provider`、`subject`。`(provider, subject)` 唯一 |
| `session` | 员工编号、令牌哈希、过期时间、作废时间 |
| `role` | 编码、名称、是否系统角色 |
| `permission` | 编码主键、所属模块、说明 |
| `role_permission` | 角色与权限 |
| `staff_role` | 员工、角色、可空的门店编号 |
| `media` | 门店、图片或视频、MIME、字节数、宽高或时长、存储键 |
| `page` | 门店、`page_key`、名称、草稿 JSON、已发布 JSON、发布时间。`(shop_id, page_key)` 唯一 |
| `page_revision` | 页面编号、文档 JSON、发布人、发布时间 |
| `booking_policy` | 一家店一行，存第 6.4 节的规则和 `closed_weekdays` JSON |
| `booking_period` | 门店、名称、开始钟点、结束钟点、是否启用 |
| `slot_capacity` | 门店、日期、开始时间、已占人数。三者唯一 |
| `reservation` | 预约码、日期、开始时间、人数、姓名、电话、备注、状态及各状态时间。`(shop_id, code)` 唯一 |
| `feature` | 模块编码、名称、是否启用 |

素材在未被草稿或已发布页面引用时物理删除，被引用时返回 `MEDIA_IN_USE`。员工只停用、不删行，以便保留操作人。预约不物理删除。

## 8. 接口

公开接口不带门店编号，服务器使用唯一的那家店。以后增加分店时，这些路径改为包含门店编号，小程序配置里写入要打开的店。

公开：

| 方法与路径 | 行为 |
|---|---|
| `GET /public/home` | 已发布首页。区块中的素材展开为可访问地址 |
| `GET /public/booking-options?date=YYYY-MM-DD` | 餐次切出的时段、剩余人数、是否可订 |
| `POST /public/reservations` | 创建预约，返回预约码和状态 |
| `POST /public/reservations/lookup` | 电话 + 预约码，返回这一笔 |
| `POST /public/reservations/cancel` | 电话 + 预约码，按顾客取消规则取消 |

管理接口都要会话和对应权限：

| 方法与路径 | 权限 |
|---|---|
| `POST /admin/auth/password` | 无 |
| `POST /admin/auth/wechat/start` | 无 |
| `GET /admin/auth/wechat/callback` | 无 |
| `POST /admin/auth/logout` | 已登录 |
| `POST /admin/me/wechat` | 已登录，只绑定当前员工自己的微信 |
| `GET/POST /admin/staff`、为他人绑定或解除微信 | `staff.manage` |
| `GET/PUT /admin/roles` | `role.manage` |
| `POST /admin/media`、`DELETE /admin/media/{id}` | `media.upload`、`media.delete` |
| `GET/PUT /admin/pages/home` | 读取需登录；写入需 `page.edit` |
| `POST /admin/pages/home/publish` | `page.publish` |
| `GET/PUT /admin/booking-policy`、餐次维护 | `booking_policy.manage` |
| `GET /admin/reservations?date=` | `reservation.read` |
| `POST /admin/reservations/{id}/confirm` | `reservation.confirm` |
| `POST /admin/reservations/{id}/reject` | `reservation.confirm` |
| `POST /admin/reservations/{id}/arrive` | `reservation.confirm` |
| `POST /admin/reservations/{id}/cancel` | `reservation.cancel` |
| `GET/PUT /admin/features` | 读取需登录；写入需 `module.manage` |

错误响应只有 JSON：`{ "code": "SLOT_FULL", "message": "这个时间已经订满，请换一个时间" }`。不返回堆栈、SQL 或内部路径。编码使用第 4、5、6 节已经出现的：`UNAUTHENTICATED`、`FORBIDDEN`、`VALIDATION`、`SLOT_FULL`、`SLOT_CLOSED`、`RESERVATION_NOT_FOUND`、`WECHAT_UNBOUND`、`MEDIA_IN_USE`。

## 9. 验证

使用真实 MySQL 做接口测试，微信提供者换成测试替身：

- 同一时段并发两笔预约，已占人数不超过上限，其中一笔得到 `SLOT_FULL`。
- 只保存草稿时，`GET /public/home` 仍是上一份已发布内容。
- 前台角色调用发布接口得到 `FORBIDDEN`。
- 已发布文档含未知区块类型时，公开接口省略该区块并返回其余区块。
- 顾客在时段开始后取消得到 `VALIDATION`，前台仍可取消。
- 未绑定的 unionid 扫码得到 `WECHAT_UNBOUND`，员工表不增加行。

管理后台和小程序的第一轮验收用手工路径：密码登录、绑定微信入口可见、上传长图、发布、公开接口看到新图、提交预约、前台确认、顾客用预约码查到「已确认」。

## 10. 第一版不做

桌位图、订金、短信、微信订阅消息、顾客账号、支付宝小程序和 H5 的单独工程、在线点餐、支付、多店切换界面、运行时安装插件。支付宝小程序和 H5 留在同一 Taro 工程的后续编译目标，不在本版交付。点餐和支付以后作为新模块加区块，不改首页渲染器的跳过规则。
