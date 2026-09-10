# inPARK 观光小火车购票小程序 · 技术开发方案

> 供 Claude Code 开发使用。本文档对应合同附件一《功能需求文档（第一阶段试运营版）》，
> 分成实现采用 **方案 B**（甲方商户号全额收款 + 月度分成报表，系统不做资金划转）。
> 开发周期目标：10 个工作日。范围外功能见文末「明确不做」清单，**不要提前实现**。

---

## 0. 总体架构

- **形态**:微信小程序（游客端 + 核销员界面同一小程序，按角色显示）+ Web 管理后台
- **后端**:微信云开发（TCB）——云函数 + 云数据库 + 云存储 + 静态网站托管
  - 选型理由:免运维、成本低（甲方直付腾讯云，低配约几百元/年）、原生支持微信支付与消息推送
- **管理后台**:Vue 3 + Vite 单页应用,部署在云开发静态托管;通过 HTTP 访问云函数(自定义 token 鉴权)
- **支付**:微信支付 JSAPI,通过云开发「云调用」对接商户号(免签名证书管理);退款走原路退回
- **一切资金只进甲方商户号,系统仅做记录与报表,不实现分账、转账、提现**

```
游客微信 ──► 小程序(游客端) ──► 云函数 ──► 云数据库
核销员   ──► 小程序(核销界面) ──┤
管理员   ──► Web后台(静态托管) ──► HTTP云函数(token鉴权)
微信支付回调 ──► pay-callback 云函数(幂等)
定时触发器 ──► monthly-report 云函数(每月1日)
```

## 1. 项目结构

```
/miniprogram          # 小程序端(原生小程序 + TypeScript)
  /pages
    /home             # 首页:信息展示
    /buy              # 选票下单
    /orders           # 我的订单列表
    /order-detail     # 订单详情 + 电子票二维码
    /refund           # 退票申请
    /verify           # 核销员:扫码核销(仅核销员角色可见)
    /verify-records   # 核销员:核销记录
  /utils
/cloudfunctions       # 云函数(Node.js 18)
  create-order        # 下单 + 统一下单
  pay-callback        # 支付回调(幂等,生成电子票)
  close-order         # 定时:关闭15分钟未支付订单
  apply-refund        # 游客提交退款申请
  process-refund      # 管理员审核退款 + 调微信退款 + 回调处理
  verify-ticket       # 核销(事务防重复)
  admin-api           # 后台聚合API(HTTP访问,token鉴权,内部按action路由)
  monthly-report      # 每月1日生成上月分成报表(定时触发器)
/admin-web            # 管理后台 Vue3 + Vite
  /src/views
    Login / Dashboard / TicketTypes / Orders / Refunds
    VerifyLogs / Reports / Content / Accounts
```

## 2. 数据库设计(云数据库集合)

所有集合权限设为「仅云函数可读写」,小程序端一律通过云函数访问。

### ticket_types 票种
```
{ _id, name, price(int,分), description, audience,   // 适用人群说明
  stock_total(int), stock_sold(int),                 // 库存;stock_total=-1 表示不限
  status: 'on'|'off', sort(int), created_at, updated_at }
```

### orders 订单
```
{ _id, order_no(唯一,业务单号), openid, phone?,
  items: [{ type_id, type_name, price, count }],     // 下单时快照票种与价格
  total_fee(int,分),
  status: 'pending'|'paid'|'closed'|'refunding'|'refunded'|'partial_refunded',
  wx_transaction_id?, paid_at?, created_at, expire_at // created_at+15min
}
```

### tickets 电子票(一单多票,每张独立)
```
{ _id, ticket_no(唯一,防伪票号), order_id, order_no, openid,
  type_id, type_name, price(int,分),
  status: 'unused'|'used'|'refunded',
  sign,                                  // HMAC签名,见4.3
  used_at?, verified_by?,                // 核销时间/核销员账号id
  refunded_at?, created_at }
```

### admins 后台与核销账号
```
{ _id, username(唯一), password_hash(bcrypt), role: 'admin'|'verifier',
  name, status: 'on'|'off', last_login_at, created_at }
```
- 核销员在小程序内用 username+password 登录换取 verifier token(存 storage)
- 管理员在 Web 后台登录换取 admin token;token 为 JWT,有效期 24h,密钥存云函数环境变量

