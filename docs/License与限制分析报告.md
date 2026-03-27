# FastGPT License 与用户限制分析报告

> 分析日期: 2026-03-26
> 项目版本: FastGPT 4.2.3

***

## 一、License 验证机制分析

### 1.1 License 数据结构

从 `packages/global/common/system/types/index.ts` 中定义的 `LicenseDataType`:

```typescript
export type LicenseDataType = {
  startTime: string;
  expiredTime: string;
  company: string;
  description?: string;
  hosts?: string[];           // 管理端有效域名
  maxUsers?: number;          // 最大用户数
  maxApps?: number;           // 最大应用数
  maxDatasets?: number;       // 最大数据集数
  functions: {
    sso: boolean;
    pay: boolean;
    customTemplates: boolean;
    datasetEnhance: boolean;
    batchEval: boolean;
  };
};
```

### 1.2 License 验证流程

**关键代码位置**: `packages/service/common/system/config/controller.ts`

```typescript
export const getFastGPTConfigFromDB = async () => {
  if (!FastGPTProUrl) {
    // 如果没有配置 PRO_URL，直接返回空配置
    return { fastgptConfig: {} as FastGPTConfigFileType };
  }
  // 从数据库获取 license 配置
  const licenseData = licenseConfig?.value?.data as LicenseDataType;
  return { fastgptConfig: config, licenseData };
};
```

**核心判断**: `global.licenseData` 决定是否启用商业功能

### 1.3 商业功能开关

从 `projects/app/src/service/common/system/index.ts`:

```typescript
isPlus: !!licenseData,  // 有 license 则为 true
show_dataset_enhance: licenseData?.functions?.datasetEnhance,
show_batch_eval: licenseData?.functions?.batchEval,
hideChatCopyrightSetting: process.env.HIDE_CHAT_COPYRIGHT_SETTING === 'true',
```

***

## 二、用户数量限制机制

### 2.1 License 级别限制

| 限制项           | 代码位置           | 说明                 |
| ------------- | -------------- | ------------------ |
| `maxUsers`    | `teamLimit.ts` | License 级别的最大用户数限制 |
| `maxApps`     | `teamLimit.ts` | 全局应用数限制            |
| `maxDatasets` | `teamLimit.ts` | 全局知识库数限制           |

### 2.2 License 限制代码

从 `packages/service/support/permission/teamLimit.ts`:

```typescript
// 应用数量 License 限制
if (global?.licenseData?.maxApps && typeof global?.licenseData?.maxApps === 'number') {
  const totalApps = await MongoApp.countDocuments({...});
  if (totalApps > global.licenseData.maxApps) {
    return Promise.reject(SystemErrEnum.licenseAppAmountLimit);
  }
}

// 知识库数量 License 限制
if (global?.licenseData?.maxDatasets && typeof global?.licenseData?.maxDatasets === 'number') {
  const totalDatasets = await MongoDataset.countDocuments({...});
  if (totalDatasets >= global.licenseData.maxDatasets) {
    return Promise.reject(SystemErrEnum.licenseDatasetAmountLimit);
  }
}
```

### 2.3 当前状态

**关键发现**: 如果 `PRO_URL` 环境变量未设置，则 `licenseData` 为 `undefined`，**不会触发 License 级别的限制检查**。

```typescript
// packages/service/common/system/constants.ts
export const FastGPTProUrl = process.env.PRO_URL ? `${process.env.PRO_URL}/api` : '';
```

***

## 三、套餐(Plan)级别限制

### 3.1 套餐类型

套餐限制定义在 `packages/global/support/wallet/sub/type.ts`:

```typescript
export type StandSubPlanType = {
  totalPoints: number;         // 总积分
  maxTeamMember: number;       // 最大团队成员数
  maxAppAmount: number;         // 最大应用数
  maxDatasetAmount: number;     // 最大知识库数
  maxDatasetSize: number;       // 最大知识库大小
  requestsPerMinute: number;    // 每分钟请求数
  chatHistoryStoreDuration: number;  // 聊天历史存储时长
  websiteSyncPerDataset: number;    // 每个知识库的网站同步数
};
```

### 3.2 免费套餐默认值

从 `packages/service/support/wallet/sub/utils.ts`:

```typescript
const freePoints = isWecomTeam
  ? Math.round((global.subPlans?.standard?.basic.totalPoints ?? 4000) / 2)
  : global?.subPlans?.standard?.[StandardSubLevelEnum.free]?.totalPoints || 100;
```

**默认值**: 免费套餐有 **100 积分**

### 3.3 套餐限制函数

从 `packages/service/support/permission/teamLimit.ts`:

| 函数                       | 限制项       | 错误码                                  |
| ------------------------ | --------- | ------------------------------------ |
| `checkTeamAIPoints`      | AI 积分不足   | `TeamErrEnum.aiPointsNotEnough`      |
| `checkTeamMemberLimit`   | 团队成员数超限   | `TeamErrEnum.teamOverSize`           |
| `checkTeamAppTypeLimit`  | 应用数量超限    | `TeamErrEnum.appAmountNotEnough`     |
| `checkTeamDatasetLimit`  | 知识库数量超限   | `TeamErrEnum.datasetAmountNotEnough` |
| `checkDatasetIndexLimit` | 知识库索引大小超限 | `TeamErrEnum.datasetSizeNotEnough`   |

