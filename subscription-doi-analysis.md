# Dittofeed 订阅分组与 DOI 流程分析报告

## 目录
1. [订阅分组架构概述](#1-订阅分组架构概述)
2. [订阅分组与渠道偏好管理](#2-订阅分组与渠道偏好管理)
3. [DOI 令牌生成机制](#3-doi-令牌生成机制)
4. [DOI 跨渠道校验流程](#4-doi-跨渠道校验流程)
5. [偏好写回路径](#5-偏好写回路径)
6. [完整流程串联](#6-完整流程串联)
7. [DOI 令牌安全与边界分析](#7-doi-令牌安全与边界分析)
8. [跨渠道校验异常分支时序链](#8-跨渠道校验异常分支时序链)
9. [关键代码位置](#9-关键代码位置)

---

## 1. 订阅分组架构概述

### 1.1 数据模型

Dittofeed 的订阅分组 (Subscription Groups) 是管理用户各渠道接收偏好的核心机制。每个订阅分组关联到特定的渠道（Email、SMS、MobilePush、Webhook），并支持两种类型：

- **OptIn（主动订阅）：用户必须显式订阅才能接收消息
- **OptOut（被动订阅）：用户默认接收，可选择退出

**数据库模型** (`packages/backend-lib/src/db/schema.ts:317-349`)：
```typescript
export const subscriptionGroup = pgTable(
  "SubscriptionGroup",
  {
    id: uuid().primaryKey().defaultRandom().notNull(),
    workspaceId: uuid().notNull(),
    name: text().notNull(),
    type: dbSubscriptionGroupType().notNull(),  // OptIn | OptOut
    channel: dbChannelType().notNull(),        // Email | SMS | MobilePush | Webhook
    createdAt: timestamp({ precision: 3, mode: "date" }).defaultNow().notNull(),
    updatedAt: timestamp({ precision: 3, mode: "date" }
      .defaultNow()
      .$onUpdate(() => new Date())
      .notNull(),
  },
  // ...
);
```

### 1.2 订阅分组与 Segments 的关系

每个订阅分组创建时会自动生成两个关联的 Segments（`packages/backend-lib/src/subscriptionGroups.ts:332-395`）：

1. **主 Segment**：命名格式 `subscriptionGroup-{id}
   - 用于标记用户是否在订阅分组内
   
2. **未订阅 Segment**：命名格式 `subscriptionGroup-unsubscribed-{id}`
   - 用于标记用户是否已从订阅分组退出

```typescript
// 命名函数定义
export function getSubscriptionGroupSegmentName(id: string) {
  return `subscriptionGroup-${id}`;
}

export function getSubscriptionGroupUnsubscribedSegmentName(id: string) {
  return `subscriptionGroup-unsubscribed-${id}`;
}
```

---

## 2. 订阅分组与渠道偏好管理

### 2.1 订阅状态判断逻辑

**核心函数** `inSubscriptionGroup` (`packages/backend-lib/src/subscriptionGroups.ts:83-91`)：

```typescript
export function inSubscriptionGroup(
  details: SubscriptionGroupDetails,
): boolean {
  // 在订阅分组 segment 尚未计算的情况下
  if (details.action === null && details.type === SubscriptionGroupType.OptIn) {
    return false;  // OptIn 默认不订阅
  }
  return details.action !== SubscriptionChange.Unsubscribe;
}
```

**判断规则**：
- **OptIn 类型**：
  - 未设置状态 (action = null) → 不订阅 (false)
  - 显式订阅 (action = Subscribe) → 订阅 (true)
  - 显式退订 (action = Unsubscribe) → 不订阅 (false)

- **OptOut 类型**：
  - 未设置状态 (action = null) → 默认订阅 (true)
  - 显式订阅 (action = Subscribe) → 订阅 (true)
  - 显式退订 (action = Unsubscribe) → 不订阅 (false)

### 2.2 消息发送时的订阅检查

在发送消息前，系统会检查用户的订阅状态 (`packages/backend-lib/src/messaging.ts:422-436`)：

```typescript
if (
  subscriptionGroupDetails &&
  !inSubscriptionGroup(subscriptionGroupDetails)
) {
  const { type: subscriptionGroupType, action: subscriptionGroupAction } =
    subscriptionGroupDetails;
  return err({
    type: InternalEventType.MessageSkipped,
    variant: {
      type: MessageSkippedType.SubscriptionState,
      action: subscriptionGroupAction,
      subscriptionGroupType,
    },
  });
}
```

如果用户不满足订阅条件，消息会被跳过，并记录 `MessageSkipped` 事件。

---

## 3. DOI 令牌生成机制

### 3.1 订阅密钥生成

工作空间初始化时会生成订阅密钥 (`packages/backend-lib/src/subscriptionGroups.ts:789-807`)：

```typescript
export async function upsertSubscriptionSecret({
  workspaceId,
}: {
  workspaceId: string;
}) {
  return insert({
    table: dbSecret,
    doNothingOnConflict: true,
    lookupExisting: and(
      eq(dbSecret.workspaceId, workspaceId),
      eq(dbSecret.name, SecretNames.Subscription),
    )!,
    values: {
      workspaceId,
      name: SecretNames.Subscription,
      value: generateSecureKey(8),  // 生成 16 字符的随机密钥
    },
  }).then(unwrap);
}
```

密钥生成使用 `crypto.randomBytes() 函数：
```typescript
// packages/backend-lib/src/crypto.ts:59-61
export function generateSecureKey(length = 32): string {
  return crypto.randomBytes(length).toString("hex");
}
```

### 3.2 订阅哈希生成

**哈希生成使用 HMAC-SHA256 算法 (`packages/backend-lib/src/subscriptionGroups.ts:426-451`)：

```typescript
export function generateSubscriptionHash({
  workspaceId,
  userId,
  identifierKey,
  identifier,
  subscriptionSecret,
}: {
  workspaceId: string;
  userId: string;
  identifierKey: string;  // 如 "email"
  identifier: string;     // 如 "user@example.com"
  subscriptionSecret: string;
}): string {
  const toHash = {
    u: userId,
    w: workspaceId,
    i: identifier,
    k: identifierKey,
  };

  const hash = generateSecureHash({
    key: subscriptionSecret,
    value: toHash,
  });
  return hash;
}
```

哈希算法实现 (`packages/backend-lib/src/crypto.ts:38-57`)：
```typescript
export function generateSecureHash({
  key,
  value,
}: {
  key: string;
  value: JSONValue;
}): string {
  const stringified = JSON.stringify(value);
  const hmac = crypto.createHmac("sha256", key);
  hmac.update(stringified);
  const hash = hmac.digest("hex");
  return hash;
}
```

### 3.3 订阅管理 URL 生成

**完整 URL 生成函数** (`packages/backend-lib/src/subscriptionGroups.ts:453-510`)：

```typescript
export function generateSubscriptionChangeUrl({
  workspaceId,
  subscriptionSecret,
  userId,
  identifier,
  identifierKey,
  changedSubscription,
  subscriptionChange,
  isPreview,
  showAllChannels,
}: {...}): string {
  const hash = generateSubscriptionHash({
    workspaceId,
    userId,
    identifierKey,
    identifier,
    subscriptionSecret,
  });

  const params: SubscriptionParams = {
    w: workspaceId,     // 工作空间 ID
    i: identifier,        // 标识符值（如邮箱、手机号）
    ik: identifierKey,  // 标识符类型（如 "email"、"phone"）
    h: hash,          // 验证哈希
  };
  
  // 可选参数
  if (changedSubscription) {
    params.s = changedSubscription;  // 订阅分组 ID
    params.sub =
      subscriptionChange === SubscriptionChange.Subscribe ? "1" : "0";  // 1=订阅，0=退订
  }
  if (isPreview) {
    params.isPreview = "true";
  }
  if (showAllChannels) {
    params.showAllChannels = "true";
  }
  
  // 构建完整 URL
  const url = new URL(config().apiBase || config().dashboardUrl);
  url.pathname = "/api/public/subscription-management/page";
  url.search = new URLSearchParams(params).toString();
  
  return url.toString();
}
```

**URL 参数说明**：
| 参数 | 说明 | 示例 |
|------|------|------|
| `w` | 工作空间 ID | `uuid-v4-string` |
| `i` | 标识符值 | `user@example.com` |
| `ik` | 标识符类型 | `email` |
| `h` | HMAC-SHA256 哈希 | `abc123...` |
| `s` | 订阅分组 ID（可选） | `subscription-group-uuid` |
| `sub` | 订阅操作（1=订阅，0=退订） | `1` 或 `0` |

### 3.4 Liquid 模板中的订阅链接

在消息模板中，通过 Liquid 标签生成订阅管理链接 (`packages/backend-lib/src/liquid.ts:80-200`)：

**退订链接标签**：
```typescript
liquidEngine.registerTag("unsubscribe_link", {
  parse(tagToken) {
    this.contents = tagToken.args;
  },
  render(scope) {
    const linkText: string = (this.contents as string) || "unsubscribe";
    const url = generateUnsubscribeUrl(scope);
    const href = url ? `href="${url}"` : "";
    return `<a class="df-unsubscribe" clicktracking=off ${href} target="_blank">${linkText}</a>`;
  },
});
```

**订阅管理链接标签**：
```typescript
liquidEngine.registerTag("subscription_management_link", {
  parse(tagToken) {
    this.contents = tagToken.args;
  },
  render(scope) {
    const linkText: string =
      (this.contents as string) || "manage subscriptions";
    const url = generateSubscriptionManagementUrl(scope);
    const href = url ? `href="${url}"` : "";
    return `<a class="df-subscription-management" clicktracking=off ${href} target="_blank">${linkText}</a>`;
  },
});
```

**注意**：链接添加了 `clicktracking=off 属性，防止 SendGrid 等邮件服务商的链接追踪干扰退订流程。

---

## 4. DOI 跨渠道校验流程

### 4.1 用户查找与哈希校验

**核心校验函数** `lookupUserForSubscriptions` (`packages/backend-lib/src/subscriptionGroups.ts:597-659`)：

```typescript
export async function lookupUserForSubscriptions({
  workspaceId,
  identifier,
  identifierKey,
  hash,
}: UserSubscriptionLookup): Promise<Result<{ userId: string }, Error>> {
  // 1. 并行获取订阅密钥和匹配的用户 ID
  const [subscriptionSecret, matchingUserIds] = await Promise.all([
    db().query.secret.findFirst({
      where: and(
        eq(dbSecret.workspaceId, workspaceId),
        eq(dbSecret.name, SecretNames.Subscription),
      ),
    }),
    findUserIdsByUserPropertyValue({
      workspaceId,
      userPropertyName: identifierKey,
      value: identifier,
    }),
  ]);

  // 2. 检查用户是否存在
  if (!matchingUserIds || matchingUserIds.length === 0) {
    logger().warn(
      {
        identifier,
        identifierKey,
      },
      "User not found",
    );
    return err(new Error("User not found"));
  }

  const secretValue = subscriptionSecret?.value;
  if (!secretValue) {
    throw new Error("Subscription secret not found");
  }

  // 3. 遍历所有匹配的用户，校验哈希
  const userId = matchingUserIds.find((uId) => {
    const generatedHash = generateSubscriptionHash({
      workspaceId,
      userId: uId,
      identifierKey,
      identifier,
      subscriptionSecret: secretValue,
    });
    return hash === generatedHash;
  });

  if (!userId) {
    logger().warn(
      {
        workspaceId,
        identifier,
        identifierKey,
        hash,
      },
      "Invalid hash",
    );
    return err(new Error("Invalid hash"));
  }
  return ok({ userId });
}
```

### 4.2 跨渠道标识符支持

**设计原理**：
- 标识符可以是任何用户属性（email、phone、managerEmail 等）
- 哈希绑定的是 `userId`，而不是特定渠道标识符
- 即使使用不同渠道的标识符，只要属于同一个用户，哈希校验就能通过

**实际测试案例** (`packages/backend-lib/src/subscriptionManagementEndToEnd.test.ts:254-510`)：
```typescript
// 用户有两个属性：
// - email: "user@example.com"
// - managerEmail: "manager@company.com"

// 发送给 manager 的消息中，使用 managerEmail 作为 identifierKey
// 生成的 unsubscribe 链接包含：
// - ik: "managerEmail"
// - i: "manager@company.com"
// - h: 基于 userId、workspaceId、identifierKey、identifier 生成的哈希

// 当 manager 点击退订链接时：
// 1. 系统根据 ik 和 i 查找用户
// 2. 找到 userId（因为 managerEmail 属性属于该用户）
// 3. 使用找到的 userId 重新计算哈希进行校验
// 4. 校验通过，执行退订操作
```

### 4.3 API 层校验

**PUT /user-subscriptions 端点校验 (`packages/api/src/controllers/subscriptionManagementController.ts:31-76`)：

```typescript
fastify.withTypeProvider<TypeBoxTypeProvider>().put(
  "/user-subscriptions",
  {
    schema: {
      description: "Allows users to manage their subscriptions.",
      body: UserSubscriptionsUpdate,
      response: {
        204: EmptyResponse,
        401: Type.Object({
          message: Type.String(),
        }),
      },
    },
  },
  async (request, reply) => {
    const { workspaceId, identifier, identifierKey, hash, changes } =
      request.body;

    // 校验用户身份
    const userLookupResult = await lookupUserForSubscriptions({
      workspaceId,
      identifier,
      identifierKey,
      hash,
    });

    if (userLookupResult.isErr()) {
      return reply.status(401).send({
        message: "Invalid user hash.",
      });
    }

    const { userId } = userLookupResult.value;

    // 更新订阅
    await updateUserSubscriptions({
      workspaceId,
      userUpdates: [
        {
          userId,
          changes,
        },
      ],
    });

    return reply.status(204).send();
  },
);
```

---

## 5. 偏好写回路径

### 5.1 订阅更新核心函数

**updateUserSubscriptions** (`packages/backend-lib/src/subscriptionGroups.ts:668-787`)：

```typescript
export async function updateUserSubscriptions({
  workspaceId,
  userUpdates,
}: {
  workspaceId: string;
  userUpdates: {
    userId: string;
    changes: UserSubscriptionsUpdate["changes"];  // Record<subscriptionGroupId, isSubscribed>
  }[];
}) {
  // 1. 获取所有相关订阅分组的 Segments
  const subscriptionGroupIds = userUpdates.flatMap((u) =>
    Object.keys(u.changes),
  );
  const segments = await db().query.segment.findMany({
    where: and(
      eq(dbSegment.workspaceId, workspaceId),
      inArray(dbSegment.subscriptionGroupId, subscriptionGroupIds),
    ),
  });

  // 2. 按订阅分组 ID 映射主 Segment 和未订阅 Segment
  const segmentsBySubscriptionGroupId = segments.reduce<
    Record<string, SegmentPair>
  >((acc, segment) => {
    // ... 映射逻辑
  }, {});

  // 3. 构建订阅变更事件
  const allUserEvents = userUpdates.flatMap(({ userId, changes }) => {
    const userChangePairs = R.entries(changes);
    const userEvents = userChangePairs.flatMap(
      ([subscriptionGroupId, isSubscribed]) =>
        buildSubscriptionChangeEvent({
          action: isSubscribed
            ? SubscriptionChange.Subscribe
            : SubscriptionChange.Unsubscribe,
          subscriptionGroupId,
          userId,
        }),
    );
    return userEvents;
  });

  // 4. 构建 Segment 赋值更新
  const segmentAssignmentUpdates: SegmentBulkUpsertItem[] = userUpdates.flatMap(
    ({ userId, changes }) => {
      const changePairs = R.entries(changes);
      return changePairs.flatMap(([subscriptionGroupId, isSubscribed]) => {
        const segmentPair = segmentsBySubscriptionGroupId[subscriptionGroupId];
        if (!segmentPair) {
          return [];
        }

        const assignments: SegmentBulkUpsertItem[] = [];

        // 主 Segment: inSegment = isSubscribed
        if (segmentPair.mainSegmentId) {
          assignments.push({
            workspaceId,
            userId,
            segmentId: segmentPair.mainSegmentId,
            inSegment: isSubscribed,
          });
        }

        // 未订阅 Segment: inSegment = !isSubscribed
        if (segmentPair.unsubscribedSegmentId) {
          assignments.push({
            workspaceId,
            userId,
            segmentId: segmentPair.unsubscribedSegmentId,
            inSegment: !isSubscribed,
          });
        }

        return assignments;
      });
    },
  );

  // 5. 并行写入 ClickHouse 和事件系统
  await Promise.all([
    insertSegmentAssignments(segmentAssignmentUpdates),
    insertUserEvents({
      workspaceId,
      userEvents: allUserEvents,
    }),
  ]);
}
```

### 5.2 Segment 赋值写入

**insertSegmentAssignments** (`packages/backend-lib/src/segments.ts:1023-1038`)：

```typescript
export async function insertSegmentAssignments(
  rawAssignments: SegmentBulkUpsertItem[],
) {
  const client = clickhouseClient();
  const assignments = rawAssignments.map((assignment) => ({
    workspace_id: assignment.workspaceId,
    type: "segment",
    user_id: assignment.userId,
    computed_property_id: assignment.segmentId,
    segment_value: assignment.inSegment,
  }));
  await client.insert({
    table: "computed_property_assignments_v2",
    values: assignments,
    format: "JSONEachRow",
    clickhouse_settings: { wait_end_of_query: 1 },
  });
}
```

### 5.3 订阅变更事件

**buildSubscriptionChangeEvent** (`packages/backend-lib/src/subscriptionGroups.ts:542-568`)：

```typescript
export function buildSubscriptionChangeEvent({
  messageId = uuid(),
  userId,
  action,
  subscriptionGroupId,
  currentTime = new Date(),
}: {
  userId: string;
  messageId?: string;
  subscriptionGroupId: string;
  currentTime?: Date;
  action: SubscriptionChange;
}): InsertUserEvent {
  const timestamp = currentTime.toISOString();
  return {
    messageId,
    messageRaw: JSON.stringify(
      buildSubscriptionChangeEventInner({
        userId,
        action,
        subscriptionGroupId,
        timestamp,
        messageId,
      }),
    ),
  };
}

function buildSubscriptionChangeEventInner({
  messageId,
  userId,
  action,
  subscriptionGroupId,
  timestamp,
}: {...}): {
  userId: string;
  timestamp: string;
  messageId: string;
} & SubscriptionChangeEvent {
  return {
    userId,
    timestamp,
    messageId,
    type: EventType.Track,
    event: InternalEventType.SubscriptionChange,
    properties: {
      subscriptionId: subscriptionGroupId,
      action,
    },
  };
}
```

---

## 6. 完整流程串联

### 6.1 完整 DOI 流程图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Double Opt-In (DOI) 完整流程                                │
└─────────────────────────────────────────────────────────────────────────────┘

  阶段 1: 消息发送与令牌生成
  ─────────────────────────────────

  [用户事件/广播触发]
         │
         ▼
  ┌─────────────────┐
  │ 生成消息模板渲染  │
  │ (Liquid 引擎)   │
  └────────┬────────┘
           │
           ├─► 提取用户属性 (userId, email, phone 等)
           ├─► 提取订阅分组信息
           └─► 获取订阅密钥 (SecretNames.Subscription)
           │
           ▼
  ┌─────────────────────┐
  │ 生成订阅哈希     │
  │ HMAC-SHA256     │
  │ {u, w, i, k}  │
  └────────┬────────┘
           │
           │ 生成 URL 参数:
           │ w={workspaceId}
           │ i={identifier}
           │ ik={identifierKey}
           │ h={hash}
           │ s={subscriptionGroupId}
           │ sub={1|0}
           │
           ▼
  ┌─────────────────┐
  │ 插入退订/管理链接 │
  │ {% unsubscribe_link %} │
  │ {% subscription_management_link %} │
  └────────┬────────┘
           │
           ▼
  [消息发送到用户渠道]
         │
         │ Email / SMS / MobilePush
         ▼


  阶段 2: 用户操作与令牌校验
  ─────────────────────────────────

  [用户点击退订链接]
         │
         ▼
  ┌─────────────────────┐
  │ GET /api/public/  │
  │ subscription-     │
  │ management/page   │
  └────────┬─────────┘
           │
           ├─► 提取 URL 参数
           │   w, i, ik, h, s, sub
           │
           ▼
  ┌─────────────────────┐
  │ 查找用户 (identifierKey + identifier)│
  └────────┬────────┘
           │
           │ 遍历匹配的用户 ID 列表
           │
           ▼
  ┌─────────────────────┐
  │ 哈希校验            │
  │ 重新计算 HMAC-SHA256 │
  │ 比对传入的 h 参数     │
  └────────┬────────┘
           │
           ├─► 校验通过 → 继续
           └─► 校验失败 → 返回 401 Unauthorized
           │
           ▼
  ┌─────────────────────┐
  │ 处理订阅变更 (s, sub 参数)│
  │ 如果 sub=0 → 退订    │
  │ 如果 sub=1 → 订阅    │
  └────────┬────────┘
           │
           ▼
  [渲染订阅管理页面]
         │
         ▼


  阶段 3: 用户偏好写回
  ─────────────────────────────────

  [用户在页面提交偏好]
         │
         ▼
  ┌─────────────────────┐
  │ POST /api/public/ │
  │ subscription-      │
  │ management/page   │
  └────────┬─────────┘
           │
           ├─► 提取表单数据
           │   w, h, i, ik
           │   sub_{sgId} = true/false
           │
           ▼
  ┌─────────────────────┐
  │ 二次哈希校验        │
  │ lookupUserForSubscriptions │
  └────────┬────────┘
           │
           ├─► 校验通过 → 继续
           └─► 校验失败 → 返回 401
           │
           ▼
  ┌─────────────────────┐
  │ updateUserSubscriptions │
  └────────┬────────┘
           │
           ├─► 构建 Segment 赋值
           │   主 Segment: inSegment = isSubscribed
           │   未订阅 Segment: inSegment = !isSubscribed
           │
           ├─► 构建订阅变更事件
           │   InternalEventType.SubscriptionChange
           │
           ▼
  ┌──────────────────────────────┐
  │ 并行写入                    │
  │ ├─► ClickHouse  │
  │ │   computed_property_assignments_v2 │
  │ │
  │ └─► 事件系统                 │
  │     DFSubscriptionChange │
  └──────────────────────────────┘
           │
           ▼
  [偏好持久化完成]
         │
         ▼
  阶段 4: 后续消息发送检查
  ─────────────────────────────────

  [下一次消息发送]
         │
         ▼
  ┌─────────────────────┐
  │ inSubscriptionGroup │
  │ 检查订阅状态        │
  └────────┬────────┘
           │
           ├─► 已订阅 → 发送消息
           └─► 未订阅 → 跳过 (MessageSkipped)
           │
           ▼
  [消息发送/跳过完成]
```

### 6.2 订阅管理页面交互流程

**GET 请求处理订阅变更 (`packages/api/src/controllers/subscriptionManagementController.ts:79-246`)：

```typescript
// GET /api/public/subscription-management/page

// 1. 解析查询参数
const {
  w: workspaceId,
  i: identifier,
  ik: identifierKey,
  h: hash,
  s: subscriptionGroupId,  // 订阅分组 ID
  sub,                   // 1=订阅, 0=退订
  isPreview,
} = request.query;

// 2. 用户查找与校验
const [userLookupResult, workspace] = await Promise.all([
  isPreview
    ? null
    : lookupUserForSubscriptions({...}),
  db().query.workspace.findFirst({...}),
]);

// 3. 如果提供了 s 和 sub 参数，执行订阅变更
if (subscriptionGroupId && sub) {
  subscriptionChange = sub === "1" ? Subscribe : Unsubscribe;
  
  if (!isPreview && targetSubscriptionGroup) {
    if (subscriptionChange === Unsubscribe) {
      // 退订该渠道的所有订阅分组
      const channelSubscriptionGroups = await db().query.subscriptionGroup.findMany({
        where: and(
          eq(dbSubscriptionGroup.workspaceId, workspaceId),
          eq(dbSubscriptionGroup.channel, targetSubscriptionGroup.channel),
        ),
      });
      
      const channelChanges: Record<string, boolean> = {};
      channelSubscriptionGroups.forEach((sg) => {
        channelChanges[sg.id] = false;
      });
      
      await updateUserSubscriptions({
        workspaceId,
        userUpdates: [{ userId, changes: channelChanges }],
      });
    } else {
      // 订阅指定的订阅分组
      await updateUserSubscriptions({
        workspaceId,
        userUpdates: [{
          userId,
          changes: { [subscriptionGroupId]: true },
        }],
      });
    }
  }
}

// 4. 获取用户当前订阅状态
const subscriptions = await getUserSubscriptions({ userId, workspaceId });

// 5. 渲染订阅管理页面
const html = await generateSubscriptionManagementPage({...});
return reply.type("text/html").send(html);
```

**POST 请求处理表单提交 (`packages/api/src/controllers/subscriptionManagementController.ts:249-353`)：

```typescript
// POST /api/public/subscription-management/page

// 1. 解析表单数据
const {
  w: workspaceId,
  h: hash,
  i: identifier,
  ik: identifierKey,
  isPreview,
  // 订阅复选框: sub_{subscriptionGroupId} = "true" 或不存在
} = typedBody;

// 2. 预览模式直接重定向
if (isPreview) {
  redirectParams.set("previewSubmitted", "true");
  return reply.redirect(302, `/api/public/subscription-management/page?${redirectParams}`);
}

// 3. 校验用户身份
const userLookupResult = await lookupUserForSubscriptions({...});
if (userLookupResult.isErr()) {
  return reply.status(401).send({ message: "Unauthorized" });
}

const { userId } = userLookupResult.value;

// 4. 获取所有订阅分组，构建变更对象
const subscriptionGroups = await db().query.subscriptionGroup.findMany({
  where: eq(dbSubscriptionGroup.workspaceId, workspaceId),
});

const changes: Record<string, boolean> = {};
for (const sg of subscriptionGroups) {
  const checkboxName = `sub_${sg.id}`;
  const isChecked = typedBody[checkboxName] === "true";
  changes[sg.id] = isChecked;
}

// 5. 执行更新
try {
  await updateUserSubscriptions({
    workspaceId,
    userUpdates: [{ userId, changes }],
  });
  redirectParams.set("success", "true");
} catch (error) {
  logger().error({ err: error }, "Failed to update subscriptions");
  redirectParams.set("error", "true");
}

// 6. 重定向回页面，显示结果
return reply.redirect(302, `/api/public/subscription-management/page?${redirectParams}`);
```

---

## 7. DOI 令牌安全与边界分析

### 7.1 令牌时效与一次性约束分析

#### 7.1.1 明确结论

**令牌是否有时效限制？**
- ❌ **无时效限制**。Dittofeed 当前的 DOI 令牌没有 TTL（Time-To-Live）或过期时间机制。

**令牌是否是一次性的？**
- ❌ **不是一次性的**。令牌可以被重复使用，没有"已使用"状态的追踪。

#### 7.1.2 证据分析

**1. 哈希生成不包含时间戳** (`packages/backend-lib/src/subscriptionGroups.ts:426-451`)：

```typescript
export function generateSubscriptionHash({
  workspaceId,
  userId,
  identifierKey,
  identifier,
  subscriptionSecret,
}: {...}): string {
  const toHash = {
    u: userId,        // 用户 ID
    w: workspaceId,   // 工作空间 ID
    i: identifier,    // 标识符值
    k: identifierKey, // 标识符类型
    // ❌ 没有 timestamp 或 expiry 字段
  };
  
  const hash = generateSecureHash({
    key: subscriptionSecret,
    value: toHash,
  });
  return hash;
}
```

**2. 校验时不检查历史记录** (`packages/backend-lib/src/subscriptionGroups.ts:597-659`)：

```typescript
export async function lookupUserForSubscriptions({...}): Promise<Result<{ userId: string }, Error>> {
  // 只验证哈希是否正确
  // ❌ 没有查询 TokenStore 或使用记录
  // ❌ 没有比较时间戳
  
  const userId = matchingUserIds.find((uId) => {
    const generatedHash = generateSubscriptionHash({...});
    return hash === generatedHash;  // 只要匹配就通过
  });
  
  if (!userId) {
    return err(new Error("Invalid hash"));
  }
  return ok({ userId });
}
```

**3. 密钥创建不会主动轮换** (`packages/backend-lib/src/subscriptionGroups.ts:789-807`)：

```typescript
export async function upsertSubscriptionSecret({
  workspaceId,
}: {
  workspaceId: string;
}) {
  return insert({
    table: dbSecret,
    doNothingOnConflict: true,  // ✅ 有冲突时不更新 = 不轮换
    lookupExisting: and(
      eq(dbSecret.workspaceId, workspaceId),
      eq(dbSecret.name, SecretNames.Subscription),
    )!,
    values: {
      workspaceId,
      name: SecretNames.Subscription,
      value: generateSecureKey(8),
    },
  }).then(unwrap);
}
```

#### 7.1.3 安全影响矩阵

| 特性 | 状态 | 实际影响 | 风险等级 |
|------|------|---------|---------|
| **有效期 (TTL)** | ❌ 无 | 链接永久有效，被泄露后可长期使用 | 高 |
| **一次性使用** | ❌ 无 | 链接可重复点击，反复修改订阅状态 | 中 |
| **非空性保护** | ✅ 有 | 必须提供有效的 identifier、identifierKey、hash | 低 |
| **哈希完整性** | ✅ 有 | HMAC-SHA256 保证未被篡改 | 低 |
| **密钥轮换** | ❌ 不支持 | 手动更新会立即失效所有历史链接 | 高 |

#### 7.1.4 实际风险场景

**场景 1：用户转发邮件给朋友**
```
用户 A 收到营销邮件
  ↓
用户 A 把邮件转发给朋友 B
  ↓
朋友 B 点击 {% unsubscribe_link %}
  ↓
系统校验通过（hash 绑定 userId，不是 email 所有者身份）
  ↓
朋友 B 可以退订用户 A 的所有订阅
```
- **风险等级**：高
- **原因**：URL 中不包含用户身份验证机制，只有哈希验证

**场景 2：邮件归档 3 年后**
```
2023 年发送的邮件被归档
  ↓
2026 年用户从归档中找回
  ↓
点击退订链接 → 仍然有效 ✓
  ↓
用户可以修改订阅状态
```
- **风险等级**：中（功能设计，非安全漏洞）
- **说明**：如果用户希望永久退订，这反而是一个优点

**场景 3：攻击者截获 URL**
```
攻击者通过中间人攻击截获邮件
  ↓
获取退订链接 URL
  ↓
攻击者可以无限次使用该 URL
  ↓
攻击者可以随时修改用户订阅偏好
```
- **风险等级**：高
- **后果**：攻击者可反复订阅/退订用户

---

### 7.2 链接被重放后系统实际行为

#### 7.2.1 重放攻击定义

**重放攻击**：攻击者获取到用户的退订链接后，重复点击或模拟请求，试图改变用户状态。

#### 7.2.2 GET 请求重放（点击退订链接）

**完整时间线分析**：

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
T0: 初始状态
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
系统状态：
  - 用户 A：订阅了 Marketing Emails
  - ClickHouse: inSegment = true
  - 未订阅 Segment: inSegment = false

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
T1: 用户第一次点击退订链接（正常操作）
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
GET /api/public/subscription-management/page
  ?w={workspaceId}
  &i=user@example.com
  &ik=email
  &h={valid_hash}
  &s={subscriptionGroupId}
  &sub=0  ← 退订

执行流程：
  1. lookupUserForSubscriptions() → 校验通过 ✓
  2. 找到 subscriptionGroup，确定 channel = Email
  3. 退订该 channel 的所有订阅分组
  4. updateUserSubscriptions() 执行：
     - 写入 ClickHouse: inSegment = false
     - 记录 DFSubscriptionChange 事件 (action: Unsubscribe)
  5. 渲染页面，显示"已退订"

系统状态变化：
  - 用户 A：已退订 Marketing Emails
  - ClickHouse: inSegment = false
  - 未订阅 Segment: inSegment = true
  - 事件日志：新增 1 条 DFSubscriptionChange

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
T2: 攻击者第一次重放 GET 请求
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
GET /api/public/subscription-management/page（相同 URL）

执行流程：
  1. lookupUserForSubscriptions() → 校验通过 ✓
     （因为 hash 仍然有效，没有使用记录）
  2. 找到 subscriptionGroup，确定 channel = Email
  3. 退订该 channel 的所有订阅分组
  4. updateUserSubscriptions() 再次执行：
     - 写入 ClickHouse: inSegment = false（幂等，值不变）
     - 再次记录 DFSubscriptionChange 事件 (action: Unsubscribe)
  5. 渲染页面，显示"已退订"

系统状态变化：
  - 用户 A：已退订（状态不变）
  - ClickHouse: inSegment = false（值不变）
  - 未订阅 Segment: inSegment = true（值不变）
  - 事件日志：新增第 2 条 DFSubscriptionChange ← ⚠️ 重复事件

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
T3: 用户通过其他途径重新订阅
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
用户访问订阅管理页面，手动勾选 Marketing Emails

系统状态变化：
  - 用户 A：已重新订阅
  - ClickHouse: inSegment = true
  - 未订阅 Segment: inSegment = false
  - 事件日志：新增 DFSubscriptionChange (action: Subscribe)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
T4: 攻击者第二次重放 GET 请求
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
GET /api/public/subscription-management/page（相同 URL）

执行流程：
  1. lookupUserForSubscriptions() → 校验通过 ✓
  2. 找到 subscriptionGroup，确定 channel = Email
  3. 退订该 channel 的所有订阅分组
  4. updateUserSubscriptions() 执行：
     - 写入 ClickHouse: inSegment = false ← ⚠️ 用户被再次退订！
     - 记录 DFSubscriptionChange 事件 (action: Unsubscribe)

系统状态变化：
  - 用户 A：再次被退订 ← ⚠️ 非预期状态变更
  - ClickHouse: inSegment = false
  - 未订阅 Segment: inSegment = true
  - 事件日志：新增第 3 条 DFSubscriptionChange
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**GET 重放的实际行为总结**：

| 时机 | 系统行为 | 用户状态 | 事件日志 |
|------|---------|---------|---------|
| 用户退订后立即重放 | 退订操作幂等 | 不变 | 重复事件 |
| 用户重新订阅后重放 | 再次退订 | 被改变 ⚠️ | 新增事件 |

#### 7.2.3 POST 请求重放（提交偏好表单）

**POST 请求的特殊性**：

```typescript
// packages/api/src/controllers/subscriptionManagementController.ts:249-353
// POST /api/public/subscription-management/page

// 构建 changes 对象从表单数据
const changes: Record<string, boolean> = {};
for (const sg of subscriptionGroups) {
  const checkboxName = `sub_${sg.id}`;
  const isChecked = typedBody[checkboxName] === "true";
  changes[sg.id] = isChecked;  // ← 使用提交时的表单快照
}

// 使用提交时的表单数据，而非当前状态
await updateUserSubscriptions({
  workspaceId,
  userUpdates: [{ userId, changes }],
});
```

**POST 重放完整时间线**：

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
T0: 初始状态
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
订阅分组状态：
  - SG1 (Marketing): 已订阅 ✓
  - SG2 (Newsletter): 已订阅 ✓
  - SG3 (Promotions): 已退订 ✗

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
T1: 用户提交表单（攻击者截获请求）
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
POST /api/public/subscription-management/page
Form Data:
  w={workspaceId}
  h={valid_hash}
  i=user@example.com
  ik=email
  sub_SG1=true
  sub_SG2=true
  sub_SG3=false  ← 用户提交时的选择

changes = { SG1: true, SG2: true, SG3: false }

系统状态变化：
  - SG1: 已订阅 ✓
  - SG2: 已订阅 ✓
  - SG3: 已退订 ✗

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
T2: 用户通过 API 退订 SG1
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
PUT /api/public/subscription-management/user-subscriptions
{
  workspaceId: "...",
  identifier: "user@example.com",
  identifierKey: "email",
  hash: "...",
  changes: { SG1: false }  ← 用户退订 SG1
}

系统状态变化：
  - SG1: 已退订 ✗  ← 变更
  - SG2: 已订阅 ✓
  - SG3: 已退订 ✗

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
T3: 攻击者重放 POST 请求
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
POST /api/public/subscription-management/page（相同表单数据）

Form Data 中的 changes = { SG1: true, SG2: true, SG3: false }

系统状态变化：
  - SG1: 已订阅 ✓  ← ⚠️ 被重新订阅！
  - SG2: 已订阅 ✓
  - SG3: 已退订 ✗

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**POST 重放的风险等级**：高
- **原因**：使用表单提交时的快照，而非当前状态
- **后果**：攻击者可将已退订的分组重新订阅

#### 7.2.4 事件日志特征

每次重放都会产生新的 `DFSubscriptionChange` 事件：

```typescript
// packages/backend-lib/src/subscriptionGroups.ts:730-743
const allUserEvents = userUpdates.flatMap(({ userId, changes }) => {
  const userChangePairs = R.entries(changes);
  const userEvents = userChangePairs.flatMap(
    ([subscriptionGroupId, isSubscribed]) =>
      buildSubscriptionChangeEvent({
        action: isSubscribed
          ? SubscriptionChange.Subscribe
          : SubscriptionChange.Unsubscribe,
        subscriptionGroupId,
        userId,
      }),
  );
  return userEvents;
});

await Promise.all([
  insertSegmentAssignments(segmentAssignmentUpdates),
  insertUserEvents({
    workspaceId,
    userEvents: allUserEvents,  // ← 每次都插入新事件
  }),
]);
```

**重放检测指标**：
- 监控短时间内同一用户同一订阅分组的多次 `DFSubscriptionChange` 事件
- 监控 `Unsubscribe → Subscribe → Unsubscribe` 这样的快速翻转序列

#### 7.2.5 幂等性对比

| 操作类型 | GET 退订 | GET 订阅 | POST 表单 |
|---------|---------|---------|----------|
| **幂等性** | ✅ 幂等（重复退订=退订） | ⚠️ 条件幂等（重复订阅=订阅） | ❌ 非幂等 |
| **状态依据** | URL 中的 sub 参数 | URL 中的 sub 参数 | 表单提交时的快照 |
| **重放影响** | 状态不变，产生重复事件 | 状态不变，产生重复事件 | 可能导致非预期状态变更 |
| **风险等级** | 低 | 低 | 高 |

---

### 7.3 订阅密钥轮换后旧链接的校验与失效

#### 7.3.1 当前密钥管理机制

**密钥创建流程**：

```
工作空间初始化
  ↓
bootstrapPostgres() 调用
  ↓
upsertSubscriptionSecret({ workspaceId })
  ↓
查询是否已有 SecretNames.Subscription
  ├─► 存在 → doNothingOnConflict → 保持不变
  └─► 不存在 → 生成新密钥 generateSecureKey(8)
                ↓
              插入 Secret 表
```

**代码证据** (`packages/backend-lib/src/subscriptionGroups.ts:789-807`)：
```typescript
export async function upsertSubscriptionSecret({
  workspaceId,
}: {
  workspaceId: string;
}) {
  return insert({
    table: dbSecret,
    doNothingOnConflict: true,  // ← 关键点：有冲突不更新
    lookupExisting: and(
      eq(dbSecret.workspaceId, workspaceId),
      eq(dbSecret.name, SecretNames.Subscription),
    )!,
    values: {
      workspaceId,
      name: SecretNames.Subscription,
      value: generateSecureKey(8),  // 16 字符随机十六进制
    },
  }).then(unwrap);
}
```

**结论**：当前系统**没有**内置的密钥轮换机制。

#### 7.3.2 密钥轮换后旧链接失效的根本原因

**哈希计算原理**：

```
哈希 = HMAC-SHA256(密钥, {userId, workspaceId, identifier, identifierKey})
```

**密钥轮换后的校验失败流程**：

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
阶段 1：原链接生成时（使用旧密钥）
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

key_old = "abc123_old_secret"

hash_old = HMAC-SHA256(key_old, {
  u: "user-123",
  w: "workspace-456",
  i: "user@example.com",
  k: "email"
})

hash_old = "a1b2c3d4e5f6..."

生成的 URL：
  /api/public/subscription-management/page
  ?w=workspace-456
  &i=user@example.com
  &ik=email
  &h=a1b2c3d4e5f6...  ← 使用旧密钥计算的哈希

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
阶段 2：手动更新 Secret 表（密钥轮换）
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

管理员执行：
  UPDATE Secret
  SET value = "xyz789_new_secret"
  WHERE name = 'subscription' AND workspaceId = 'workspace-456'

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
阶段 3：用户点击旧链接（校验失败）
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

用户点击旧链接：
  GET /api/public/subscription-management/page
  ?w=workspace-456
  &i=user@example.com
  &ik=email
  &h=a1b2c3d4e5f6...  ← URL 中的旧哈希

lookupUserForSubscriptions() 执行：
  ├─► 从数据库获取当前密钥
  │     key_new = "xyz789_new_secret"  ← 新密钥！
  │
  ├─► 找到用户 userId = "user-123"
  │
  └─► 用新密钥重新计算哈希：
        hash_new = HMAC-SHA256(key_new, {
          u: "user-123",
          w: "workspace-456",
          i: "user@example.com",
          k: "email"
        })
        hash_new = "x9y8z7w6v5u4..."  ← 与旧哈希不同！
  │
  ├─► 比对：hash_old === hash_new?
  │     "a1b2c3d4e5f6..." === "x9y8z7w6v5u4..."  →  FALSE
  │
  └─► 返回 Error("Invalid hash")

控制器处理：
  if (userLookupResult.isErr()) {
    return reply.status(401).send({ message: "Unauthorized" });
  }

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

#### 7.3.3 代码层面的校验流程

**校验时只取当前密钥** (`packages/backend-lib/src/subscriptionGroups.ts:603-615`)：

```typescript
const [subscriptionSecret, matchingUserIds] = await Promise.all([
  db().query.secret.findFirst({
    where: and(
      eq(dbSecret.workspaceId, workspaceId),
      eq(dbSecret.name, SecretNames.Subscription),  // ← 只查当前密钥
    ),
    // ❌ 没有历史密钥的概念
    // ❌ 没有查询多个密钥的逻辑
  }),
  findUserIdsByUserPropertyValue({...}),
]);
```

**使用当前唯一密钥进行校验** (`packages/backend-lib/src/subscriptionGroups.ts:628-644`)：

```typescript
const secretValue = subscriptionSecret?.value;
// secretValue = 当前唯一的密钥

const userId = matchingUserIds.find((uId) => {
  const generatedHash = generateSubscriptionHash({
    workspaceId,
    userId: uId,
    identifierKey,
    identifier,
    subscriptionSecret: secretValue,  // ← 只使用当前密钥
  });
  return hash === generatedHash;
});

if (!userId) {
  return err(new Error("Invalid hash"));  // ← 旧链接全部失效
}
```

#### 7.3.4 密钥轮换的影响矩阵

| 场景 | 当前行为 | 用户影响 | 业务影响 |
|------|---------|---------|---------|
| **手动更新 Secret 表** | 所有旧链接立即失效 | 用户收到 401 Unauthorized | 历史邮件退订功能不可用 |
| **工作空间迁移** | 新密钥生成，旧链接失效 | 需要重新发送订阅确认邮件 | 迁移成本高 |
| **密钥泄露后紧急轮换** | 正确行为，但影响大 | 所有历史邮件的退订链接失效 | 安全优先，但用户体验差 |
| **忘记旧密钥** | 无法恢复旧链接 | 永久失效 | 数据丢失 |

#### 7.3.5 建议的改进方案：双密钥过渡机制

**设计思路**：

```
密钥轮换时：
  1. 保留旧密钥（标记为 previous）
  2. 生成新密钥（标记为 active）
  3. 校验时先试 active，再试 previous
  4. 过渡期后（如 90 天）删除 previous
```

**建议的代码实现**：

```typescript
// 新增枚举或常量
enum SecretNames {
  Subscription = "subscription",
  SubscriptionPrevious = "subscription-previous",  // 新增
}

// 改进的校验函数
export async function lookupUserForSubscriptions({...}): Promise<Result<{ userId: string }, Error>> {
  const [secrets, matchingUserIds] = await Promise.all([
    db().query.secret.findMany({
      where: and(
        eq(dbSecret.workspaceId, workspaceId),
        inArray(dbSecret.name, [
          SecretNames.Subscription,           // active
          SecretNames.SubscriptionPrevious,   // previous
        ]),
      ),
      orderBy: [
        // 先尝试 active，再尝试 previous
        sql.raw(`CASE name WHEN '${SecretNames.Subscription}' THEN 0 ELSE 1 END`),
      ],
    }),
    findUserIdsByUserPropertyValue({...}),
  ]);

  if (!matchingUserIds || matchingUserIds.length === 0) {
    return err(new Error("User not found"));
  }

  // 遍历所有密钥进行校验
  for (const secret of secrets) {
    const userId = matchingUserIds.find((uId) => {
      const generatedHash = generateSubscriptionHash({
        workspaceId,
        userId: uId,
        identifierKey,
        identifier,
        subscriptionSecret: secret.value,
      });
      return hash === generatedHash;
    });
    
    if (userId) {
      // 可选：如果使用的是 previous 密钥，记录警告日志
      if (secret.name === SecretNames.SubscriptionPrevious) {
        logger().warn(
          { userId, identifier, identifierKey },
          "User validated using previous subscription secret - consider refreshing links",
        );
      }
      return ok({ userId });
    }
  }

  return err(new Error("Invalid hash"));
}
```

**双密钥机制的优点**：
- 密钥轮换时旧链接不会立即失效
- 有充足的过渡期通知用户更新偏好
- 可以逐步淘汰旧链接

---

### 7.4 安全边界总结

| 维度 | 现状 | 风险等级 | 改进建议 |
|------|------|---------|---------|
| **令牌时效** | 无限制 | 高 | 添加 TTL，如 30-90 天 |
| **一次性使用** | 不支持 | 中 | 记录 Token 使用状态（可选） |
| **重放保护** | 无 | 中 | 依赖业务幂等性，监控异常事件 |
| **密钥轮换** | 不支持（手动更新会失效所有链接） | 高 | 实现双密钥过渡机制 |
| **哈希算法** | HMAC-SHA256 | 低（安全） | ✅ 当前实现良好 |
| **密钥长度** | 64 位 (8 bytes) | 低（可接受） | 可增加到 256 位 |
| **跨渠道校验** | 基于 userId 绑定 | 低（设计合理） | ✅ 当前实现良好 |

---

## 8. 跨渠道校验异常分支时序链

### 8.1 校验入口概述

**核心校验函数**：`lookupUserForSubscriptions()`

**输入参数**：
```typescript
interface UserSubscriptionLookup {
  workspaceId: string;    // w 参数
  identifier: string;     // i 参数
  identifierKey: string;  // ik 参数
  hash: string;           // h 参数
}
```

**校验流程的两个关键步骤**：
```
步骤 1：findUserIdsByUserPropertyValue()
  └─► 根据 identifierKey + identifier 查找匹配的用户 ID 列表

步骤 2：哈希校验
  └─► 遍历匹配的用户 ID，用每个 userId 重新计算哈希
  └─► 找到匹配的 userId 或返回错误
```

---

### 8.2 异常分支 1：用户不存在（同标识命中 0 用户）

#### 8.2.1 触发条件

`identifierKey + identifier` 组合在系统中找不到任何用户。

**可能的原因**：
1. 用户已被删除
2. 批量邮件中的用户数据已过期
3. URL 参数被篡改
4. 用户属性值输入错误

#### 8.2.2 完整时序链

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
T0: 链接被点击
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
GET /api/public/subscription-management/page
  ?w=workspace-abc
  &i=unknown@example.com  ← 这个邮箱不存在
  &ik=email
  &h=any_hash_value

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
T1: 控制器层入口
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
packages/api/src/controllers/subscriptionManagementController.ts:79-126

fastify.withTypeProvider<TypeBoxTypeProvider>().get("/page", {
  schema: {...},
}, async (request, reply) => {
  const {
    w: workspaceId,           // "workspace-abc"
    i: identifier,            // "unknown@example.com"
    ik: identifierKey,        // "email"
    h: hash,                  // "any_hash_value"
    // ...
  } = request.query;

  // 预览模式跳过
  const isPreview = isPreviewParam === "true";

  // 调用 lookupUserForSubscriptions
  const [userLookupResult, workspace] = await Promise.all([
    isPreview
      ? null
      : lookupUserForSubscriptions({
          workspaceId,
          identifier,
          identifierKey,
          hash,
        }),
    db().query.workspace.findFirst({...}),
  ]);

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
T2: lookupUserForSubscriptions() 执行
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
packages/backend-lib/src/subscriptionGroups.ts:597-659

export async function lookupUserForSubscriptions({...}) {
  // 并行获取订阅密钥和匹配的用户 ID
  const [subscriptionSecret, matchingUserIds] = await Promise.all([
    db().query.secret.findFirst({...}),  // 找到密钥
    
    findUserIdsByUserPropertyValue({
      workspaceId: "workspace-abc",
      userPropertyName: "email",
      value: "unknown@example.com",
    }),
  ]);

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
T3: findUserIdsByUserPropertyValue() 执行
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
packages/backend-lib/src/userProperties.ts:851-880

export async function findUserIdsByUserPropertyValue({
  workspaceId,
  userPropertyName,
  value,
}: {...}): Promise<string[] | null> {
  // 1. 先找到 userProperty 定义
  const userProperty = await db()
    .select()
    .from(dbUserProperty)
    .where(
      and(
        eq(dbUserProperty.workspaceId, workspaceId),
        eq(dbUserProperty.name, userPropertyName),  // "email"
      ),
    );

  if (!userProperty || userProperty.length === 0) {
    return null;
  }

  // 2. 查询 UserPropertyAssignment
  const assignments = await db()
    .select({ userId: dbUserPropertyAssignment.userId })
    .from(dbUserPropertyAssignment)
    .where(
      and(
        eq(dbUserPropertyAssignment.workspaceId, workspaceId),
        eq(
          dbUserPropertyAssignment.userPropertyId,
          userProperty[0].id,
        ),
        eq(
          dbUserPropertyAssignment.value,
          JSON.stringify(value),  // "unknown@example.com"
        ),
      ),
    );

  // 3. 返回匹配的用户 ID 列表
  if (assignments.length === 0) {
    return [];  // ← 返回空数组！
  }

  return assignments.map((a) => a.userId);
}

返回：matchingUserIds = []

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
T4: 用户不存在检查
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
packages/backend-lib/src/subscriptionGroups.ts:617-626

// matchingUserIds = []
if (!matchingUserIds || matchingUserIds.length === 0) {
  logger().warn(
    {
      identifier: "unknown@example.com",
      identifierKey: "email",
    },
    "User not found",  // ← 记录 WARN 日志
  );
  return err(new Error("User not found"));  // ← 返回错误
}

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
T5: 控制器层处理错误
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
packages/api/src/controllers/subscriptionManagementController.ts:128-140

let userId: string | undefined;
if (userLookupResult) {
  if (userLookupResult.isErr()) {  // ✓ 进入此分支
    logger().info(
      {
        err: userLookupResult.error,  // Error: "User not found"
      },
      "Failed user lookup for subscription page",
    );
    return reply.status(401).send({
      message: "Unauthorized",  // ← 返回 401
    });
  }
  // ...
}

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
T6: 偏好写回结果
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
由于控制器在 T5 已返回 401，后续流程全部跳过：

✗ lookupUserForSubscriptions() 失败
  ↓
✗ 未获取到 userId
  ↓
✗ 未执行 getUserSubscriptions()
  ↓
✗ 未执行 updateUserSubscriptions() （即使 URL 中有 s 和 sub 参数）
  ↓
✗ 未写入 ClickHouse
✗ 未记录 DFSubscriptionChange 事件

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
T7: 用户看到的结果
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
HTTP/1.1 401 Unauthorized
Content-Type: application/json

{
  "message": "Unauthorized"
}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

#### 8.2.3 日志输出

```
[WARN] User not found
  identifier: "unknown@example.com"
  identifierKey: "email"

[INFO] Failed user lookup for subscription page
  err: Error: User not found
```

#### 8.2.4 写回结果对照表

| 操作 | 状态 |
|------|------|
| ClickHouse 写入 | ❌ 无 |
| Segment 赋值 | ❌ 无 |
| DFSubscriptionChange 事件 | ❌ 无 |
| 用户订阅状态 | 保持不变 |

---

### 8.3 异常分支 2：同标识命中多用户，哈希全部不匹配

#### 8.3.1 触发条件

多个用户拥有相同的用户属性值（如共享邮箱），但没有任何一个用户的哈希匹配传入的 hash。

**典型场景**：
- 多个用户使用同一个 `managerEmail` 属性值（如团队邮箱）
- URL 中的 hash 是为某个特定用户生成的，但该用户已被删除
- 攻击者尝试用已知邮箱枚举用户

#### 8.3.2 完整时序链

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
T0: 链接被点击
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
GET /api/public/subscription-management/page
  ?w=workspace-abc
  &i=shared@company.com
  &ik=managerEmail
  &h=wrong_hash_or_user_deleted

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
T1: findUserIdsByUserPropertyValue() 返回多个用户
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
数据库状态：
  UserPropertyAssignment 表中有 3 条记录：
    - userId: user-A, userPropertyId: managerEmail, value: "shared@company.com"
    - userId: user-B, userPropertyId: managerEmail, value: "shared@company.com"
    - userId: user-C, userPropertyId: managerEmail, value: "shared@company.com"

findUserIdsByUserPropertyValue() 返回：
  matchingUserIds = ["user-A", "user-B", "user-C"]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
T2: 遍历校验哈希（array.find()）
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
packages/backend-lib/src/subscriptionGroups.ts:635-644

const secretValue = subscriptionSecret?.value;  // "current_secret_key"

const userId = matchingUserIds.find((uId) => {
  // 对每个匹配的用户 ID 计算哈希
  const generatedHash = generateSubscriptionHash({
    workspaceId: "workspace-abc",
    userId: uId,              // 依次是 "user-A", "user-B", "user-C"
    identifierKey: "managerEmail",
    identifier: "shared@company.com",
    subscriptionSecret: secretValue,
  });
  return hash === generatedHash;  // 比对
});

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
T3: 逐个用户校验细节
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
第 1 次迭代 (uId = "user-A"):
  toHash = {
    u: "user-A",
    w: "workspace-abc",
    i: "shared@company.com",
    k: "managerEmail"
  }
  generatedHash_A = HMAC-SHA256("current_secret_key", toHash)
  generatedHash_A = "hash_for_user_A_abc123..."
  
  hash === generatedHash_A?
  "wrong_hash_or_user_deleted" === "hash_for_user_A_abc123..."  →  FALSE

第 2 次迭代 (uId = "user-B"):
  toHash = {
    u: "user-B",  ← 不同的 userId！
    w: "workspace-abc",
    i: "shared@company.com",
    k: "managerEmail"
  }
  generatedHash_B = HMAC-SHA256("current_secret_key", toHash)
  generatedHash_B = "hash_for_user_B_def456..."
  
  hash === generatedHash_B?
  "wrong_hash_or_user_deleted" === "hash_for_user_B_def456..."  →  FALSE

第 3 次迭代 (uId = "user-C"):
  toHash = {
    u: "user-C",  ← 不同的 userId！
    w: "workspace-abc",
    i: "shared@company.com",
    k: "managerEmail"
  }
  generatedHash_C = HMAC-SHA256("current_secret_key", toHash)
  generatedHash_C = "hash_for_user_C_ghi789..."
  
  hash === generatedHash_C?
  "wrong_hash_or_user_deleted" === "hash_for_user_C_ghi789..."  →  FALSE

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
T4: 遍历结束，无匹配
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
array.find() 返回：
  userId = undefined  ← 没有找到匹配

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
T5: 无效哈希检查
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
packages/backend-lib/src/subscriptionGroups.ts:646-656

if (!userId) {
  logger().warn(
    {
      workspaceId: "workspace-abc",
      identifier: "shared@company.com",
      identifierKey: "managerEmail",
      hash: "wrong_hash_or_user_deleted",
    },
    "Invalid hash",  // ← 记录 WARN 日志
  );
  return err(new Error("Invalid hash"));  // ← 返回错误
}

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
T6: 控制器层处理错误
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
packages/api/src/controllers/subscriptionManagementController.ts:128-140

let userId: string | undefined;
if (userLookupResult) {
  if (userLookupResult.isErr()) {  // ✓ 进入此分支
    logger().info(
      { err: userLookupResult.error },  // Error: "Invalid hash"
      "Failed user lookup for subscription page",
    );
    return reply.status(401).send({
      message: "Unauthorized",  // ← 返回 401
    });
  }
}

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
T7: 偏好写回结果
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
与分支 1 相同，后续流程全部跳过：

✗ 未获取到任何 userId
  ↓
✗ 未执行 getUserSubscriptions()
  ↓
✗ 未执行 updateUserSubscriptions()
  ↓
✗ 未写入 ClickHouse
✗ 未记录 DFSubscriptionChange 事件

所有用户（user-A, user-B, user-C）的订阅状态保持不变 ✓

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

#### 8.3.3 关键代码：多用户遍历校验

```typescript
// packages/backend-lib/src/subscriptionGroups.ts:635-644
const userId = matchingUserIds.find((uId) => {
  const generatedHash = generateSubscriptionHash({
    workspaceId,
    userId: uId,              // ← 关键：每个用户的哈希不同
    identifierKey,
    identifier,
    subscriptionSecret: secretValue,
  });
  return hash === generatedHash;
});
```

**设计优点**：
- 即使多个用户共享相同的 `identifier`，由于 `userId` 不同，哈希也不同
- 只有生成该链接的原始用户才能通过校验

#### 8.3.4 日志输出

```
[WARN] Invalid hash
  workspaceId: "workspace-abc"
  identifier: "shared@company.com"
  identifierKey: "managerEmail"
  hash: "wrong_hash_or_user_deleted"

[INFO] Failed user lookup for subscription page
  err: Error: Invalid hash
```

---

### 8.4 成功分支：同标识命中多用户，哈希匹配其中一个

#### 8.4.1 触发条件

多个用户共享同一属性值，哈希匹配正确的用户（这是设计预期的正常场景）。

#### 8.4.2 完整时序链（到偏好写回）

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
T0: 链接被点击（用户 A 的链接）
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
GET /api/public/subscription-management/page
  ?w=workspace-abc
  &i=shared@company.com
  &ik=managerEmail
  &h=hash_for_user_A_abc123...  ← 为 user-A 生成的哈希
  &s=sg-marketing
  &sub=0

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
T1: findUserIdsByUserPropertyValue() 返回多个用户
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
matchingUserIds = ["user-A", "user-B", "user-C"]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
T2: 遍历校验哈希
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
第 1 次迭代 (uId = "user-A"):
  generatedHash_A = HMAC-SHA256(secret, {
    u: "user-A", w, i, k
  })
  generatedHash_A = "hash_for_user_A_abc123..."
  
  hash === generatedHash_A?
  "hash_for_user_A_abc123..." === "hash_for_user_A_abc123..."  →  TRUE ✓
  
  → array.find() 立即停止遍历，返回 userId = "user-A"

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
T3: 返回成功
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
return ok({ userId: "user-A" });

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
T4: 控制器层继续执行
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
packages/api/src/controllers/subscriptionManagementController.ts:156-219

// 处理 s 和 sub 参数
if (subscriptionGroupId && sub) {
  subscriptionChange = sub === "1" ? Subscribe : Unsubscribe;  // Unsubscribe
  
  const targetSubscriptionGroup = await db().query.subscriptionGroup.findFirst({
    where: eq(schema.subscriptionGroup.id, "sg-marketing"),
  });
  // channel = Email
  
  if (subscriptionChange === Unsubscribe) {
    // 退订该 channel 的所有订阅分组
    const channelSubscriptionGroups = await db().query.subscriptionGroup.findMany({
      where: and(
        eq(dbSubscriptionGroup.workspaceId, workspaceId),
        eq(dbSubscriptionGroup.channel, "Email"),
      ),
    });
    
    const channelChanges: Record<string, boolean> = {};
    channelSubscriptionGroups.forEach((sg) => {
      channelChanges[sg.id] = false;
    });
    
    // 执行更新
    await updateUserSubscriptions({
      workspaceId,
      userUpdates: [{
        userId: "user-A",  // ✓ 只更新 user-A
        changes: channelChanges,
      }],
    });
  }
}

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
T5: updateUserSubscriptions() 执行
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
packages/backend-lib/src/subscriptionGroups.ts:668-787

// 获取相关 Segments
const segments = await db().query.segment.findMany({...});

// 构建 Segment 赋值更新
const segmentAssignmentUpdates: SegmentBulkUpsertItem[] = [
  {
    workspaceId: "workspace-abc",
    userId: "user-A",        // ✓ 只更新 user-A
    segmentId: "segment-sg-marketing-main",
    inSegment: false,
  },
  {
    workspaceId: "workspace-abc",
    userId: "user-A",        // ✓ 只更新 user-A
    segmentId: "segment-sg-marketing-unsubscribed",
    inSegment: true,
  },
  // ... 其他 Email 渠道的订阅分组
];

// 构建事件
const allUserEvents = [
  {
    messageId: "uuid-xxx",
    messageRaw: JSON.stringify({
      userId: "user-A",       // ✓ 只记录 user-A
      type: "track",
      event: "$df_subscription_change",
      properties: {
        subscriptionId: "sg-marketing",
        action: "Unsubscribe",
      },
    }),
  },
  // ...
];

// 并行写入
await Promise.all([
  insertSegmentAssignments(segmentAssignmentUpdates),
  insertUserEvents({ workspaceId, userEvents: allUserEvents }),
]);

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
T6: 偏好写回结果
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
ClickHouse (computed_property_assignments_v2):
  ✓ user-A, sg-marketing-main: inSegment = false
  ✓ user-A, sg-marketing-unsubscribed: inSegment = true
  ✗ user-B: 无变化
  ✗ user-C: 无变化

事件系统:
  ✓ 记录 user-A 的 DFSubscriptionChange 事件
  ✗ user-B: 无事件
  ✗ user-C: 无事件

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
T7: 最终状态
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
用户 A：已退订 Marketing Emails ✓
用户 B：订阅状态保持不变 ✓
用户 C：订阅状态保持不变 ✓
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

### 8.5 异常分支 3：用户存在但哈希不匹配

#### 8.5.1 触发条件

找到用户（单个或多个），但哈希校验失败。

**可能的原因**：
1. URL 参数被手动修改
2. 订阅密钥已轮换（见 7.3 节）
3. 链接来自不同的工作空间
4. 哈希生成算法被修改（升级场景）
5. 攻击者尝试暴力破解

#### 8.5.2 与分支 2 的区别

| 维度 | 分支 2（多用户+无匹配） | 分支 3（单用户+无匹配） |
|------|----------------------|----------------------|
| 匹配用户数 | ≥ 1（通常多个） | ≥ 1（通常单个） |
| 哈希校验 | 全部不匹配 | 不匹配 |
| 错误信息 | "Invalid hash" | "Invalid hash" |
| 写回结果 | 无任何写入 | 无任何写入 |

#### 8.5.3 时序链（简化版）

```
T0: 链接被点击（篡改后的 hash）
  ?i=user@example.com
  &ik=email
  &h=hacked_hash

T1: findUserIdsByUserPropertyValue()
  matchingUserIds = ["user-123"]

T2: 哈希校验
  generatedHash = HMAC-SHA256(secret, {u: "user-123", w, i, k})
  generatedHash = "correct_hash_abc123..."
  
  hash === generatedHash?
  "hacked_hash" === "correct_hash_abc123..."  →  FALSE

T3: 返回错误
  return err(new Error("Invalid hash"))

T4: 控制器返回 401
  HTTP 401 Unauthorized

T5: 写回结果
  ✗ 无数据库写入
  ✗ 无 Segment 赋值
  ✗ 无订阅变更事件
  
  用户状态保持不变 ✓
```

---

### 8.6 异常分支决策树（完整）

```
                            用户点击链接
                                 │
                                 ▼
                    ┌─────────────────────────────┐
                    │  提取 URL 参数               │
                    │  w, i, ik, h, (s, sub)     │
                    └─────────────┬───────────────┘
                                  │
                                  ▼
                    ┌─────────────────────────────┐
                    │  lookupUserForSubscriptions()│
                    └─────────────┬───────────────┘
                                  │
          ┌───────────────────────┼───────────────────────┐
          │                       │                       │
          ▼                       ▼                       ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│ 步骤 1:          │    │ 步骤 1:          │    │ 步骤 1:          │
│ findUserIds      │    │ findUserIds      │    │ findUserIds      │
│ ByUserProperty   │    │ ByUserProperty   │    │ ByUserProperty   │
│ Value()          │    │ Value()          │    │ Value()          │
└────────┬────────┘    └────────┬────────┘    └────────┬────────┘
         │                      │                      │
         ▼                      ▼                      ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│ 返回 []         │    │ 返回 [A,B,C]    │    │ 返回 [A]        │
│ (0 个用户)     │    │ (≥1 个用户)     │    │ (1 个用户)      │
└────────┬────────┘    └────────┬────────┘    └────────┬────────┘
         │                      │                      │
         ▼                      ▼                      ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│ "User not       │    │ 步骤 2: 遍历    │    │ 步骤 2: 计算    │
│ found"          │    │ 校验哈希        │    │ 哈希            │
└────────┬────────┘    └────────┬────────┘    └────────┬────────┘
         │                      │                      │
    ┌────┴────┐           ┌─────┴─────┐          ┌─────┴─────┐
    │         │           │           │          │           │
    ▼         ▼           ▼           ▼          ▼           ▼
  401      无写入      有匹配?      无匹配      匹配?       不匹配
           无事件       │            │          │           │
                        ▼            ▼          ▼           ▼
                     正常流程     "Invalid    正常流程    "Invalid
                     (分支 3)     hash"       (正常)      hash"
                                  │                         │
                                  ▼                         ▼
                                401                       401
                                无写入                     无写入
                                无事件                     无事件
```

---

### 8.7 各分支写回结果对比表

| 分支编号 | 场景描述 | HTTP 状态 | ClickHouse 写入 | DFSubscriptionChange 事件 | 用户订阅状态 |
|---------|---------|----------|----------------|-------------------------|------------|
| **1** | 用户不存在（0 匹配） | 401 | ❌ 无 | ❌ 无 | 无变化 |
| **2** | 多用户命中，无匹配哈希 | 401 | ❌ 无 | ❌ 无 | 所有用户无变化 |
| **3** | 多用户命中，哈希匹配一个 | 200 | ✅ 写入匹配用户 | ✅ 记录匹配用户 | 匹配用户变更，其他不变 |
| **4** | 单用户命中，哈希不匹配 | 401 | ❌ 无 | ❌ 无 | 无变化 |
| **正常** | 单用户 + 哈希匹配 | 200 | ✅ 写入 | ✅ 记录 | 正常变更 |

---

### 8.8 事件审计视角

#### 8.8.1 正常流程的事件序列

```
T0: MessageSent
  消息类型: Email
  用户: user@example.com

T1: UserClicked (可选，取决于邮件服务商)
  链接: /api/public/subscription-management/page?...

T2: DFSubscriptionChange
  userId: "user-123"
  properties:
    subscriptionId: "sg-marketing"
    action: "Unsubscribe"

T3: (后续广播)
  MessageSkipped
    variant:
      type: SubscriptionState
      action: Unsubscribe
      subscriptionGroupType: OptOut
```

#### 8.8.2 异常分支的事件序列（分支 1/2/4）

```
T0: MessageSent
  消息类型: Email
  用户: user@example.com

T1: UserClicked (链接被点击)

T2: (无 DFSubscriptionChange)
  原因:
    - lookupUserForSubscriptions 返回 "User not found"
    - 或返回 "Invalid hash"
    - updateUserSubscriptions 未执行

T3: (后续广播)
  MessageSent
    原因: 用户订阅状态未变
```

#### 8.8.3 日志监控建议

**需要关注的 WARN 日志**：

```
# 分支 1：用户不存在
[WARN] User not found
  identifier: "..."
  identifierKey: "..."

# 分支 2/4：哈希不匹配
[WARN] Invalid hash
  workspaceId: "..."
  identifier: "..."
  identifierKey: "..."
  hash: "..."

# INFO 级别补充
[INFO] Failed user lookup for subscription page
  err: Error: "User not found" 或 "Invalid hash"
```

**异常率过高可能表示**：
1. 批量邮件中的用户已被删除
2. 有人尝试枚举用户（identifier 变化，hash 固定）
3. 订阅密钥已轮换（所有旧链接失效）
4. URL 生成逻辑有 bug
5. 攻击者尝试暴力破解

---

## 9. 关键代码位置汇总

### 9.1 核心逻辑文件

| 文件路径 | 功能描述 |
|---------|---------|
| `packages/backend-lib/src/subscriptionGroups.ts` | 订阅分组核心逻辑，包括 URL 生成、哈希生成、用户查找、订阅更新 |
| `packages/backend-lib/src/subscriptionManagementPage.ts` | 订阅管理页面渲染逻辑 |
| `packages/backend-lib/src/subscriptionManagementTemplate.ts` | 订阅管理页面模板和 Liquid 标签 |
| `packages/backend-lib/src/liquid.ts` | Liquid 模板引擎，包含订阅链接标签 |
| `packages/backend-lib/src/crypto.ts` | 加密工具函数（HMAC-SHA256、安全密钥生成） |
| `packages/api/src/controllers/subscriptionManagementController.ts` | 订阅管理 API 控制器 |
| `packages/api/src/controllers/subscriptionGroupsController.ts` | 订阅分组管理 API 控制器 |
| `packages/backend-lib/src/messaging.ts` | 消息发送时的订阅检查逻辑 |

### 7.2 测试文件

| 文件路径 | 测试内容 |
|---------|---------|
| `packages/backend-lib/src/subscriptionGroups.test.ts` | 订阅分组单元测试 |
| `packages/backend-lib/src/subscriptionManagementEndToEnd.test.ts` | 端到端测试，包含完整的退订流程 |
| `packages/backend-lib/src/jsdom-tests/subscriptionManagementPage.test.ts` | 订阅管理页面测试 |

### 7.3 数据库 Schema

| 文件路径 | 说明 |
|---------|------|
| `packages/backend-lib/src/db/schema.ts:317-349` | SubscriptionGroup 表定义 |
| `packages/backend-lib/src/db/schema.ts:749` | Segment 表的 subscriptionGroupId 字段 |
| `packages/backend-lib/src/db/schema.ts:57-60` | DBSubscriptionGroupType 枚举定义 |
| `packages/backend-lib/src/db/schema.ts` | Secret 表（存储订阅密钥） |

---

## 附录: 类型定义速查

### SubscriptionParams (URL 参数)
```typescript
export const SubscriptionParams = Type.Object({
  w: Type.String({ description: "Workspace Id." }),
  i: Type.String({ description: 'Identifier value for channel e.g. "name@email.com".' }),
  ik: Type.String({ description: 'Identifier key for channel e.g. "email".' }),
  h: Type.String({ description: "Hash for verifying the user." }),
  s: Type.Optional(Type.String({ description: "Subscription group id." })),
  sub: Type.Optional(Type.Union([Type.Literal("0"), Type.Literal("1")], {
    description: "Subscription action. 1 for subscribe, 0 for unsubscribe.",
  })),
  isPreview: Type.Optional(Type.String({ description: "Whether the request is a preview." })),
  showAllChannels: Type.Optional(Type.String({ description: "Whether to show all channels." })),
  success: Type.Optional(Type.String({ description: "Whether the form submission was successful." })),
  error: Type.Optional(Type.String({ description: "Whether the form submission failed." })),
  previewSubmitted: Type.Optional(Type.String({ description: "Whether the preview form was submitted." })),
});
```

### UserSubscriptionsUpdate (订阅更新请求)
```typescript
export const UserSubscriptionsUpdate = Type.Intersect([
  UserSubscriptionLookup,  // workspaceId, hash, identifier, identifierKey
  Type.Object({
    changes: Type.Record(Type.String(), Type.Boolean(), {
      description: "Subscription changes.",
    }),
  }),
]);
```

### SubscriptionChangeEvent (订阅变更事件)
```typescript
export interface SubscriptionChangeEvent {
  type: EventType.Track;
  event: InternalEventType.SubscriptionChange;
  properties: {
    subscriptionId: string;
    action: SubscriptionChange;  // "Subscribe" | "Unsubscribe"
  };
}
```
