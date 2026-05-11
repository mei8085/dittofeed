# Dittofeed 璁㈤槄鍒嗙粍涓?DOI 娴佺▼鍒嗘瀽鎶ュ憡

## 鐩綍
1. [璁㈤槄鍒嗙粍鏋舵瀯姒傝堪](#1-璁㈤槄鍒嗙粍鏋舵瀯姒傝堪)
2. [璁㈤槄鍒嗙粍涓庢笭閬撳亸濂界鐞哴(#2-璁㈤槄鍒嗙粍涓庢笭閬撳亸濂界鐞?
3. [DOI 浠ょ墝鐢熸垚鏈哄埗](#3-doi-浠ょ墝鐢熸垚鏈哄埗)
4. [DOI 璺ㄦ笭閬撴牎楠屾祦绋媇(#4-doi-璺ㄦ笭閬撴牎楠屾祦绋?
5. [鍋忓ソ鍐欏洖璺緞](#5-鍋忓ソ鍐欏洖璺緞)
6. [瀹屾暣娴佺▼涓茶仈](#6-瀹屾暣娴佺▼涓茶仈)
7. [DOI 浠ょ墝瀹夊叏涓庤竟鐣屽垎鏋怾(#7-doi-浠ょ墝瀹夊叏涓庤竟鐣屽垎鏋?
8. [璺ㄦ笭閬撴牎楠屽紓甯稿垎鏀椂搴忛摼](#8-璺ㄦ笭閬撴牎楠屽紓甯稿垎鏀椂搴忛摼)
9. [鍏抽敭浠ｇ爜浣嶇疆](#9-鍏抽敭浠ｇ爜浣嶇疆)

---

## 1. 璁㈤槄鍒嗙粍鏋舵瀯姒傝堪

### 1.1 鏁版嵁妯″瀷

Dittofeed 鐨勮闃呭垎缁?(Subscription Groups) 鏄鐞嗙敤鎴峰悇娓犻亾鎺ユ敹鍋忓ソ鐨勬牳蹇冩満鍒躲€傛瘡涓闃呭垎缁勫叧鑱斿埌鐗瑰畾鐨勬笭閬擄紙Email銆丼MS銆丮obilePush銆乄ebhook锛夛紝骞舵敮鎸佷袱绉嶇被鍨嬶細

- **OptIn锛堜富鍔ㄨ闃咃級锛氱敤鎴峰繀椤绘樉寮忚闃呮墠鑳芥帴鏀舵秷鎭?- **OptOut锛堣鍔ㄨ闃咃級锛氱敤鎴烽粯璁ゆ帴鏀讹紝鍙€夋嫨閫€鍑?
**鏁版嵁搴撴ā鍨?* (`packages/backend-lib/src/db/schema.ts:317-349`)锛?```typescript
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

### 1.2 璁㈤槄鍒嗙粍涓?Segments 鐨勫叧绯?
姣忎釜璁㈤槄鍒嗙粍鍒涘缓鏃朵細鑷姩鐢熸垚涓や釜鍏宠仈鐨?Segments锛坄packages/backend-lib/src/subscriptionGroups.ts:332-395`锛夛細

1. **涓?Segment**锛氬懡鍚嶆牸寮?`subscriptionGroup-{id}
   - 鐢ㄤ簬鏍囪鐢ㄦ埛鏄惁鍦ㄨ闃呭垎缁勫唴
   
2. **鏈闃?Segment**锛氬懡鍚嶆牸寮?`subscriptionGroup-unsubscribed-{id}`
   - 鐢ㄤ簬鏍囪鐢ㄦ埛鏄惁宸蹭粠璁㈤槄鍒嗙粍閫€鍑?
```typescript
// 鍛藉悕鍑芥暟瀹氫箟
export function getSubscriptionGroupSegmentName(id: string) {
  return `subscriptionGroup-${id}`;
}

export function getSubscriptionGroupUnsubscribedSegmentName(id: string) {
  return `subscriptionGroup-unsubscribed-${id}`;
}
```

---

## 2. 璁㈤槄鍒嗙粍涓庢笭閬撳亸濂界鐞?
### 2.1 璁㈤槄鐘舵€佸垽鏂€昏緫

**鏍稿績鍑芥暟** `inSubscriptionGroup` (`packages/backend-lib/src/subscriptionGroups.ts:83-91`)锛?
```typescript
export function inSubscriptionGroup(
  details: SubscriptionGroupDetails,
): boolean {
  // 鍦ㄨ闃呭垎缁?segment 灏氭湭璁＄畻鐨勬儏鍐典笅
  if (details.action === null && details.type === SubscriptionGroupType.OptIn) {
    return false;  // OptIn 榛樿涓嶈闃?  }
  return details.action !== SubscriptionChange.Unsubscribe;
}
```

**鍒ゆ柇瑙勫垯**锛?- **OptIn 绫诲瀷**锛?  - 鏈缃姸鎬?(action = null) 鈫?涓嶈闃?(false)
  - 鏄惧紡璁㈤槄 (action = Subscribe) 鈫?璁㈤槄 (true)
  - 鏄惧紡閫€璁?(action = Unsubscribe) 鈫?涓嶈闃?(false)

- **OptOut 绫诲瀷**锛?  - 鏈缃姸鎬?(action = null) 鈫?榛樿璁㈤槄 (true)
  - 鏄惧紡璁㈤槄 (action = Subscribe) 鈫?璁㈤槄 (true)
  - 鏄惧紡閫€璁?(action = Unsubscribe) 鈫?涓嶈闃?(false)

### 2.2 娑堟伅鍙戦€佹椂鐨勮闃呮鏌?
鍦ㄥ彂閫佹秷鎭墠锛岀郴缁熶細妫€鏌ョ敤鎴风殑璁㈤槄鐘舵€?(`packages/backend-lib/src/messaging.ts:422-436`)锛?
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

濡傛灉鐢ㄦ埛涓嶆弧瓒宠闃呮潯浠讹紝娑堟伅浼氳璺宠繃锛屽苟璁板綍 `MessageSkipped` 浜嬩欢銆?
---

## 3. DOI 浠ょ墝鐢熸垚鏈哄埗

### 3.1 璁㈤槄瀵嗛挜鐢熸垚

宸ヤ綔绌洪棿鍒濆鍖栨椂浼氱敓鎴愯闃呭瘑閽?(`packages/backend-lib/src/subscriptionGroups.ts:789-807`)锛?
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
      value: generateSecureKey(8),  // 鐢熸垚 16 瀛楃鐨勯殢鏈哄瘑閽?    },
  }).then(unwrap);
}
```

瀵嗛挜鐢熸垚浣跨敤 `crypto.randomBytes() 鍑芥暟锛?```typescript
// packages/backend-lib/src/crypto.ts:59-61
export function generateSecureKey(length = 32): string {
  return crypto.randomBytes(length).toString("hex");
}
```

### 3.2 璁㈤槄鍝堝笇鐢熸垚

**鍝堝笇鐢熸垚浣跨敤 HMAC-SHA256 绠楁硶 (`packages/backend-lib/src/subscriptionGroups.ts:426-451`)锛?
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
  identifierKey: string;  // 濡?"email"
  identifier: string;     // 濡?"user@example.com"
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

鍝堝笇绠楁硶瀹炵幇 (`packages/backend-lib/src/crypto.ts:38-57`)锛?```typescript
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

### 3.3 璁㈤槄绠＄悊 URL 鐢熸垚

**瀹屾暣 URL 鐢熸垚鍑芥暟** (`packages/backend-lib/src/subscriptionGroups.ts:453-510`)锛?
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
    w: workspaceId,     // 宸ヤ綔绌洪棿 ID
    i: identifier,        // 鏍囪瘑绗﹀€硷紙濡傞偖绠便€佹墜鏈哄彿锛?    ik: identifierKey,  // 鏍囪瘑绗︾被鍨嬶紙濡?"email"銆?phone"锛?    h: hash,          // 楠岃瘉鍝堝笇
  };
  
  // 鍙€夊弬鏁?  if (changedSubscription) {
    params.s = changedSubscription;  // 璁㈤槄鍒嗙粍 ID
    params.sub =
      subscriptionChange === SubscriptionChange.Subscribe ? "1" : "0";  // 1=璁㈤槄锛?=閫€璁?  }
  if (isPreview) {
    params.isPreview = "true";
  }
  if (showAllChannels) {
    params.showAllChannels = "true";
  }
  
  // 鏋勫缓瀹屾暣 URL
  const url = new URL(config().apiBase || config().dashboardUrl);
  url.pathname = "/api/public/subscription-management/page";
  url.search = new URLSearchParams(params).toString();
  
  return url.toString();
}
```

**URL 鍙傛暟璇存槑**锛?| 鍙傛暟 | 璇存槑 | 绀轰緥 |
|------|------|------|
| `w` | 宸ヤ綔绌洪棿 ID | `uuid-v4-string` |
| `i` | 鏍囪瘑绗﹀€?| `user@example.com` |
| `ik` | 鏍囪瘑绗︾被鍨?| `email` |
| `h` | HMAC-SHA256 鍝堝笇 | `abc123...` |
| `s` | 璁㈤槄鍒嗙粍 ID锛堝彲閫夛級 | `subscription-group-uuid` |
| `sub` | 璁㈤槄鎿嶄綔锛?=璁㈤槄锛?=閫€璁級 | `1` 鎴?`0` |

### 3.4 Liquid 妯℃澘涓殑璁㈤槄閾炬帴

鍦ㄦ秷鎭ā鏉夸腑锛岄€氳繃 Liquid 鏍囩鐢熸垚璁㈤槄绠＄悊閾炬帴 (`packages/backend-lib/src/liquid.ts:80-200`)锛?
**閫€璁㈤摼鎺ユ爣绛?*锛?```typescript
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

**璁㈤槄绠＄悊閾炬帴鏍囩**锛?```typescript
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

**娉ㄦ剰**锛氶摼鎺ユ坊鍔犱簡 `clicktracking=off 灞炴€э紝闃叉 SendGrid 绛夐偖浠舵湇鍔″晢鐨勯摼鎺ヨ拷韪共鎵伴€€璁㈡祦绋嬨€?
---

## 4. DOI 璺ㄦ笭閬撴牎楠屾祦绋?
### 4.1 鐢ㄦ埛鏌ユ壘涓庡搱甯屾牎楠?
**鏍稿績鏍￠獙鍑芥暟** `lookupUserForSubscriptions` (`packages/backend-lib/src/subscriptionGroups.ts:597-659`)锛?
```typescript
export async function lookupUserForSubscriptions({
  workspaceId,
  identifier,
  identifierKey,
  hash,
}: UserSubscriptionLookup): Promise<Result<{ userId: string }, Error>> {
  // 1. 骞惰鑾峰彇璁㈤槄瀵嗛挜鍜屽尮閰嶇殑鐢ㄦ埛 ID
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

  // 2. 妫€鏌ョ敤鎴锋槸鍚﹀瓨鍦?  if (!matchingUserIds || matchingUserIds.length === 0) {
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

  // 3. 閬嶅巻鎵€鏈夊尮閰嶇殑鐢ㄦ埛锛屾牎楠屽搱甯?  const userId = matchingUserIds.find((uId) => {
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

### 4.2 璺ㄦ笭閬撴爣璇嗙鏀寔

**璁捐鍘熺悊**锛?- 鏍囪瘑绗﹀彲浠ユ槸浠讳綍鐢ㄦ埛灞炴€э紙email銆乸hone銆乵anagerEmail 绛夛級
- 鍝堝笇缁戝畾鐨勬槸 `userId`锛岃€屼笉鏄壒瀹氭笭閬撴爣璇嗙
- 鍗充娇浣跨敤涓嶅悓娓犻亾鐨勬爣璇嗙锛屽彧瑕佸睘浜庡悓涓€涓敤鎴凤紝鍝堝笇鏍￠獙灏辫兘閫氳繃

**瀹為檯娴嬭瘯妗堜緥** (`packages/backend-lib/src/subscriptionManagementEndToEnd.test.ts:254-510`)锛?```typescript
// 鐢ㄦ埛鏈変袱涓睘鎬э細
// - email: "user@example.com"
// - managerEmail: "manager@company.com"

// 鍙戦€佺粰 manager 鐨勬秷鎭腑锛屼娇鐢?managerEmail 浣滀负 identifierKey
// 鐢熸垚鐨?unsubscribe 閾炬帴鍖呭惈锛?// - ik: "managerEmail"
// - i: "manager@company.com"
// - h: 鍩轰簬 userId銆亀orkspaceId銆乮dentifierKey銆乮dentifier 鐢熸垚鐨勫搱甯?
// 褰?manager 鐐瑰嚮閫€璁㈤摼鎺ユ椂锛?// 1. 绯荤粺鏍规嵁 ik 鍜?i 鏌ユ壘鐢ㄦ埛
// 2. 鎵惧埌 userId锛堝洜涓?managerEmail 灞炴€у睘浜庤鐢ㄦ埛锛?// 3. 浣跨敤鎵惧埌鐨?userId 閲嶆柊璁＄畻鍝堝笇杩涜鏍￠獙
// 4. 鏍￠獙閫氳繃锛屾墽琛岄€€璁㈡搷浣?```

### 4.3 API 灞傛牎楠?
**PUT /user-subscriptions 绔偣鏍￠獙 (`packages/api/src/controllers/subscriptionManagementController.ts:31-76`)锛?
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

    // 鏍￠獙鐢ㄦ埛韬唤
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

    // 鏇存柊璁㈤槄
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

## 5. 鍋忓ソ鍐欏洖璺緞

### 5.1 璁㈤槄鏇存柊鏍稿績鍑芥暟

**updateUserSubscriptions** (`packages/backend-lib/src/subscriptionGroups.ts:668-787`)锛?
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
  // 1. 鑾峰彇鎵€鏈夌浉鍏宠闃呭垎缁勭殑 Segments
  const subscriptionGroupIds = userUpdates.flatMap((u) =>
    Object.keys(u.changes),
  );
  const segments = await db().query.segment.findMany({
    where: and(
      eq(dbSegment.workspaceId, workspaceId),
      inArray(dbSegment.subscriptionGroupId, subscriptionGroupIds),
    ),
  });

  // 2. 鎸夎闃呭垎缁?ID 鏄犲皠涓?Segment 鍜屾湭璁㈤槄 Segment
  const segmentsBySubscriptionGroupId = segments.reduce<
    Record<string, SegmentPair>
  >((acc, segment) => {
    // ... 鏄犲皠閫昏緫
  }, {});

  // 3. 鏋勫缓璁㈤槄鍙樻洿浜嬩欢
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

  // 4. 鏋勫缓 Segment 璧嬪€兼洿鏂?  const segmentAssignmentUpdates: SegmentBulkUpsertItem[] = userUpdates.flatMap(
    ({ userId, changes }) => {
      const changePairs = R.entries(changes);
      return changePairs.flatMap(([subscriptionGroupId, isSubscribed]) => {
        const segmentPair = segmentsBySubscriptionGroupId[subscriptionGroupId];
        if (!segmentPair) {
          return [];
        }

        const assignments: SegmentBulkUpsertItem[] = [];

        // 涓?Segment: inSegment = isSubscribed
        if (segmentPair.mainSegmentId) {
          assignments.push({
            workspaceId,
            userId,
            segmentId: segmentPair.mainSegmentId,
            inSegment: isSubscribed,
          });
        }

        // 鏈闃?Segment: inSegment = !isSubscribed
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

  // 5. 骞惰鍐欏叆 ClickHouse 鍜屼簨浠剁郴缁?  await Promise.all([
    insertSegmentAssignments(segmentAssignmentUpdates),
    insertUserEvents({
      workspaceId,
      userEvents: allUserEvents,
    }),
  ]);
}
```

### 5.2 Segment 璧嬪€煎啓鍏?
**insertSegmentAssignments** (`packages/backend-lib/src/segments.ts:1023-1038`)锛?
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

### 5.3 璁㈤槄鍙樻洿浜嬩欢

**buildSubscriptionChangeEvent** (`packages/backend-lib/src/subscriptionGroups.ts:542-568`)锛?
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

## 6. 瀹屾暣娴佺▼涓茶仈

### 6.1 瀹屾暣 DOI 娴佺▼鍥?
```
鈹屸攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?鈹?                   Double Opt-In (DOI) 瀹屾暣娴佺▼                                鈹?鈹斺攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?
  闃舵 1: 娑堟伅鍙戦€佷笌浠ょ墝鐢熸垚
  鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€

  [鐢ㄦ埛浜嬩欢/骞挎挱瑙﹀彂]
         鈹?         鈻?  鈹屸攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?  鈹?鐢熸垚娑堟伅妯℃澘娓叉煋  鈹?  鈹?(Liquid 寮曟搸)   鈹?  鈹斺攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?           鈹?           鈹溾攢鈻?鎻愬彇鐢ㄦ埛灞炴€?(userId, email, phone 绛?
           鈹溾攢鈻?鎻愬彇璁㈤槄鍒嗙粍淇℃伅
           鈹斺攢鈻?鑾峰彇璁㈤槄瀵嗛挜 (SecretNames.Subscription)
           鈹?           鈻?  鈹屸攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?  鈹?鐢熸垚璁㈤槄鍝堝笇     鈹?  鈹?HMAC-SHA256     鈹?  鈹?{u, w, i, k}  鈹?  鈹斺攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?           鈹?           鈹?鐢熸垚 URL 鍙傛暟:
           鈹?w={workspaceId}
           鈹?i={identifier}
           鈹?ik={identifierKey}
           鈹?h={hash}
           鈹?s={subscriptionGroupId}
           鈹?sub={1|0}
           鈹?           鈻?  鈹屸攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?  鈹?鎻掑叆閫€璁?绠＄悊閾炬帴 鈹?  鈹?{% unsubscribe_link %} 鈹?  鈹?{% subscription_management_link %} 鈹?  鈹斺攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?           鈹?           鈻?  [娑堟伅鍙戦€佸埌鐢ㄦ埛娓犻亾]
         鈹?         鈹?Email / SMS / MobilePush
         鈻?

  闃舵 2: 鐢ㄦ埛鎿嶄綔涓庝护鐗屾牎楠?  鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€

  [鐢ㄦ埛鐐瑰嚮閫€璁㈤摼鎺
         鈹?         鈻?  鈹屸攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?  鈹?GET /api/public/  鈹?  鈹?subscription-     鈹?  鈹?management/page   鈹?  鈹斺攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?           鈹?           鈹溾攢鈻?鎻愬彇 URL 鍙傛暟
           鈹?  w, i, ik, h, s, sub
           鈹?           鈻?  鈹屸攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?  鈹?鏌ユ壘鐢ㄦ埛 (identifierKey + identifier)鈹?  鈹斺攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?           鈹?           鈹?閬嶅巻鍖归厤鐨勭敤鎴?ID 鍒楄〃
           鈹?           鈻?  鈹屸攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?  鈹?鍝堝笇鏍￠獙            鈹?  鈹?閲嶆柊璁＄畻 HMAC-SHA256 鈹?  鈹?姣斿浼犲叆鐨?h 鍙傛暟     鈹?  鈹斺攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?           鈹?           鈹溾攢鈻?鏍￠獙閫氳繃 鈫?缁х画
           鈹斺攢鈻?鏍￠獙澶辫触 鈫?杩斿洖 401 Unauthorized
           鈹?           鈻?  鈹屸攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?  鈹?澶勭悊璁㈤槄鍙樻洿 (s, sub 鍙傛暟)鈹?  鈹?濡傛灉 sub=0 鈫?閫€璁?   鈹?  鈹?濡傛灉 sub=1 鈫?璁㈤槄    鈹?  鈹斺攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?           鈹?           鈻?  [娓叉煋璁㈤槄绠＄悊椤甸潰]
         鈹?         鈻?

  闃舵 3: 鐢ㄦ埛鍋忓ソ鍐欏洖
  鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€

  [鐢ㄦ埛鍦ㄩ〉闈㈡彁浜ゅ亸濂絔
         鈹?         鈻?  鈹屸攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?  鈹?POST /api/public/ 鈹?  鈹?subscription-      鈹?  鈹?management/page   鈹?  鈹斺攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?           鈹?           鈹溾攢鈻?鎻愬彇琛ㄥ崟鏁版嵁
           鈹?  w, h, i, ik
           鈹?  sub_{sgId} = true/false
           鈹?           鈻?  鈹屸攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?  鈹?浜屾鍝堝笇鏍￠獙        鈹?  鈹?lookupUserForSubscriptions 鈹?  鈹斺攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?           鈹?           鈹溾攢鈻?鏍￠獙閫氳繃 鈫?缁х画
           鈹斺攢鈻?鏍￠獙澶辫触 鈫?杩斿洖 401
           鈹?           鈻?  鈹屸攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?  鈹?updateUserSubscriptions 鈹?  鈹斺攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?           鈹?           鈹溾攢鈻?鏋勫缓 Segment 璧嬪€?           鈹?  涓?Segment: inSegment = isSubscribed
           鈹?  鏈闃?Segment: inSegment = !isSubscribed
           鈹?           鈹溾攢鈻?鏋勫缓璁㈤槄鍙樻洿浜嬩欢
           鈹?  InternalEventType.SubscriptionChange
           鈹?           鈻?  鈹屸攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?  鈹?骞惰鍐欏叆                    鈹?  鈹?鈹溾攢鈻?ClickHouse  鈹?  鈹?鈹?  computed_property_assignments_v2 鈹?  鈹?鈹?  鈹?鈹斺攢鈻?浜嬩欢绯荤粺                 鈹?  鈹?    DFSubscriptionChange 鈹?  鈹斺攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?           鈹?           鈻?  [鍋忓ソ鎸佷箙鍖栧畬鎴怾
         鈹?         鈻?  闃舵 4: 鍚庣画娑堟伅鍙戦€佹鏌?  鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€

  [涓嬩竴娆℃秷鎭彂閫乚
         鈹?         鈻?  鈹屸攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?  鈹?inSubscriptionGroup 鈹?  鈹?妫€鏌ヨ闃呯姸鎬?       鈹?  鈹斺攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?           鈹?           鈹溾攢鈻?宸茶闃?鈫?鍙戦€佹秷鎭?           鈹斺攢鈻?鏈闃?鈫?璺宠繃 (MessageSkipped)
           鈹?           鈻?  [娑堟伅鍙戦€?璺宠繃瀹屾垚]
```

### 6.2 璁㈤槄绠＄悊椤甸潰浜や簰娴佺▼

**GET 璇锋眰澶勭悊璁㈤槄鍙樻洿 (`packages/api/src/controllers/subscriptionManagementController.ts:79-246`)锛?
```typescript
// GET /api/public/subscription-management/page

// 1. 瑙ｆ瀽鏌ヨ鍙傛暟
const {
  w: workspaceId,
  i: identifier,
  ik: identifierKey,
  h: hash,
  s: subscriptionGroupId,  // 璁㈤槄鍒嗙粍 ID
  sub,                   // 1=璁㈤槄, 0=閫€璁?  isPreview,
} = request.query;

// 2. 鐢ㄦ埛鏌ユ壘涓庢牎楠?const [userLookupResult, workspace] = await Promise.all([
  isPreview
    ? null
    : lookupUserForSubscriptions({...}),
  db().query.workspace.findFirst({...}),
]);

// 3. 濡傛灉鎻愪緵浜?s 鍜?sub 鍙傛暟锛屾墽琛岃闃呭彉鏇?if (subscriptionGroupId && sub) {
  subscriptionChange = sub === "1" ? Subscribe : Unsubscribe;
  
  if (!isPreview && targetSubscriptionGroup) {
    if (subscriptionChange === Unsubscribe) {
      // 閫€璁㈣娓犻亾鐨勬墍鏈夎闃呭垎缁?      const channelSubscriptionGroups = await db().query.subscriptionGroup.findMany({
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
      // 璁㈤槄鎸囧畾鐨勮闃呭垎缁?      await updateUserSubscriptions({
        workspaceId,
        userUpdates: [{
          userId,
          changes: { [subscriptionGroupId]: true },
        }],
      });
    }
  }
}

// 4. 鑾峰彇鐢ㄦ埛褰撳墠璁㈤槄鐘舵€?const subscriptions = await getUserSubscriptions({ userId, workspaceId });

// 5. 娓叉煋璁㈤槄绠＄悊椤甸潰
const html = await generateSubscriptionManagementPage({...});
return reply.type("text/html").send(html);
```

**POST 璇锋眰澶勭悊琛ㄥ崟鎻愪氦 (`packages/api/src/controllers/subscriptionManagementController.ts:249-353`)锛?
```typescript
// POST /api/public/subscription-management/page

// 1. 瑙ｆ瀽琛ㄥ崟鏁版嵁
const {
  w: workspaceId,
  h: hash,
  i: identifier,
  ik: identifierKey,
  isPreview,
  // 璁㈤槄澶嶉€夋: sub_{subscriptionGroupId} = "true" 鎴栦笉瀛樺湪
} = typedBody;

// 2. 棰勮妯″紡鐩存帴閲嶅畾鍚?if (isPreview) {
  redirectParams.set("previewSubmitted", "true");
  return reply.redirect(302, `/api/public/subscription-management/page?${redirectParams}`);
}

// 3. 鏍￠獙鐢ㄦ埛韬唤
const userLookupResult = await lookupUserForSubscriptions({...});
if (userLookupResult.isErr()) {
  return reply.status(401).send({ message: "Unauthorized" });
}

const { userId } = userLookupResult.value;

// 4. 鑾峰彇鎵€鏈夎闃呭垎缁勶紝鏋勫缓鍙樻洿瀵硅薄
const subscriptionGroups = await db().query.subscriptionGroup.findMany({
  where: eq(dbSubscriptionGroup.workspaceId, workspaceId),
});

const changes: Record<string, boolean> = {};
for (const sg of subscriptionGroups) {
  const checkboxName = `sub_${sg.id}`;
  const isChecked = typedBody[checkboxName] === "true";
  changes[sg.id] = isChecked;
}

// 5. 鎵ц鏇存柊
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

// 6. 閲嶅畾鍚戝洖椤甸潰锛屾樉绀虹粨鏋?return reply.redirect(302, `/api/public/subscription-management/page?${redirectParams}`);
```

---

## 7. DOI 浠ょ墝瀹夊叏涓庤竟鐣屽垎鏋?
### 7.1 浠ょ墝鏃舵晥涓庝竴娆℃€х害鏉熷垎鏋?
#### 7.1.1 鏄庣‘缁撹

**浠ょ墝鏄惁鏈夋椂鏁堥檺鍒讹紵**
- 鉂?**鏃犳椂鏁堥檺鍒?*銆侱ittofeed 褰撳墠鐨?DOI 浠ょ墝娌℃湁 TTL锛圱ime-To-Live锛夋垨杩囨湡鏃堕棿鏈哄埗銆?
**浠ょ墝鏄惁鏄竴娆℃€х殑锛?*
- 鉂?**涓嶆槸涓€娆℃€х殑**銆備护鐗屽彲浠ヨ閲嶅浣跨敤锛屾病鏈?宸蹭娇鐢?鐘舵€佺殑杩借釜銆?
#### 7.1.2 璇佹嵁鍒嗘瀽

**1. 鍝堝笇鐢熸垚涓嶅寘鍚椂闂存埑** (`packages/backend-lib/src/subscriptionGroups.ts:426-451`)锛?
```typescript
export function generateSubscriptionHash({
  workspaceId,
  userId,
  identifierKey,
  identifier,
  subscriptionSecret,
}: {...}): string {
  const toHash = {
    u: userId,        // 鐢ㄦ埛 ID
    w: workspaceId,   // 宸ヤ綔绌洪棿 ID
    i: identifier,    // 鏍囪瘑绗﹀€?    k: identifierKey, // 鏍囪瘑绗︾被鍨?    // 鉂?娌℃湁 timestamp 鎴?expiry 瀛楁
  };
  
  const hash = generateSecureHash({
    key: subscriptionSecret,
    value: toHash,
  });
  return hash;
}
```

**2. 鏍￠獙鏃朵笉妫€鏌ュ巻鍙茶褰?* (`packages/backend-lib/src/subscriptionGroups.ts:597-659`)锛?
```typescript
export async function lookupUserForSubscriptions({...}): Promise<Result<{ userId: string }, Error>> {
  // 鍙獙璇佸搱甯屾槸鍚︽纭?  // 鉂?娌℃湁鏌ヨ TokenStore 鎴栦娇鐢ㄨ褰?  // 鉂?娌℃湁姣旇緝鏃堕棿鎴?  
  const userId = matchingUserIds.find((uId) => {
    const generatedHash = generateSubscriptionHash({...});
    return hash === generatedHash;  // 鍙鍖归厤灏遍€氳繃
  });
  
  if (!userId) {
    return err(new Error("Invalid hash"));
  }
  return ok({ userId });
}
```

**3. 瀵嗛挜鍒涘缓涓嶄細涓诲姩杞崲** (`packages/backend-lib/src/subscriptionGroups.ts:789-807`)锛?
```typescript
export async function upsertSubscriptionSecret({
  workspaceId,
}: {
  workspaceId: string;
}) {
  return insert({
    table: dbSecret,
    doNothingOnConflict: true,  // 鉁?鏈夊啿绐佹椂涓嶆洿鏂?= 涓嶈疆鎹?    lookupExisting: and(
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

#### 7.1.3 瀹夊叏褰卞搷鐭╅樀

| 鐗规€?| 鐘舵€?| 瀹為檯褰卞搷 | 椋庨櫓绛夌骇 |
|------|------|---------|---------|
| **鏈夋晥鏈?(TTL)** | 鉂?鏃?| 閾炬帴姘镐箙鏈夋晥锛岃娉勯湶鍚庡彲闀挎湡浣跨敤 | 楂?|
| **涓€娆℃€т娇鐢?* | 鉂?鏃?| 閾炬帴鍙噸澶嶇偣鍑伙紝鍙嶅淇敼璁㈤槄鐘舵€?| 涓?|
| **闈炵┖鎬т繚鎶?* | 鉁?鏈?| 蹇呴』鎻愪緵鏈夋晥鐨?identifier銆乮dentifierKey銆乭ash | 浣?|
| **鍝堝笇瀹屾暣鎬?* | 鉁?鏈?| HMAC-SHA256 淇濊瘉鏈绡℃敼 | 浣?|
| **瀵嗛挜杞崲** | 鉂?涓嶆敮鎸?| 鎵嬪姩鏇存柊浼氱珛鍗冲け鏁堟墍鏈夊巻鍙查摼鎺?| 楂?|

#### 7.1.4 瀹為檯椋庨櫓鍦烘櫙

**鍦烘櫙 1锛氱敤鎴疯浆鍙戦偖浠剁粰鏈嬪弸**
```
鐢ㄦ埛 A 鏀跺埌钀ラ攢閭欢
  鈫?鐢ㄦ埛 A 鎶婇偖浠惰浆鍙戠粰鏈嬪弸 B
  鈫?鏈嬪弸 B 鐐瑰嚮 {% unsubscribe_link %}
  鈫?绯荤粺鏍￠獙閫氳繃锛坔ash 缁戝畾 userId锛屼笉鏄?email 鎵€鏈夎€呰韩浠斤級
  鈫?鏈嬪弸 B 鍙互閫€璁㈢敤鎴?A 鐨勬墍鏈夎闃?```
- **椋庨櫓绛夌骇**锛氶珮
- **鍘熷洜**锛歎RL 涓笉鍖呭惈鐢ㄦ埛韬唤楠岃瘉鏈哄埗锛屽彧鏈夊搱甯岄獙璇?
**鍦烘櫙 2锛氶偖浠跺綊妗?3 骞村悗**
```
2023 骞村彂閫佺殑閭欢琚綊妗?  鈫?2026 骞寸敤鎴蜂粠褰掓。涓壘鍥?  鈫?鐐瑰嚮閫€璁㈤摼鎺?鈫?浠嶇劧鏈夋晥 鉁?  鈫?鐢ㄦ埛鍙互淇敼璁㈤槄鐘舵€?```
- **椋庨櫓绛夌骇**锛氫腑锛堝姛鑳借璁★紝闈炲畨鍏ㄦ紡娲烇級
- **璇存槑**锛氬鏋滅敤鎴峰笇鏈涙案涔呴€€璁紝杩欏弽鑰屾槸涓€涓紭鐐?
**鍦烘櫙 3锛氭敾鍑昏€呮埅鑾?URL**
```
鏀诲嚮鑰呴€氳繃涓棿浜烘敾鍑绘埅鑾烽偖浠?  鈫?鑾峰彇閫€璁㈤摼鎺?URL
  鈫?鏀诲嚮鑰呭彲浠ユ棤闄愭浣跨敤璇?URL
  鈫?鏀诲嚮鑰呭彲浠ラ殢鏃朵慨鏀圭敤鎴疯闃呭亸濂?```
- **椋庨櫓绛夌骇**锛氶珮
- **鍚庢灉**锛氭敾鍑昏€呭彲鍙嶅璁㈤槄/閫€璁㈢敤鎴?
---

### 7.2 閾炬帴琚噸鏀惧悗绯荤粺瀹為檯琛屼负

#### 7.2.1 閲嶆斁鏀诲嚮瀹氫箟

**閲嶆斁鏀诲嚮**锛氭敾鍑昏€呰幏鍙栧埌鐢ㄦ埛鐨勯€€璁㈤摼鎺ュ悗锛岄噸澶嶇偣鍑绘垨妯℃嫙璇锋眰锛岃瘯鍥炬敼鍙樼敤鎴风姸鎬併€?
#### 7.2.2 GET 璇锋眰閲嶆斁锛堢偣鍑婚€€璁㈤摼鎺ワ級

**瀹屾暣鏃堕棿绾垮垎鏋?*锛?
```
鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣
T0: 鍒濆鐘舵€?鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣
绯荤粺鐘舵€侊細
  - 鐢ㄦ埛 A锛氳闃呬簡 Marketing Emails
  - ClickHouse: inSegment = true
  - 鏈闃?Segment: inSegment = false

鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣
T1: 鐢ㄦ埛绗竴娆＄偣鍑婚€€璁㈤摼鎺ワ紙姝ｅ父鎿嶄綔锛?鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣
GET /api/public/subscription-management/page
  ?w={workspaceId}
  &i=user@example.com
  &ik=email
  &h={valid_hash}
  &s={subscriptionGroupId}
  &sub=0  鈫?閫€璁?
鎵ц娴佺▼锛?  1. lookupUserForSubscriptions() 鈫?鏍￠獙閫氳繃 鉁?  2. 鎵惧埌 subscriptionGroup锛岀‘瀹?channel = Email
  3. 閫€璁㈣ channel 鐨勬墍鏈夎闃呭垎缁?  4. updateUserSubscriptions() 鎵ц锛?     - 鍐欏叆 ClickHouse: inSegment = false
     - 璁板綍 DFSubscriptionChange 浜嬩欢 (action: Unsubscribe)
  5. 娓叉煋椤甸潰锛屾樉绀?宸查€€璁?

绯荤粺鐘舵€佸彉鍖栵細
  - 鐢ㄦ埛 A锛氬凡閫€璁?Marketing Emails
  - ClickHouse: inSegment = false
  - 鏈闃?Segment: inSegment = true
  - 浜嬩欢鏃ュ織锛氭柊澧?1 鏉?DFSubscriptionChange

鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣
T2: 鏀诲嚮鑰呯涓€娆￠噸鏀?GET 璇锋眰
鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣
GET /api/public/subscription-management/page锛堢浉鍚?URL锛?
鎵ц娴佺▼锛?  1. lookupUserForSubscriptions() 鈫?鏍￠獙閫氳繃 鉁?     锛堝洜涓?hash 浠嶇劧鏈夋晥锛屾病鏈変娇鐢ㄨ褰曪級
  2. 鎵惧埌 subscriptionGroup锛岀‘瀹?channel = Email
  3. 閫€璁㈣ channel 鐨勬墍鏈夎闃呭垎缁?  4. updateUserSubscriptions() 鍐嶆鎵ц锛?     - 鍐欏叆 ClickHouse: inSegment = false锛堝箓绛夛紝鍊间笉鍙橈級
     - 鍐嶆璁板綍 DFSubscriptionChange 浜嬩欢 (action: Unsubscribe)
  5. 娓叉煋椤甸潰锛屾樉绀?宸查€€璁?

绯荤粺鐘舵€佸彉鍖栵細
  - 鐢ㄦ埛 A锛氬凡閫€璁紙鐘舵€佷笉鍙橈級
  - ClickHouse: inSegment = false锛堝€间笉鍙橈級
  - 鏈闃?Segment: inSegment = true锛堝€间笉鍙橈級
  - 浜嬩欢鏃ュ織锛氭柊澧炵 2 鏉?DFSubscriptionChange 鈫?鈿狅笍 閲嶅浜嬩欢

鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣
T3: 鐢ㄦ埛閫氳繃鍏朵粬閫斿緞閲嶆柊璁㈤槄
鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣
鐢ㄦ埛璁块棶璁㈤槄绠＄悊椤甸潰锛屾墜鍔ㄥ嬀閫?Marketing Emails

绯荤粺鐘舵€佸彉鍖栵細
  - 鐢ㄦ埛 A锛氬凡閲嶆柊璁㈤槄
  - ClickHouse: inSegment = true
  - 鏈闃?Segment: inSegment = false
  - 浜嬩欢鏃ュ織锛氭柊澧?DFSubscriptionChange (action: Subscribe)

鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣
T4: 鏀诲嚮鑰呯浜屾閲嶆斁 GET 璇锋眰
鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣
GET /api/public/subscription-management/page锛堢浉鍚?URL锛?
鎵ц娴佺▼锛?  1. lookupUserForSubscriptions() 鈫?鏍￠獙閫氳繃 鉁?  2. 鎵惧埌 subscriptionGroup锛岀‘瀹?channel = Email
  3. 閫€璁㈣ channel 鐨勬墍鏈夎闃呭垎缁?  4. updateUserSubscriptions() 鎵ц锛?     - 鍐欏叆 ClickHouse: inSegment = false 鈫?鈿狅笍 鐢ㄦ埛琚啀娆￠€€璁紒
     - 璁板綍 DFSubscriptionChange 浜嬩欢 (action: Unsubscribe)

绯荤粺鐘舵€佸彉鍖栵細
  - 鐢ㄦ埛 A锛氬啀娆¤閫€璁?鈫?鈿狅笍 闈為鏈熺姸鎬佸彉鏇?  - ClickHouse: inSegment = false
  - 鏈闃?Segment: inSegment = true
  - 浜嬩欢鏃ュ織锛氭柊澧炵 3 鏉?DFSubscriptionChange
鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣
```

**GET 閲嶆斁鐨勫疄闄呰涓烘€荤粨**锛?
| 鏃舵満 | 绯荤粺琛屼负 | 鐢ㄦ埛鐘舵€?| 浜嬩欢鏃ュ織 |
|------|---------|---------|---------|
| 鐢ㄦ埛閫€璁㈠悗绔嬪嵆閲嶆斁 | 閫€璁㈡搷浣滃箓绛?| 涓嶅彉 | 閲嶅浜嬩欢 |
| 鐢ㄦ埛閲嶆柊璁㈤槄鍚庨噸鏀?| 鍐嶆閫€璁?| 琚敼鍙?鈿狅笍 | 鏂板浜嬩欢 |

#### 7.2.3 POST 璇锋眰閲嶆斁锛堟彁浜ゅ亸濂借〃鍗曪級

**POST 璇锋眰鐨勭壒娈婃€?*锛?
```typescript
// packages/api/src/controllers/subscriptionManagementController.ts:249-353
// POST /api/public/subscription-management/page

// 鏋勫缓 changes 瀵硅薄浠庤〃鍗曟暟鎹?const changes: Record<string, boolean> = {};
for (const sg of subscriptionGroups) {
  const checkboxName = `sub_${sg.id}`;
  const isChecked = typedBody[checkboxName] === "true";
  changes[sg.id] = isChecked;  // 鈫?浣跨敤鎻愪氦鏃剁殑琛ㄥ崟蹇収
}

// 浣跨敤鎻愪氦鏃剁殑琛ㄥ崟鏁版嵁锛岃€岄潪褰撳墠鐘舵€?await updateUserSubscriptions({
  workspaceId,
  userUpdates: [{ userId, changes }],
});
```

**POST 閲嶆斁瀹屾暣鏃堕棿绾?*锛?
```
鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣
T0: 鍒濆鐘舵€?鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣
璁㈤槄鍒嗙粍鐘舵€侊細
  - SG1 (Marketing): 宸茶闃?鉁?  - SG2 (Newsletter): 宸茶闃?鉁?  - SG3 (Promotions): 宸查€€璁?鉁?
鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣
T1: 鐢ㄦ埛鎻愪氦琛ㄥ崟锛堟敾鍑昏€呮埅鑾疯姹傦級
鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣
POST /api/public/subscription-management/page
Form Data:
  w={workspaceId}
  h={valid_hash}
  i=user@example.com
  ik=email
  sub_SG1=true
  sub_SG2=true
  sub_SG3=false  鈫?鐢ㄦ埛鎻愪氦鏃剁殑閫夋嫨

changes = { SG1: true, SG2: true, SG3: false }

绯荤粺鐘舵€佸彉鍖栵細
  - SG1: 宸茶闃?鉁?  - SG2: 宸茶闃?鉁?  - SG3: 宸查€€璁?鉁?
鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣
T2: 鐢ㄦ埛閫氳繃 API 閫€璁?SG1
鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣
PUT /api/public/subscription-management/user-subscriptions
{
  workspaceId: "...",
  identifier: "user@example.com",
  identifierKey: "email",
  hash: "...",
  changes: { SG1: false }  鈫?鐢ㄦ埛閫€璁?SG1
}

绯荤粺鐘舵€佸彉鍖栵細
  - SG1: 宸查€€璁?鉁? 鈫?鍙樻洿
  - SG2: 宸茶闃?鉁?  - SG3: 宸查€€璁?鉁?
鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣
T3: 鏀诲嚮鑰呴噸鏀?POST 璇锋眰
鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣
POST /api/public/subscription-management/page锛堢浉鍚岃〃鍗曟暟鎹級

Form Data 涓殑 changes = { SG1: true, SG2: true, SG3: false }

绯荤粺鐘舵€佸彉鍖栵細
  - SG1: 宸茶闃?鉁? 鈫?鈿狅笍 琚噸鏂拌闃咃紒
  - SG2: 宸茶闃?鉁?  - SG3: 宸查€€璁?鉁?
鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣
```

**POST 閲嶆斁鐨勯闄╃瓑绾?*锛氶珮
- **鍘熷洜**锛氫娇鐢ㄨ〃鍗曟彁浜ゆ椂鐨勫揩鐓э紝鑰岄潪褰撳墠鐘舵€?- **鍚庢灉**锛氭敾鍑昏€呭彲灏嗗凡閫€璁㈢殑鍒嗙粍閲嶆柊璁㈤槄

#### 7.2.4 浜嬩欢鏃ュ織鐗瑰緛

姣忔閲嶆斁閮戒細浜х敓鏂扮殑 `DFSubscriptionChange` 浜嬩欢锛?
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
    userEvents: allUserEvents,  // 鈫?姣忔閮芥彃鍏ユ柊浜嬩欢
  }),
]);
```

**閲嶆斁妫€娴嬫寚鏍?*锛?- 鐩戞帶鐭椂闂村唴鍚屼竴鐢ㄦ埛鍚屼竴璁㈤槄鍒嗙粍鐨勫娆?`DFSubscriptionChange` 浜嬩欢
- 鐩戞帶 `Unsubscribe 鈫?Subscribe 鈫?Unsubscribe` 杩欐牱鐨勫揩閫熺炕杞簭鍒?
#### 7.2.5 骞傜瓑鎬у姣?
| 鎿嶄綔绫诲瀷 | GET 閫€璁?| GET 璁㈤槄 | POST 琛ㄥ崟 |
|---------|---------|---------|----------|
| **骞傜瓑鎬?* | 鉁?骞傜瓑锛堥噸澶嶉€€璁?閫€璁級 | 鈿狅笍 鏉′欢骞傜瓑锛堥噸澶嶈闃?璁㈤槄锛?| 鉂?闈炲箓绛?|
| **鐘舵€佷緷鎹?* | URL 涓殑 sub 鍙傛暟 | URL 涓殑 sub 鍙傛暟 | 琛ㄥ崟鎻愪氦鏃剁殑蹇収 |
| **閲嶆斁褰卞搷** | 鐘舵€佷笉鍙橈紝浜х敓閲嶅浜嬩欢 | 鐘舵€佷笉鍙橈紝浜х敓閲嶅浜嬩欢 | 鍙兘瀵艰嚧闈為鏈熺姸鎬佸彉鏇?|
| **椋庨櫓绛夌骇** | 浣?| 浣?| 楂?|

---

### 7.3 璁㈤槄瀵嗛挜杞崲鍚庢棫閾炬帴鐨勬牎楠屼笌澶辨晥

#### 7.3.1 褰撳墠瀵嗛挜绠＄悊鏈哄埗

**瀵嗛挜鍒涘缓娴佺▼**锛?
```
宸ヤ綔绌洪棿鍒濆鍖?  鈫?bootstrapPostgres() 璋冪敤
  鈫?upsertSubscriptionSecret({ workspaceId })
  鈫?鏌ヨ鏄惁宸叉湁 SecretNames.Subscription
  鈹溾攢鈻?瀛樺湪 鈫?doNothingOnConflict 鈫?淇濇寔涓嶅彉
  鈹斺攢鈻?涓嶅瓨鍦?鈫?鐢熸垚鏂板瘑閽?generateSecureKey(8)
                鈫?              鎻掑叆 Secret 琛?```

**浠ｇ爜璇佹嵁** (`packages/backend-lib/src/subscriptionGroups.ts:789-807`)锛?```typescript
export async function upsertSubscriptionSecret({
  workspaceId,
}: {
  workspaceId: string;
}) {
  return insert({
    table: dbSecret,
    doNothingOnConflict: true,  // 鈫?鍏抽敭鐐癸細鏈夊啿绐佷笉鏇存柊
    lookupExisting: and(
      eq(dbSecret.workspaceId, workspaceId),
      eq(dbSecret.name, SecretNames.Subscription),
    )!,
    values: {
      workspaceId,
      name: SecretNames.Subscription,
      value: generateSecureKey(8),  // 16 瀛楃闅忔満鍗佸叚杩涘埗
    },
  }).then(unwrap);
}
```

**缁撹**锛氬綋鍓嶇郴缁?*娌℃湁**鍐呯疆鐨勫瘑閽ヨ疆鎹㈡満鍒躲€?
#### 7.3.2 瀵嗛挜杞崲鍚庢棫閾炬帴澶辨晥鐨勬牴鏈師鍥?
**鍝堝笇璁＄畻鍘熺悊**锛?
```
鍝堝笇 = HMAC-SHA256(瀵嗛挜, {userId, workspaceId, identifier, identifierKey})
```

**瀵嗛挜杞崲鍚庣殑鏍￠獙澶辫触娴佺▼**锛?
```
鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣
闃舵 1锛氬師閾炬帴鐢熸垚鏃讹紙浣跨敤鏃у瘑閽ワ級
鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣

key_old = "abc123_old_secret"

hash_old = HMAC-SHA256(key_old, {
  u: "user-123",
  w: "workspace-456",
  i: "user@example.com",
  k: "email"
})

hash_old = "a1b2c3d4e5f6..."

鐢熸垚鐨?URL锛?  /api/public/subscription-management/page
  ?w=workspace-456
  &i=user@example.com
  &ik=email
  &h=a1b2c3d4e5f6...  鈫?浣跨敤鏃у瘑閽ヨ绠楃殑鍝堝笇

鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣
闃舵 2锛氭墜鍔ㄦ洿鏂?Secret 琛紙瀵嗛挜杞崲锛?鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣

绠＄悊鍛樻墽琛岋細
  UPDATE Secret
  SET value = "xyz789_new_secret"
  WHERE name = 'subscription' AND workspaceId = 'workspace-456'

鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣
闃舵 3锛氱敤鎴风偣鍑绘棫閾炬帴锛堟牎楠屽け璐ワ級
鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣

鐢ㄦ埛鐐瑰嚮鏃ч摼鎺ワ細
  GET /api/public/subscription-management/page
  ?w=workspace-456
  &i=user@example.com
  &ik=email
  &h=a1b2c3d4e5f6...  鈫?URL 涓殑鏃у搱甯?
lookupUserForSubscriptions() 鎵ц锛?  鈹溾攢鈻?浠庢暟鎹簱鑾峰彇褰撳墠瀵嗛挜
  鈹?    key_new = "xyz789_new_secret"  鈫?鏂板瘑閽ワ紒
  鈹?  鈹溾攢鈻?鎵惧埌鐢ㄦ埛 userId = "user-123"
  鈹?  鈹斺攢鈻?鐢ㄦ柊瀵嗛挜閲嶆柊璁＄畻鍝堝笇锛?        hash_new = HMAC-SHA256(key_new, {
          u: "user-123",
          w: "workspace-456",
          i: "user@example.com",
          k: "email"
        })
        hash_new = "x9y8z7w6v5u4..."  鈫?涓庢棫鍝堝笇涓嶅悓锛?  鈹?  鈹溾攢鈻?姣斿锛歨ash_old === hash_new?
  鈹?    "a1b2c3d4e5f6..." === "x9y8z7w6v5u4..."  鈫? FALSE
  鈹?  鈹斺攢鈻?杩斿洖 Error("Invalid hash")

鎺у埗鍣ㄥ鐞嗭細
  if (userLookupResult.isErr()) {
    return reply.status(401).send({ message: "Unauthorized" });
  }

鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣
```

#### 7.3.3 浠ｇ爜灞傞潰鐨勬牎楠屾祦绋?
**鏍￠獙鏃跺彧鍙栧綋鍓嶅瘑閽?* (`packages/backend-lib/src/subscriptionGroups.ts:603-615`)锛?
```typescript
const [subscriptionSecret, matchingUserIds] = await Promise.all([
  db().query.secret.findFirst({
    where: and(
      eq(dbSecret.workspaceId, workspaceId),
      eq(dbSecret.name, SecretNames.Subscription),  // 鈫?鍙煡褰撳墠瀵嗛挜
    ),
    // 鉂?娌℃湁鍘嗗彶瀵嗛挜鐨勬蹇?    // 鉂?娌℃湁鏌ヨ澶氫釜瀵嗛挜鐨勯€昏緫
  }),
  findUserIdsByUserPropertyValue({...}),
]);
```

**浣跨敤褰撳墠鍞竴瀵嗛挜杩涜鏍￠獙** (`packages/backend-lib/src/subscriptionGroups.ts:628-644`)锛?
```typescript
const secretValue = subscriptionSecret?.value;
// secretValue = 褰撳墠鍞竴鐨勫瘑閽?
const userId = matchingUserIds.find((uId) => {
  const generatedHash = generateSubscriptionHash({
    workspaceId,
    userId: uId,
    identifierKey,
    identifier,
    subscriptionSecret: secretValue,  // 鈫?鍙娇鐢ㄥ綋鍓嶅瘑閽?  });
  return hash === generatedHash;
});

if (!userId) {
  return err(new Error("Invalid hash"));  // 鈫?鏃ч摼鎺ュ叏閮ㄥけ鏁?}
```

#### 7.3.4 瀵嗛挜杞崲鐨勫奖鍝嶇煩闃?
| 鍦烘櫙 | 褰撳墠琛屼负 | 鐢ㄦ埛褰卞搷 | 涓氬姟褰卞搷 |
|------|---------|---------|---------|
| **鎵嬪姩鏇存柊 Secret 琛?* | 鎵€鏈夋棫閾炬帴绔嬪嵆澶辨晥 | 鐢ㄦ埛鏀跺埌 401 Unauthorized | 鍘嗗彶閭欢閫€璁㈠姛鑳戒笉鍙敤 |
| **宸ヤ綔绌洪棿杩佺Щ** | 鏂板瘑閽ョ敓鎴愶紝鏃ч摼鎺ュけ鏁?| 闇€瑕侀噸鏂板彂閫佽闃呯‘璁ら偖浠?| 杩佺Щ鎴愭湰楂?|
| **瀵嗛挜娉勯湶鍚庣揣鎬ヨ疆鎹?* | 姝ｇ‘琛屼负锛屼絾褰卞搷澶?| 鎵€鏈夊巻鍙查偖浠剁殑閫€璁㈤摼鎺ュけ鏁?| 瀹夊叏浼樺厛锛屼絾鐢ㄦ埛浣撻獙宸?|
| **蹇樿鏃у瘑閽?* | 鏃犳硶鎭㈠鏃ч摼鎺?| 姘镐箙澶辨晥 | 鏁版嵁涓㈠け |

#### 7.3.5 寤鸿鐨勬敼杩涙柟妗堬細鍙屽瘑閽ヨ繃娓℃満鍒?
**璁捐鎬濊矾**锛?
```
瀵嗛挜杞崲鏃讹細
  1. 淇濈暀鏃у瘑閽ワ紙鏍囪涓?previous锛?  2. 鐢熸垚鏂板瘑閽ワ紙鏍囪涓?active锛?  3. 鏍￠獙鏃跺厛璇?active锛屽啀璇?previous
  4. 杩囨浮鏈熷悗锛堝 90 澶╋級鍒犻櫎 previous
```

**寤鸿鐨勪唬鐮佸疄鐜?*锛?
```typescript
// 鏂板鏋氫妇鎴栧父閲?enum SecretNames {
  Subscription = "subscription",
  SubscriptionPrevious = "subscription-previous",  // 鏂板
}

// 鏀硅繘鐨勬牎楠屽嚱鏁?export async function lookupUserForSubscriptions({...}): Promise<Result<{ userId: string }, Error>> {
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
        // 鍏堝皾璇?active锛屽啀灏濊瘯 previous
        sql.raw(`CASE name WHEN '${SecretNames.Subscription}' THEN 0 ELSE 1 END`),
      ],
    }),
    findUserIdsByUserPropertyValue({...}),
  ]);

  if (!matchingUserIds || matchingUserIds.length === 0) {
    return err(new Error("User not found"));
  }

  // 閬嶅巻鎵€鏈夊瘑閽ヨ繘琛屾牎楠?  for (const secret of secrets) {
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
      // 鍙€夛細濡傛灉浣跨敤鐨勬槸 previous 瀵嗛挜锛岃褰曡鍛婃棩蹇?      if (secret.name === SecretNames.SubscriptionPrevious) {
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

**鍙屽瘑閽ユ満鍒剁殑浼樼偣**锛?- 瀵嗛挜杞崲鏃舵棫閾炬帴涓嶄細绔嬪嵆澶辨晥
- 鏈夊厖瓒崇殑杩囨浮鏈熼€氱煡鐢ㄦ埛鏇存柊鍋忓ソ
- 鍙互閫愭娣樻卑鏃ч摼鎺?
---

### 7.4 瀹夊叏杈圭晫鎬荤粨

| 缁村害 | 鐜扮姸 | 椋庨櫓绛夌骇 | 鏀硅繘寤鸿 |
|------|------|---------|---------|
| **浠ょ墝鏃舵晥** | 鏃犻檺鍒?| 楂?| 娣诲姞 TTL锛屽 30-90 澶?|
| **涓€娆℃€т娇鐢?* | 涓嶆敮鎸?| 涓?| 璁板綍 Token 浣跨敤鐘舵€侊紙鍙€夛級 |
| **閲嶆斁淇濇姢** | 鏃?| 涓?| 渚濊禆涓氬姟骞傜瓑鎬э紝鐩戞帶寮傚父浜嬩欢 |
| **瀵嗛挜杞崲** | 涓嶆敮鎸侊紙鎵嬪姩鏇存柊浼氬け鏁堟墍鏈夐摼鎺ワ級 | 楂?| 瀹炵幇鍙屽瘑閽ヨ繃娓℃満鍒?|
| **鍝堝笇绠楁硶** | HMAC-SHA256 | 浣庯紙瀹夊叏锛?| 鉁?褰撳墠瀹炵幇鑹ソ |
| **瀵嗛挜闀垮害** | 64 浣?(8 bytes) | 浣庯紙鍙帴鍙楋級 | 鍙鍔犲埌 256 浣?|
| **璺ㄦ笭閬撴牎楠?* | 鍩轰簬 userId 缁戝畾 | 浣庯紙璁捐鍚堢悊锛?| 鉁?褰撳墠瀹炵幇鑹ソ |

---

## 8. 璺ㄦ笭閬撴牎楠屽紓甯稿垎鏀椂搴忛摼

### 8.1 鏍￠獙鍏ュ彛姒傝堪

**鏍稿績鏍￠獙鍑芥暟**锛歚lookupUserForSubscriptions()`

**杈撳叆鍙傛暟**锛?```typescript
interface UserSubscriptionLookup {
  workspaceId: string;    // w 鍙傛暟
  identifier: string;     // i 鍙傛暟
  identifierKey: string;  // ik 鍙傛暟
  hash: string;           // h 鍙傛暟
}
```

**鏍￠獙娴佺▼鐨勪袱涓叧閿楠?*锛?```
姝ラ 1锛歠indUserIdsByUserPropertyValue()
  鈹斺攢鈻?鏍规嵁 identifierKey + identifier 鏌ユ壘鍖归厤鐨勭敤鎴?ID 鍒楄〃

姝ラ 2锛氬搱甯屾牎楠?  鈹斺攢鈻?閬嶅巻鍖归厤鐨勭敤鎴?ID锛岀敤姣忎釜 userId 閲嶆柊璁＄畻鍝堝笇
  鈹斺攢鈻?鎵惧埌鍖归厤鐨?userId 鎴栬繑鍥為敊璇?```

---

### 8.2 寮傚父鍒嗘敮 1锛氱敤鎴蜂笉瀛樺湪锛堝悓鏍囪瘑鍛戒腑 0 鐢ㄦ埛锛?
#### 8.2.1 瑙﹀彂鏉′欢

`identifierKey + identifier` 缁勫悎鍦ㄧ郴缁熶腑鎵句笉鍒颁换浣曠敤鎴枫€?
**鍙兘鐨勫師鍥?*锛?1. 鐢ㄦ埛宸茶鍒犻櫎
2. 鎵归噺閭欢涓殑鐢ㄦ埛鏁版嵁宸茶繃鏈?3. URL 鍙傛暟琚鏀?4. 鐢ㄦ埛灞炴€у€艰緭鍏ラ敊璇?
#### 8.2.2 瀹屾暣鏃跺簭閾?
```
鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣
T0: 閾炬帴琚偣鍑?鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣
GET /api/public/subscription-management/page
  ?w=workspace-abc
  &i=unknown@example.com  鈫?杩欎釜閭涓嶅瓨鍦?  &ik=email
  &h=any_hash_value

鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣
T1: 鎺у埗鍣ㄥ眰鍏ュ彛
鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣鈹佲攣
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


```
