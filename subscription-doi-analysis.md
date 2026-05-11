# Dittofeed 订阅分组与 DOI 流程分析报告

## 目录
1. [订阅分组架构概述](#1-订阅分组架构概述)
2. [订阅分组与渠道偏好管理](#2-订阅分组与渠道偏好管理)
3. [DOI 令牌生成机制](#3-doi-令牌生成机制)
4. [DOI 跨渠道校验流程](#4-doi-跨渠道校验流程)
5. [偏好写回路径](#5-偏好写回路径)
6. [完整流程串联](#6-完整流程串联)
7. [关键代码位置](#7-关键代码位置)

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

## 7. 关键代码位置汇总

### 7.1 核心逻辑文件

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