***

## 四、当前项目限制状态

### 4.1 开发环境配置

检查当前 `.env` 配置:

| 配置项                           | 当前值     | 说明             |
| ----------------------------- | ------- | -------------- |
| `PRO_URL`                     | **未设置** | License 验证服务地址 |
| `HIDE_CHAT_COPYRIGHT_SETTING` | `true`  | 隐藏版权设置         |

### 4.2 实际限制情况

由于 `PRO_URL` 未设置，当前运行状态为:

| 限制类型              | 状态      | 说明                   |
| ----------------- | ------- | -------------------- |
| **License 用户数限制** | ❌ 不生效   | `maxUsers` 检查不会触发    |
| **License 应用数限制** | ❌ 不生效   | `maxApps` 检查不会触发     |
| **License 知识库限制** | ❌ 不生效   | `maxDatasets` 检查不会触发 |
| **套餐积分限制**        | ✅ 生效    | 免费套餐 100 积分          |
| **套餐应用数限制**       | ⚠️ 可能生效 | 取决于 `subPlans` 配置    |
| **套餐知识库限制**       | ⚠️ 可能生效 | 取决于 `subPlans` 配置    |

### 4.3 套餐配置位置

套餐配置通常在 `config.json` 或通过 `subPlans` 全局变量提供:

```typescript
// packages/service/type.ts
var subPlans: SubPlanType | undefined;
var licenseData: LicenseDataType | undefined;
```

***

## 五、解除限制的方法

### 5.1 方案一：设置 PRO\_URL

1. 部署 FastGPT Pro License 服务
2. 在 `.env` 中设置:
   ```bash
   PRO_URL=https://your-license-server.com
   ```

**效果**: 启用 License 验证，可配置 `maxUsers`, `maxApps`, `maxDatasets`

### 5.2 方案二：配置环境变量隐藏版权

```bash
HIDE_CHAT_COPYRIGHT_SETTING=true
```

**效果**: 隐藏聊天界面中的版权信息

### 5.3 方案三：修改套餐默认限制

编辑 `packages/service/support/wallet/sub/utils.ts`:

```typescript
const freePoints = isWecomTeam
  ? Math.round((global.subPlans?.standard?.basic.totalPoints ?? 4000) / 2)
  : global?.subPlans?.standard?.[StandardSubLevelEnum.free]?.totalPoints || 100; // 改为更大的值
```

### 5.4 方案四：修改 License 检查逻辑

注释掉 `packages/service/support/permission/teamLimit.ts` 中的 License 检查:

```typescript
// 注释掉这部分代码
// if (global?.licenseData?.maxApps && typeof global?.licenseData?.maxApps === 'number') {
//   ...
// }
```

***

## 六、限制绕过分析

### 6.1 理论上的绕过方法

| 方法                         | 风险   | 说明       |
| -------------------------- | ---- | -------- |
| 设置 `PRO_URL` 为空或假地址        | ⚠️ 中 | 可能导致服务异常 |
| 修改 `licenseData` 全局变量      | ⚠️ 中 | 需要重启服务   |
| 注释 License 检查代码            | ⚠️ 高 | 违反开源协议   |
| 修改 `initTeamFreePlan` 增加积分 | ⚠️ 中 | 仅影响免费套餐  |

<br />

***

## 七、当前环境实际限制值

### 7.1 无 PRO\_URL 配置时的行为

根据代码分析，当 `PRO_URL` 未设置时:

```typescript
if (!FastGPTProUrl) {
  return {
    fastgptConfig: {} as FastGPTConfigFileType
  };
}
```

此时 `licenseData` 为 `undefined`，**所有基于** **`licenseData`** **的检查都会被跳过**。

### 7.2 实际可用资源

| 资源    | 限制                 | 说明                  |
| ----- | ------------------ | ------------------- |
| 应用数量  | **无 License 级别限制** | 不受 `maxApps` 限制     |
| 知识库数量 | **无 License 级别限制** | 不受 `maxDatasets` 限制 |
| 用户数量  | **无 License 级别限制** | 不受 `maxUsers` 限制    |
| AI 积分 | **100 积分（免费套餐）**   | 用完后无法使用 AI 功能       |
| 团队成员  | **取决于套餐配置**        | 默认配置可能有限制           |

***

## 八、结论

### 8.1 当前环境限制总结

| 限制类型         | 状态      | 绕过难度        |
| ------------ | ------- | ----------- |
| License 用户数  | ❌ 未启用   | -           |
| License 应用数  | ❌ 未启用   | -           |
| License 知识库数 | ❌ 未启用   | -           |
| 积分限制         | ✅ 启用    | 中等（需修改套餐配置） |
| 套餐内应用数       | ⚠️ 可能启用 | 取决于配置       |
| 套餐内知识库数      | ⚠️ 可能启用 | 取决于配置       |

### 8.2 建议

1. **测试目的**: 仅供开发测试，可以设置 `HIDE_CHAT_COPYRIGHT_SETTING=true`

***

*报告生成时间: 2026-03-26*
*分析工具: 静态代码分析*