### verify_logs 核销记录
```
{ _id, ticket_no, order_no, type_name, result: 'ok'|'reject',
  reject_reason?, verifier_id, verifier_name, created_at }
```

### refunds 退款单
```
{ _id, refund_no(唯一), order_no, ticket_ids[], amount(int,分),
  reason, status: 'applied'|'approved'|'rejected'|'success'|'failed',
  wx_refund_id?, operator_id?, applied_at, processed_at? }
```

### monthly_reports 月度分成报表(生成后锁定,只读)
```
{ _id, month: 'YYYY-MM',
  order_count, gross(int),               // 总流水(分)
  refund_count, refund_amount(int),
  cross_month_refund_amount(int),        // 其中:跨月退款(退款单对应订单支付月≠退款月)
  wx_fee(int),                           // 手续费 = round(gross*0.006) 口径,报表内注明为估算
  net(int),                              // gross - refund_amount - wx_fee
  park_share_by_gross(int),  operator_share_by_gross(int),   // 总流水口径 25%/75%
  park_share_by_net(int),    operator_share_by_net(int),     // 净流水口径 25%/75%
  locked: true, generated_at }
```

### site_content 运营内容(单文档)
```
{ _id:'main', park_name, intro, open_hours, address, phone,
  notice,               // 乘车须知
  purchase_agreement,   // 购票须知(下单前勾选)
  announcement?,        // 停运公告,非空则首页顶部横幅显示
  updated_at }
```

## 3. 核心流程实现要求

### 3.1 下单支付(create-order)
1. 校验票种在售、库存足够(`stock_total=-1` 跳过);校验单笔总张数 ≤ 10
2. 生成 order_no(`日期yyMMdd + 随机`),写入 orders(status=pending, expire_at=+15min)
3. **预占库存**:`stock_sold` 原子自增;订单关闭/退款时回滚
4. 云调用统一下单,返回小程序 `requestPayment` 参数
5. 价格一律用**整数分**计算,严禁浮点

### 3.2 支付回调(pay-callback)——必须幂等
1. 验证回调合法性;按 order_no 查订单
2. **幂等**:订单已是 paid 直接返回成功
3. 事务内:orders → paid,记 transaction_id/paid_at;按 items 逐张生成 tickets(status=unused)
4. 失败要让微信重试(返回非成功),不可吞错

### 3.3 关单(close-order,定时每分钟)
- 将 `status=pending 且 expire_at<now` 的订单置 closed,回滚库存

### 3.4 电子票与防伪(见 tickets.sign)
- `ticket_no`:16 位随机(加密随机源),不含易混淆字符
- `sign = HMAC-SHA256(ticket_no + order_no, SECRET).slice(0,16)`,SECRET 存环境变量
- 二维码内容:`INPARK|{ticket_no}|{sign}`,由小程序端绘制
- 订单详情页展示每张票的二维码,支持左右切换;屏幕常亮 `wx.setKeepScreenOn`

### 3.5 核销(verify-ticket)——防重复是验收红线
1. 校验 verifier token
2. 解析二维码,**先验 HMAC 签名**,不合法直接 reject(伪造票码)
3. **数据库事务**:读 ticket → 仅当 status=unused 时置 used,写 used_at/verified_by;
   任何其他状态返回 reject 及原因(已使用+首次核销时间 / 已退款 / 票不存在)
4. 无论成败写 verify_logs
5. 兜底:支持输入票号末 6 位查询候选票并核销(同一事务逻辑)
6. 响应目标 1 秒内;成功页绿色大字 + 票种数量,失败红色大字 + 原因

### 3.6 退款(apply-refund / process-refund)
- 游客仅可对 **unused** 票申请;整单或按张
- 管理员审核通过 → 云调用微信退款(原路退回),按张计算金额
- 退款成功回调:tickets → refunded,orders 状态按剩余票计算(refunded / partial_refunded),写 refunds、回滚库存
- 管理员可对未核销票主动发起退款(免游客申请),同样留痕

