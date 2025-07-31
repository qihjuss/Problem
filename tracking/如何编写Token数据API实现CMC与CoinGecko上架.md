```markdown
# 如何编写Token数据API实现CMC与CoinGecko上架

## 项目启动前的准备事项

在正式编写API前，需要完成基础准备工作。这包括：

1. 获取区块链节点服务（推荐使用[Infura](https://infura.io/)）
2. 搭建数据库环境（MongoDB Atlas提供免费方案）
3. 准备智能合约ABI文件
4. 整理项目地址清单

👉 [获取区块链节点服务](https://bit.ly/okx_welcome)

### 关键数据指标解析

在构建API前必须明确三个核心指标：

| 指标名称 | 定义 | 计算公式 |
|---------|------|---------|
| 总供应量(Total Supply) | 已铸造且未销毁的代币总量 | 总铸造量 - 已销毁量 |
| 最大供应量(Max Supply) | 合约预设的代币上限 | 固定值（部分代币无上限） |
| 流通供应量(Circulating Supply) | 实际参与市场流通的代币量 | 总供应量 - 锁仓量 - 销毁量 |

**锁仓代币识别标准**：
- 团队预留代币（开发/运营/激励等用途）
- 投资者代币（按计划解锁）
- 激励代币（按规则释放）

## API开发全流程指南

### 技术架构设计

完整的API系统包含以下核心模块：

```
项目结构：
├── abi/            # 存放智能合约ABI
├── addresses/      # 地址配置文件
├── middleware/     # 请求中间件
├── routes/         # API路由定义
├── utils/          # 核心功能模块
│   ├── db.js       # 数据库连接
│   ├── getChainData.js  # 区块链数据获取
│   └── getProjectData.js # 代币数据计算
└── server.js       # 服务启动入口
```

### 核心功能实现

#### 区块链数据获取模块

```javascript
// utils/getChainData.js
const schedule = require("node-schedule");
const Web3 = require("web3");

const setupWeb3 = async () => {
  // 多节点配置实现容错
  const endpoints = [
    "https://bsc-dataseed.binance.org/",
    "https://bsc-dataseed1.defibit.io/"
  ];
  
  // 创建Web3实例
  let web3Instance;
  while(!web3Instance) {
    for(const endpoint of endpoints) {
      try {
        web3Instance = new Web3(new Web3.providers.HttpProvider(endpoint));
        break;
      } catch(err) {
        continue;
      }
    }
    await new Promise(resolve => setTimeout(resolve, 5000));
  }
  return web3Instance;
}

// 定时任务配置（每15分钟更新）
schedule.scheduleJob("0,15,30,45 * * * *", async () => {
  const web3 = await setupWeb3();
  updateTokenData(web3);
});
```

#### 代币数据计算逻辑

```javascript
// utils/getProjectData.js
const calculateSupply = async (contractAddress, burntAddress) => {
  const contract = new web3.eth.Contract(abi, contractAddress);
  
  // 基础数据获取
  const totalSupply = await contract.methods.totalSupply().call();
  const burntBalance = await contract.methods.balanceOf(burntAddress).call();
  
  // 锁仓地址遍历
  let lockedBalance = 0;
  for(const address of lockupAddresses) {
    lockedBalance += await contract.methods.balanceOf(address).call();
  }
  
  // 最终供应量计算
  return {
    totalSupply: totalSupply - burntBalance,
    circulatingSupply: totalSupply - burntBalance - lockedBalance
  };
}
```

## 常见问题解答

**Q：API响应时间超过多少会影响审核？**  
A：建议控制在3秒以内，CoinGecko要求API响应时间低于5秒

**Q：是否需要同时支持HTTP和HTTPS？**  
A：必须强制使用HTTPS，CoinMarketCap要求TLS 1.2及以上加密协议

**Q：如何处理多链部署的供应量统计？**  
A：需要分别获取各链数据后做聚合计算，参考：  
```javascript
const totalSupply = ethSupply + bscSupply + polygonSupply
```

**Q：API需要支持哪些HTTP方法？**  
A：建议仅开放GET方法，设置CORS白名单提升安全性

👉 [提升API安全性方案](https://bit.ly/okx_welcome)

## 部署与测试要点

### 测试用例设计

| 测试场景 | 预期结果 | 测试工具 |
|---------|----------|---------|
| 无效地址请求 | 404错误 | Postman |
| 高频请求测试 | 429限流 | Artillery |
| 节点故障模拟 | 自动切换节点 | Chaos Monkey |

### 性能优化建议

1. 使用Redis做热点数据缓存（比MongoDB快5-10倍）
2. 启用Gzip压缩（减少70%传输体积）
3. 设置CDN加速（推荐Cloudflare免费方案）
4. 采用异步计算+预生成模式

👉 [获取CDN加速服务](https://bit.ly/okx_welcome)

## 审核材料准备技巧

### 供应量计算证明材料

需要准备的证明文件包括：
- 区块链浏览器截图（显示销毁地址余额）
- 锁仓地址所有权证明
- 智能合约代码审计报告
- 代币经济模型白皮书

### 常见审核驳回原因

| 问题类型 | 出现概率 | 解决方案 |
|---------|----------|----------|
| 供应量数据不一致 | 38% | 建立自动校验机制 |
| API响应超时 | 25% | 增加节点冗余 |
| 数据格式错误 | 18% | 实施严格schema验证 |
| 锁仓证明不足 | 12% | 提供多方审计报告 |
