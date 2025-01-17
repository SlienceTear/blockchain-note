# 怎样让智能合约更安全

注意：除了下面的列表之外，您还可以 [在 Guy Lando 的知识列表](https://github.com/guylando/KnowledgeLists/blob/master/EthereumSmartContracts.md) 和 [Consensys GitHub 存储库](https://consensys.github.io/smart-contract-best-practices/) 中找到更多安全建议和最佳实践。

## 陷阱

### 私人信息和随机性

您在智能合约中使用的所有内容都是公开可见的，甚至标记为 的局部变量和状态变量`private`。

如果您不想让矿工作弊，那么在智能合约中使用随机数是非常棘手的。

### 重入

合约 (A) 与另一合约 (B) 的任何交互以及以太坊的任何 `transfer` 都将控制权移交给该合约 (B)。这使得 B 可以在此交互完成之前回调 A。举个例子，以下代码包含一个错误（它只是一个片段，而不是完整的合约）：

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.6.0 <0.9.0;

// THIS CONTRACT CONTAINS A BUG - DO NOT USE
contract Fund {
    /// @dev Mapping of ether shares of the contract.
    mapping(address => uint) shares;
    /// Withdraw your share.
    function withdraw() public {
        if (payable(msg.sender).send(shares[msg.sender]))
            shares[msg.sender] = 0;
    }
}
```

因为 `send` 的 gas 限制，这个问题看起不来不太严重，但它仍然暴露了一个缺陷：Ether transfer 能够包含一个代码执行，因此接收者可能是一个会回调 `withdraw` 的合约。这回让他得到多次退款。

特别是下面的合约将允许一个攻击者多次退款，因为它使用 `call` ，`call` 将默认转发剩余的所有 gas：

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.6.2 <0.9.0;

// THIS CONTRACT CONTAINS A BUG - DO NOT USE
contract Fund {
    /// @dev Mapping of ether shares of the contract.
    mapping(address => uint) shares;
    /// Withdraw your share.
    function withdraw() public {
        (bool success,) = msg.sender.call{value: shares[msg.sender]}("");
        if (success)
            shares[msg.sender] = 0;
    }
}
```

为了避免重入攻击，你可以使用 检查-效果-交互（Checks-Effects-Interactions） 模式：

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.6.0 <0.9.0;

contract Fund {
    /// @dev Mapping of ether shares of the contract.
    mapping(address => uint) shares;
    /// Withdraw your share.
    function withdraw() public {
        uint share = shares[msg.sender];
        shares[msg.sender] = 0;
        payable(msg.sender).transfer(share);
    }
}
```

请注意，重入攻击不仅会受 Ether transfer 影响，实际任何合约调用都会影响。此外，还需要考虑多合约调用的情况。一个被调用的合约可以修改你所依赖的另一个合约的状态的情况。

### gas 限制与循环

没有固定迭代次数的循环，例如依赖于存储值的循环，必须谨慎使用：由于区块 gas 限制，交易只能消耗一定量的gas。无论是明确地还是仅仅由于正常操作，循环中的迭代次数都可能超过区块 gas 限制，这可能导致整个合约在某个点停滞不前。这可能不适用于 `view` 仅执行从区块链读取数据的功能。尽管如此，作为链上操作的一部分，其他合约可能会调用这些函数并让这些函数停止。请在合同文件中明确说明此类情况。

### 调用栈深度

外部函数调用可能随时失败，因为它们超过了最大调用堆栈大小限制 1024。在这种情况下，Solidity 会抛出异常。恶意参与者可能会在与您的合约交互之前将调用堆栈强制设置为高值。请注意，由于 [Tangerine Whistle](https://eips.ethereum.org/EIPS/eip-608) 硬分叉，[63/64 规则 ](https://eips.ethereum.org/EIPS/eip-150)使得调用堆栈深度攻击变得不切实际。另请注意，调用栈和表达式栈是不相关的，即使它们的大小限制为 1024 个堆栈槽。

请注意，如果调用栈耗尽，`.send()` 不会抛出异常，而是返回 false。低级函数 `.call()` 、 `.delegatecall()` 和 `staticcall()` 的行为与 `.send()` 相同。

### 代理

如果你的合约可以充当代理，即它可以使用用户提供的数据调用任意合约，那么用户就可以伪装代理合约的身份。即便你有保护措施，但最好也使代理没有任何权限（甚至对自己也没有）。如果需要，您可以使用第二个代理来完成此操作：

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity ^0.8.0;
contract ProxyWithMoreFunctionality {
    PermissionlessProxy proxy;

    function callOther(address _addr, bytes memory _payload) public
            returns (bool, bytes memory) {
        return proxy.callOther(_addr, _payload);
    }
    // Other functions and other functionality
}

// This is the full contract, it has no other functionality and
// requires no privileges to work.
contract PermissionlessProxy {
    function callOther(address _addr, bytes memory _payload) public
            returns (bool, bytes memory) {
        return _addr.call(_payload);
    }
}
```

### tx.origin

切勿使用 tx.origin 进行授权。假设你有一个这样的钱包合约：

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.7.0 <0.9.0;
// THIS CONTRACT CONTAINS A BUG - DO NOT USE
contract TxUserWallet {
    address owner;

    constructor() {
        owner = msg.sender;
    }

    function transferTo(address payable dest, uint amount) public {
        // THE BUG IS RIGHT HERE, you must use msg.sender instead of tx.origin
        require(tx.origin == owner);
        dest.transfer(amount);
    }
}
```

现在有人欺骗你将以太币发送到这个攻击钱包的地址：

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.7.0 <0.9.0;
interface TxUserWallet {
    function transferTo(address payable dest, uint amount) external;
}

contract TxAttackWallet {
    address payable owner;

    constructor() {
        owner = payable(msg.sender);
    }

    receive() external payable {
        TxUserWallet(msg.sender).transferTo(owner, msg.sender.balance);
    }
}
```

我们来看下调用链：

User - > TxAttackWallet(msg.sender = user, tx.origin = user) -> TxUserWallet(msg.sender = TxAttackWallet, tx.origin = user)

最后 `require(tx.origin == owner)` 判断通过，因此攻击钱包会立即耗尽您的所有资金。

### 二进制补码/下溢/上溢

与许多编程语言一样，Solidity 的整数类型实际上并不是整数。当值较小时，它们类似于整数，但不能表示任意大的数字。

以下代码导致溢出，因为加法的结果太大而无法存储在 type 中`uint8`：

```solidity
uint8 x = 255;
uint8 y = 1;
return x + y;
```

Solidity 有两种处理这些溢出的模式：

- 检查模式
- 不检查或“包裹”模式

默认检查模式将检测溢出并导致断言失败。您可以使用 `unchecked { ... }` 禁用此检查，导致溢出被静默忽略。如果上面的代码包裹在 `unchecked { ... }` 中，将返回 0 。

即使在检查模式下，也不要假设您受到溢出错误的保护。在这种模式下，溢出总是会恢复。如果无法避免溢出，这可能会导致智能合约卡在某个状态。

一般来说，阅读二进制补码表示的限制，它甚至有一些更特殊的有符号数的边缘情况。

尝试使用 `require` 将输入的大小限制在合理的范围内，并使用 [SMT 检查器](https://docs.soliditylang.org/en/v0.8.11/layout-of-source-files.html#smt-checker)来查找潜在的溢出。

### 删除 Mapping

Solidity 类型`mapping`（请参阅[Mapping Types](https://docs.soliditylang.org/en/v0.8.11/types.html#mapping-types)）是一种仅存储键值数据结构，它不跟踪分配了非零值的键。正因为如此，在没有关于写入键的额外信息的情况下清理映射是不可能的。

如果 `mapping` 用作动态存储数组的基本类型，则 `delete` 或 `pop` 数组对 `mapping` 元素没有影响。例如，如果 `mapping` 作为 `struct` 的成员字段，且该 `struct` 作为动态存储数组的基本类型，在对该结构体或数组进行分配时，同样对 `mapping` 没有影响。

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.6.0 <0.9.0;

contract Map {
    mapping (uint => uint)[] array;

    function allocate(uint _newMaps) public {
        for (uint i = 0; i < _newMaps; i++)
            array.push();
    }

    function writeMap(uint _map, uint _key, uint _value) public {
        array[_map][_key] = _value;
    }

    function readMap(uint _map, uint _key) public view returns (uint) {
        return array[_map][_key];
    }

    function eraseMaps() public {
        delete array;
    }
}
```

考虑下上面的示例以及这样的调用序列： `allocate(10)`, `writeMap(4, 128, 256)` 。此时，调用 `readMap(4, 128)` 将返回 256. 如果我们调用 `eraseMaps`，状态变量 `array` 的长度会被置为 0，但因为它的元素 `mapping` 不能被置为 0，因此 `mapping` 中的数据在合约存储中仍然有效。在删除 `array` 之后，调用 `allocate(5)` 允许我们再次访问 `array[4]` ，并且甚至在不调用 `writeMap` 的情况下，调用 `readMap(4, 128)` 将返回 256。

如果你的 `mapping` 信息必须删除，请考虑使用 [iterable mapping](https://github.com/ethereum/dapp-bin/blob/master/library/iterable_mapping.sol) 类似的库，允许你遍历 key 与 value，并删除 value。

### 脏高位bit

不占用全部的 32 字节的类型可能包含“脏高位bit”。如果你访问 `msg.data` 的话，这很重要。它会导致一个展延性风险：你可以精心制作一个交易，它用原始字节参数 `0xff000001` 和 `0x00000001` 调用 `f(uint8 x)` 。这两个参数用于此函数时，对于 `uint8 x` 而言，看起来都是 1。但 `msg.data` 是不同的，如果你使用 `keccak256(msg.data)` ，你将得到不同的结果。

### 正确使用 assert()，require()，revert()

函数 `assert` 和 `require` 可用于检查条件，如果条件不满足则抛出异常。

`assert` 函数只能用于测试内部错误和检查不变量。 

应该使用 `require` 函数来确保满足有效条件，例如输入或合约状态变量，或者验证来自外部合约调用的返回值。 

遵循这种范式允许形式分析工具验证永远无法达到无效操作码：这意味着代码中没有不变量被违反并且代码被形式验证。

```solidity
pragma solidity ^0.5.0;

contract Sharer {
    function sendHalf(address payable addr) public payable returns (uint balance) {
        require(msg.value % 2 == 0, "Even value required."); //Require() can have an optional message string
        uint balanceBeforeTransfer = address(this).balance;
        (bool success, ) = addr.call.value(msg.value / 2)("");
        require(success);
        // Since we reverted if the transfer failed, there should be
        // no way for us to still have half of the money.
        assert(address(this).balance == balanceBeforeTransfer - msg.value / 2); // used for internal error checking
        return address(this).balance;
    }
}
```

### 仅将函数修饰器用于检查

修饰符内的代码通常在函数体之前执行，因此任何状态更改或外部调用都会违反 [Checks-Effects-Interactions](https://solidity.readthedocs.io/en/develop/security-considerations.html#use-the-checks-effects-interactions-pattern) 模式。此外，开发人员也可能不会注意到这些语句，因为修饰符的代码可能远离函数声明。例如，修饰符的外部调用可能导致重入攻击：

```solidity
contract Registry {
    address owner;

    function isVoter(address _addr) external returns(bool) {
        // Code
    }
}

contract Election {
    Registry registry;

    modifier isEligible(address _addr) {
        require(registry.isVoter(_addr));
        _;
    }

    function vote() isEligible(msg.sender) public {
        // Code
    }
}
```

在这种情况下， `Registry` 合约可以通过调用 `Election.vote()` inside 进行重入攻击`isVoter()`。

**注意：**使用 [函数修饰器](https://solidity.readthedocs.io/en/develop/contracts.html#function-modifiers) 替换多个函数中的重复条件检查，例如 `isOwner()`，否则在函数内部使用 `require` 或 `revert` 。这使您的智能合约代码更具可读性和更易于审计。

### 注意整数除法的舍入

所有整数除法都向下舍入到最接近的整数。如果您需要更高的精度，请考虑使用乘数，或同时存储分子和分母。

```solidity
// bad
uint x = 5 / 2; // Result is 2, all integer divison rounds DOWN to the nearest integer
```

使用乘数可以防止四舍五入，在将来使用 x 时需要考虑这个乘数：

```solidity
// good
uint multiplier = 10;
uint x = (5 * multiplier) / 2;
```

存储分子和分母意味着你可以计算 `numerator/denominator` 链下的结果：

```solidity
// good
uint numerator = 5;
uint denominator = 2;
```

### 注意抽象合约和接口之间的权衡

接口和抽象合约都为智能合约提供了一种可定制和可重用的方法。Solidity 0.4.11 中引入的接口类似于抽象合约，但不能实现任何功能。接口也有限制，例如不能访问存储或从其他接口继承，这通常使抽象合约更实用。虽然，接口对于在实现之前设计合同肯定有用。此外，重要的是要记住，如果合约继承自抽象合约，它必须通过覆盖实现所有未实现的功能，否则它也将是抽象的。

### Fallback 函数

#### 保持Fallback函数简单

当发送一个没有参数的消息到合约（或者没有函数匹配）时，[Fallback 函数](https://solidity.readthedocs.io/en/latest/contracts.html#fallback-function) 被调用，并且当从 `.send()` 或 `.transfer()` 调用时只能使用 2300 gas 。如果您希望能够从 `.send()` 或 `.transfer()` 接收 Ether ，那么您在Fallback 函数中最多可以做的就是记录一个事件。如果需要更多 gas，请使用适当的函数。

```solidity
// bad
function() payable { balances[msg.sender] += msg.value; }

// good
function deposit() payable external { balances[msg.sender] += msg.value; }

function() payable { require(msg.data.length == 0); emit LogDepositReceived(msg.sender); }
```

#### 在 Fallback 函数中检查 data 长度

由于 [Fallback 函数](https://solidity.readthedocs.io/en/latest/contracts.html#fallback-function) 不仅在普通 Ether transfer（没有数据）时调用，而且在没有其他函数匹配时调用，如果Fallback 函数仅用于记录接收到的 Ether，则应检查数据是否为空。否则，如果你的合约使用不正确，调用了不存在的函数，调用者将不会注意到。

```solidity
// bad
function() payable { emit LogDepositReceived(msg.sender); }

// good
function() payable { require(msg.data.length == 0); emit LogDepositReceived(msg.sender); }
```

### 显式标记 Payable 函数和状态变量

从 Solidity 开始 `0.4.0`，每个接收 Ether 的函数都必须使用 `payable` 修饰符，否则如果交易 `msg.value > 0` 将revert（[强制时除外](https://consensys.github.io/smart-contract-best-practices/recommendations/#remember-that-ether-can-be-forcibly-sent-to-an-account)）。

**注意：**可能不明显的事情： `payable` 修饰符仅适用于来自 *外部* 合约的调用。如果我在同一个合约的 Payable 函数中调用了一个非 Payable 函数，这个非 Payable 函数不会失败，尽管 `msg.value` 它仍然被设置。

### 显式标记函数和状态变量的可见性

明确标记函数和状态变量的可见性。函数可以指定为 `external`，`public`，`internal` 或 `private` 。请理解它们之间的差异。明确标记可见性将更容易捕捉关于谁可以调用函数或访问变量的错误假设。

- `External` 函数是合约接口的一部分。不能在内部调用 `External` 函数 f（例如，f() 不行，但 this.f() 可以）。`External` 函数在接收大量数据时可能更高效。
- `Public` 函数是合约接口的一部分，既可以在内部调用，也可以通过消息调用。对于公共状态变量，会生成一个自动 getter 函数。
- `Internal` 函数和状态变量只能在内部访问，不能使用 `this`。
- `Private` 函数和状态变量仅对定义它们的合约可见，而在派生合约中不可见。 **注意**：合约内的所有内容对区块链外部的所有观察者都是可见的，甚至是 `Private` 变量。

```solidity
// bad
uint x; // the default is internal for state variables, but it should be made explicit
function buy() { // the default is public
    // public code
}

// good
uint private y;
function buy() external {
    // only callable externally or using this.buy()
}

function utility() public {
    // callable externally, as well as internally: changing this code requires thinking about both cases.
}

function internalAction() internal {
    // internal code
}
```

### 将编译指示锁定到特定的编译器版本

合约应该使用与它们经过最多测试的相同编译器版本和标志来部署。锁定 pragma 有助于确保合约不会被意外部署，例如使用可能具有更高风险未发现错误的最新编译器。合同也可能由其他人部署，并且 pragma 指示原作者预期的编译器版本。

```solidity
// bad
pragma solidity ^0.4.4;


// good
pragma solidity 0.4.4;
```

注意：浮动 pragma 版本（即 `^0.4.25`）用 `0.4.26-nightly.2018.9.25` 来编译是 OK 的，但不应使用 nightly 来编译生产代码。

**警告：**当合约打算供其他开发人员使用时，可以允许 Pragma 语句浮动，例如库或 EthPM 包中的合约。否则，开发人员需要手动更新编译指示才能在本地编译。

### 使用事件来监控合约活动

有一种方法可以在部署后监控合约的活动是很有用的。实现这一点的一种方法是查看合约的所有交易，但这可能还不够，因为合约之间的消息调用不会记录在区块链中。此外，它只显示输入参数，而不是对状态进行的实际更改。事件也可用于触发用户界面中的功能。

```solidity
contract Charity {
    mapping(address => uint) balances;

    function donate() payable public {
        balances[msg.sender] += msg.value;
    }
}

contract Game {
    function buyCoins() payable public {
        // 5% goes to charity
        charity.donate.value(msg.value / 20)();
    }
}
```

在这里， `Game` 合约将内部调用 `Charity.donate()`。该交易不会出现在 `Charity` 的外部交易列表中，而只在内部交易中可见。

事件是记录合约中发生的事情的便捷方式。发出的事件与其他合同数据一起留在区块链中，可供将来审计。这是对上述示例的改进，使用事件来提供慈善机构的捐赠历史。

```solidity
contract Charity {
    // define event
    event LogDonate(uint _amount);

    mapping(address => uint) balances;

    function donate() payable public {
        balances[msg.sender] += msg.value;
        // emit event
        emit LogDonate(msg.value);
    }
}

contract Game {
    function buyCoins() payable public {
        // 5% goes to charity
        charity.donate.value(msg.value / 20)();
    }
}
```

在这里，无论是否直接通过合同的所有交易都 `Charity` 将与捐赠的金额一起显示在该合同的事件列表中。

### 请注意，“built-in”函数可能会被隐藏

目前可以在 Solidity 中 [隐藏](https://en.wikipedia.org/wiki/Variable_shadowing) 内置的全局变量。这允许合约覆盖built-in函数的功能，例如 `msg` 和 `revert()`。尽管这是 [有意为之](https://github.com/ethereum/solidity/issues/1249)，但它可能会误导合约用户对合约的真实行为。

```solidity
contract PretendingToRevert {
    function revert() internal constant {}
}

contract ExampleContract is PretendingToRevert {
    function somethingBad() public {
        revert();
    }
}
```

合约用户（和审计员）应该了解他们打算使用的任何应用程序的完整智能合约源代码。

### 时间戳依赖

使用时间戳执行合约中的关键功能时，有三个主要考虑因素，尤其是当操作涉及资金转移时。

#### 时间戳操作

请注意，区块的时间戳可以由矿工操纵。考虑这个[合约](https://etherscan.io/address/0xcac337492149bdb66b088bf5914bedfbf78ccc18#code)：

```solidity
uint256 constant private salt =  block.timestamp;

function random(uint Max) constant private returns (uint256 result){
    //get the best seed for randomness
    uint256 x = salt * 100/Max;
    uint256 y = salt * block.number/(salt % 5) ;
    uint256 seed = block.number/3 + (salt % 300) + Last_Payout + y;
    uint256 h = uint256(block.blockhash(seed));

    return uint256((h / x)) % Max + 1; //random number between 1 and Max
}
```

当合约使用时间戳播种一个随机数时，矿工实际上可以在区块被验证后的 15 秒内发布一个时间戳，从而有效地允许矿工预先计算一个更有利于他们中奖机会的选项。时间戳不是随机的，不应在该上下文中使用。

#### 15秒规则

[Ethereum 黄皮书](https://ethereum.github.io/yellowpaper/paper.pdf) 没有规定多少块可以随时间漂移的限制，但 [它确实规定](https://ethereum.stackexchange.com/a/5926/46821) 了每个时间戳应该大于其父时间戳。流行的以太坊协议实现 [Geth](https://github.com/ethereum/go-ethereum/blob/4e474c74dc2ac1d26b339c32064d0bac98775e77/consensus/ethash/consensus.go#L45) 和 [Parity](https://github.com/paritytech/parity-ethereum/blob/73db5dda8c0109bb6bc1392624875078f973be14/ethcore/src/verification/verification.rs#L296-L307) 都拒绝未来时间戳超过 15 秒的块。因此，评估时间戳使用的一个好的经验法则是：如果你的时间相关事件的规模可以变化15秒并保持完整性，那么使用 `block.timestamp`。

#### 避免 block.number 用作时间戳

可以使用 `block.number` 属性和 [平均出块时间](https://etherscan.io/chart/blocktime)来估计时间增量，但这不是未来的证据，因为块时间可能会改变（例如 [分叉重组](https://blog.ethereum.org/2015/08/08/chain-reorganisation-depth-expectations/) 和 [难度炸弹](https://github.com/ethereum/EIPs/issues/649)）。在跨越几天的销售中，15 秒规则允许人们获得更可靠的时间估计。

见 [SWC-116](https://swcregistry.io/docs/SWC-116)

### 多重继承

在 Solidity 中使用多重继承时，了解编译器如何构成继承图非常重要。

```solidity
contract Final {
    uint public a;
    function Final(uint f) public {
        a = f;
    }
}

contract B is Final {
    int public fee;

    function B(uint f) Final(f) public {
    }
    function setFee() public {
        fee = 3;
    }
}

contract C is Final {
    int public fee;

    function C(uint f) Final(f) public {
    }
    function setFee() public {
        fee = 5;
    }
}

contract A is B, C {
  function A() public B(3) C(5) {
      setFee();
  }
}
```

部署合约时，编译器将 从右到左 *线性化继承*（在关键字 *is* 之后 ，父项从最基类到最派生列出）。这是合约 A 的线性化：

**最终 <- B <- C <- A**

线性化的结果将产生 `fee` 5 的值，因为 C 是最衍生的合约。这似乎很明显，但想象一下 C 能够隐藏关键函数、重新排序布尔子句并导致开发人员编写可利用的合约的场景。静态分析目前不会引发被遮盖的函数的问题，因此必须手动检查。

为了帮助做出贡献，Solidity 的 Github 有一个 包含所有继承相关问题的[项目](https://github.com/ethereum/solidity/projects/9#card-8027020)。

见 [SWC-125](https://swcregistry.io/docs/SWC-125)

### 使用接口类型而不是地址来保证类型安全

当函数将合约地址作为参数时，最好传递接口或合约类型而不是 原始的 `address`。如果函数在源代码的其他地方被调用，编译器将提供额外的类型安全保证。

在这里，我们看到了两种选择：

```
contract Validator {
    function validate(uint) external returns(bool);
}

contract TypeSafeAuction {
    // good
    function validateBet(Validator _validator, uint _value) internal returns(bool) {
        bool valid = _validator.validate(_value);
        return valid;
    }
}

contract TypeUnsafeAuction {
    // bad
    function validateBet(address _addr, uint _value) internal returns(bool) {
        Validator validator = Validator(_addr);
        bool valid = validator.validate(_value);
        return valid;
    }
}
```

然后可以从以下示例中看出使用上述 `TypeSafeAuction` 合约的好处 。如果 `validateBet()` 使用 `address` 参数而不是 `Validator` 合约类型，编译器将抛出此错误：

```solidity
contract NonValidator{}

contract Auction is TypeSafeAuction {
    NonValidator nonValidator;

    function bet(uint _value) {
        bool valid = validateBet(nonValidator, _value); // TypeError: Invalid type for argument in function call.
                                                        // Invalid implicit conversion from contract NonValidator
                                                        // to contract Validator requested.
    }
}
```

### 避免 extcodesize 用于检查外部拥有帐户(EOA)

以下修饰符（或类似的检查）通常用于验证调用是来自外部拥有的账户（EOA）还是合约账户：

```solidity
// bad
modifier isNotContract(address _a) {
  uint size;
  assembly {
    size := extcodesize(_a)
  }
    require(size == 0);
     _;
}
```

这个想法很简单：如果一个地址包含代码，它就不是一个 EOA，而是一个合约账户。但是， **合约在构建期间没有可用的源代码**。这意味着在构造函数运行时，它可以调用其他合约，但 `extcodesize` 它的地址返回零。下面是一个最小的例子，展示了如何绕过这个检查：

```solidity
contract OnlyForEOA {    
    uint public flag;

    // bad
    modifier isNotContract(address _a){
        uint len;
        assembly { len := extcodesize(_a) }
        require(len == 0);
        _;
    }

    function setFlag(uint i) public isNotContract(msg.sender){
        flag = i;
    }
}

contract FakeEOA {
    constructor(address _a) public {
        OnlyForEOA c = OnlyForEOA(_a);
        c.setFlag(1);
    }
}
```



**警告：**这个问题很微妙。如果您的目标是阻止其他合约调用您的合约，那么 `extcodesize` 检查可能就足够了。另一种方法是检查 的值 `(tx.origin == msg.sender)`，尽管这也 [有缺点](https://consensys.github.io/smart-contract-best-practices/recommendations/#avoid-using-txorigin)。

## 建议

### 认真对待警告

如果编译器警告你某事，你应该改变它。即使您不认为此特定警告具有安全隐患，也可能隐藏在其下的另一个问题。我们发出的任何编译器警告都可以通过对代码的轻微更改来消除。

始终使用最新版本的编译器来通知所有最近引入的警告。

编译器发出的消息类型`info`并不危险，只是表示编译器认为可能对用户有用的额外建议和可选信息。

### 限制以太币数量

限制可以存储在智能合约中的以太币（或其他代币）的数量。如果您的源代码、编译器或平台出现错误，这些资金可能会丢失。如果你想限制你的损失，限制以太币的数量。

### 保持小型和模块化

使您的合同小而易懂。在其他合约或库中挑选出不相关的功能。关于源代码质量的一般建议当然适用：限制局部变量的数量、函数的长度等。给函数加以说明，以便其他人可以看到您的意图以及它是否与代码的作用不同。

### 使用检查-效果-交互模式

大多数函数将首先执行一些检查（谁调用了函数，参数是否在范围内，它们是否发送了足够的以太币，这个人是否有令牌等）。这些检查应首先进行。

作为第二步，如果所有检查都通过，则应该对当前合约的状态变量产生影响。与其他合约的交互应该是任何功能的最后一步。

早期的合约延迟了一些效果，并等待外部函数调用以非错误状态返回。由于上面解释的重入问题，这通常是一个严重的错误。

另外请注意，对已知合约的调用可能反过来会导致对未知合约的调用，因此最好始终应用此模式。

### 包括故障安全模式

虽然使您的系统完全去中心化将消除任何中介，但包含某种故障安全机制可能是一个好主意，尤其是对于新代码：

你可以在你的智能合约中添加一个函数来执行一些自我检查，比如“是否有任何以太币泄露？”、“代币的总和是否等于合约的余额？” 或类似的东西。请记住，您不能为此使用过多的气体，因此可能需要通过链下计算提供帮助。

如果自检失败，合约会自动切换到某种“故障安全”模式，例如禁用大部分功能，将控制权交给固定且受信任的第三方，或者只是将合约转换为简单的“把我的钱还给我”合同。

### 要求同行评审

检查一段代码的人越多，发现的问题就越多。要求人们审查您的代码也有助于交叉检查以了解您的代码是否易于理解——这是良好智能合约的一个非常重要的标准。

## 工具

### 形式化验证



## 参考

[1] [Solidity: Security Considerations](https://docs.soliditylang.org/en/v0.8.11/security-considerations.html)

[2] [Solidity Best Practices for Smart Contract Security](https://consensys.net/blog/developers/solidity-best-practices-for-smart-contract-security/)