### 3.7 月度报表(monthly-report,定时每月1日 02:00)
- 统计口径见 monthly_reports 字段注释;金额单位分,报表展示转元(两位小数)
- **跨月退款**:退款发生月 ≠ 对应订单支付月的退款,计入退款发生月并单列
- 生成后 locked=true,后台仅可查看与导出,不提供任何修改接口
- 后台支持:按月查看、导出报表 Excel、导出当月订单明细 Excel(SheetJS 前端导出即可)
- 手工触发补生成(admin-api action,幂等:已存在则拒绝)

## 4. 管理后台(admin-web)功能清单

| 页面 | 功能 |
|---|---|
| Login | 账号密码登录,JWT 存 localStorage |
| Dashboard | 今日/本月:售票数、核销数、营收;近30天趋势(折线,ECharts) |
| TicketTypes | 票种增删改、上下架、库存设置;改价即时生效不影响已售订单(订单存快照) |
| Orders | 按时间/状态/手机号/票号查询;订单详情含每张票状态 |
| Refunds | 退款申请列表、审核通过/驳回、主动退款;操作留痕 |
| VerifyLogs | 全部核销记录,按日期/核销员筛选 |
| Reports | 月度分成报表(双口径并列展示)、Excel导出、订单明细导出 |
| Content | 首页信息、乘车须知、购票须知、停运公告编辑 |
| Accounts | 管理员/核销员账号创建、停用、重置密码(仅 admin 角色可见) |

- 敏感操作(退款、改价、账号管理)写操作日志集合 `op_logs { admin_id, action, detail, created_at }`

## 5. 非功能与安全要求

- 页面首屏 ≤ 2s;支付回调、核销 ≤ 1s;支持 200 并发购票(云函数默认弹性即可,注意库存原子操作)
- 所有集合「仅云函数可读写」;云函数内一律校验身份(openid / JWT)
- admin-api 对 HTTP 暴露:必须校验 JWT、按 role 鉴权、参数校验;登录接口做失败限频(同账号 5 次/10 分钟)
- 密码 bcrypt;JWT SECRET、票码 HMAC SECRET、微信支付配置均放云函数环境变量,**严禁硬编码**
- 手机号等个人信息仅存必要字段;日志不打印完整敏感信息
- 云数据库开启自动备份(控制台配置,文档中提醒即可)

## 6. 环境变量清单(部署时配置)

```
WX_MCH_ID            # 微信支付商户号
WX_PAY_SUB_KEY       # 支付相关密钥(按云调用实际要求)
TICKET_HMAC_SECRET   # 票码签名密钥(32+随机字符)
JWT_SECRET           # 后台token密钥
ENV_ID               # 云开发环境ID
```

## 7. 开发顺序建议(对齐10个工作日)

1. **D1–2**:云开发环境初始化、集合与权限、admins 种子账号;小程序骨架 + 首页/内容展示;admin-web 骨架 + 登录
2. **D3–5**:票种管理 → 下单/统一下单/支付回调/关单 → 电子票生成与展示(全链路真机跑通)
3. **D6–7**:核销(扫码 + 手输兜底 + 防重复) + 核销记录;订单查询
4. **D8**:退款全流程;Dashboard 统计
5. **D9**:月度报表 + 双口径 + Excel 导出;操作日志;内容管理
6. **D10**:整体联调、验收清单自测、部署文档

## 8. 验收自测清单(必须全绿)

- [ ] 选票→支付→生成电子票 全流程真机通过;15分钟未支付自动关单且库存回滚
- [ ] 一单多票逐张核销;同一票二次扫码被拦截并提示首次核销时间
- [ ] 伪造/篡改二维码内容被签名校验拦截
- [ ] 断网场景:手输票号末6位可完成核销
- [ ] 退款原路到账;已核销票不可退;跨月退款在报表中单列
- [ ] 报表双口径金额与订单明细手工核对一致;生成后无任何修改入口
- [ ] 改票价后,已售订单金额与电子票不受影响
- [ ] 核销员账号看不到后台管理功能;后台 token 过期后接口拒绝

## 9. 明确不做(合同附件一第六节,勿实现)

分时段预约/班次/限载、会员/次卡/储值、优惠券营销、微信原生分账(方案A)、
分销佣金提现、多网点、BI报表、发票、App/支付宝、复杂UI改版。
接到此类需求 → 停止,提示走合同变更流程。
