# Light Account (Fork Version)

## 项目概述

[Alchemy Light Account](https://github.com/alchemyplatform/light-account) 的 fork 版本，是一套轻量级的 ERC-4337 兼容智能合约账户，具有指定所有权功能。

### 核心功能

- **账户抽象**：完全兼容 ERC-4337 标准的智能合约账户
- **所有权管理**：支持单一或多所有者账户配置
- **签名验证**：支持 EOA 和 ERC-1271 合约签名验证
- **可升级性**：使用 ERC-1967 代理模式实现账户升级
- **Gas 优化**：通过命名空间存储和 Solady 库优化 gas 消耗

## 新增改动

### ERC20 代付功能

本 fork 版本新增了 **ERC20 Payment Token** 配置功能，解决了智能账户部署时在不使用 guarantor 预付的情况下无法使用 ERC20 代付的问题。

#### 实现机制

1. **Factory 配置**：在 `LightAccountFactory` 和 `BaseLightAccountFactory` 中添加了 `erc20PaymentToken` 映射，支持配置多个 ERC20 代付代币。

2. **账户初始化时授权**：在账户初始化 (`initialize`) 时，如果配置了 ERC20 代付代币，账户会自动向指定的 `paymasterAddress` 授权最大额度，使得 paymaster 能够代扣 ERC20 代币用于支付 gas 费用。

3. **Salt 编码方式**：通过 `salt` 参数的最低 16 位来选择使用哪个 ERC20 代付代币索引，实现灵活的代币选择。

#### Salt 逻辑说明

`createAccount` 函数的 `salt` 参数具有双重用途：

```solidity
function createAccount(address owner, uint256 salt) external returns (LightAccount account)
```

- **高 240 位**：用于生成不同的账户地址（与 owner 一起计算）
- **低 16 位**：用于选择 ERC20 代付代币索引

示例：

```solidity
// index 存储在 salt 的最低 16 位
uint index = uint16(salt);  // 提取最低 16 位作为索引
account.initialize(owner, erc20PaymentToken[index], paymasterAddress);
```

这样设计的好处：

- 同一个 owner 可以创建多个使用不同 ERC20 代币的账户
- 通过调整 salt 值，可以在部署时自由选择使用哪种 ERC20 代币进行 gas 代付
- 账户地址由 `keccak256(owner, salt)` 确定，确保地址的唯一性和可预测性

#### 部署时配置授权交易

**步骤 1：Factory 配置 ERC20 代币**

Factory owner 需要预先配置 ERC20 代付代币：

```solidity
// 配置索引 0 的 ERC20 代币（USDC 示例）
factory.setERC20PaymentToken(0, usdcAddress);

// 配置索引 1 的 ERC20 代币（DAI 示例）
factory.setERC20PaymentToken(1, daiAddress);

// 配置 paymaster 地址
factory.setPaymasterAddress(paymasterAddress);
```

**步骤 2：计算账户地址**

在部署前，可以通过 `getAddress` 预计算账户地址：

```solidity
// 使用索引 0 的 ERC20 代币（USDC）
uint256 salt = 0x123400000000000000000000000000000000000000000000000000000000;  // 低 16 位为 0
address accountAddress = factory.getAddress(ownerAddress, salt);

// 使用索引 1 的 ERC20 代币（DAI）
uint256 salt2 = 0x123400000000000000000000000000000000000000000000000000000001; // 低 16 位为 1
address accountAddress2 = factory.getAddress(ownerAddress, salt2);
```

**步骤 3：构造 UserOperation**

在 `initCode` 中包含 factory 地址和 `createAccount` 调用：

```solidity
bytes memory initCode = abi.encodePacked(
    address(factory),
    abi.encodeWithSelector(
        factory.createAccount.selector,
        ownerAddress,
        salt  // 通过 salt 的低 16 位指定使用哪个 ERC20 代币
    )
);

// 构造 UserOperation，使用 ERC20 paymaster
PackedUserOperation memory userOp = PackedUserOperation({
    sender: accountAddress,
    nonce: 0,
    initCode: initCode,
    callData: executeCallData,
    // ... 其他字段
    paymasterAndData: abi.encodePacked(
        paymasterAddress,
        // paymaster 特定数据
    )
});
```

**步骤 4：自动授权流程**

当 `createAccount` 被调用时：

1. 部署账户合约（如果尚未部署）
2. 调用 `initialize(owner, erc20PaymentToken[index], paymasterAddress)`
3. 账户自动执行 `IERC20(erc20PaymentToken).approve(paymasterAddress, type(uint256).max)`
4. Paymaster 现在可以从账户中扣除 ERC20 代币用于支付 gas

**关键优势**：

- 部署和授权在同一交易中完成，无需额外交易
- 无需 guarantor 预付 gas
- 用户可以直接使用 ERC20 代币支付账户部署和初始化的 gas 费用
  |
