## 🆘 Help Wanted: Encrypted Roulette Game with Zama FHE VM – `placeBet` Fails to Initialize Encrypted Input

### 🧩 Problem Description

I'm building a fully homomorphic encryption (FHE) powered roulette betting DApp using Zama’s FHE VM and SDK.  
The goal is to allow users to submit encrypted bets, which are privately processed by a Solidity smart contract using `@fhevm/solidity`.

However, the contract fails when decrypting external encrypted input using `FHE.fromExternal(...)`.

---

### 🧪 Project Setup

#### 🔗 Smart Contract (Solidity Snippet)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@fhevm/solidity/lib/FHE.sol";
import "encrypted-types/EncryptedTypes.sol";

contract FullyEncryptedRouletteGame is ReentrancyGuard, Ownable {
    IERC20 public zartcToken; // ZARTC 代币合约（可修改）
    uint256 public constant DECIMALS = 10; // ZARTC 精度
    uint32 public constant MAX_NUMBER = 36; // 轮盘最大数字
    uint32 public constant SINGLE_NUMBER_PAYOUT = 35; // 单号赔率 x35
    uint32 public constant GRID_PAYOUT = 11; // 网格赔率 x1.1 (乘以10避免小数)
    uint32 public constant GRID_PAYOUT_DENOMINATOR = 10; // 网格赔率分母

    // 下注类型
    enum BetType { SingleNumber, Red, Black }

    // 下注结构体（使用加密类型）
    struct Bet {
        address player;
        euint64 amount; // 改为 euint64
        euint64 betType; // 改为 euint64
        euint64 number;  // 改为 euint64
    }

    // 轮盘结果
    struct Result {
        euint64 number; // 改为 euint64
        ebool isRed; // 加密的红/黑
        bool fulfilled; // 是否已处理
    }

    mapping(uint256 => Bet[]) public roundBets; // 每轮下注
    mapping(uint256 => Result) public roundResults; // 每轮结果
    mapping(address => euint64) public pendingRewards; // 改为 euint64
    uint256 public currentRound; // 当前轮次
    uint32[] public redNumbers = [1, 3, 5, 7, 9, 12, 14, 16, 18, 19, 21, 23, 25, 27, 30, 32, 34, 36]; // 红号

    // 事件
    event BetPlaced(address indexed player, uint256 round, bytes32 amountHandle, bytes32 betTypeHandle, bytes32 numberHandle);
    event ResultFulfilled(uint256 indexed round, bytes32 numberHandle, bytes32 isRedHandle);
    event RewardDistributed(address indexed player, bytes32 amountHandle);
    event RewardReadyForDecryption(address indexed player, bytes32 rewardHandle);
    event ZARTCTokenUpdated(address indexed oldToken, address indexed newToken);
    event ContractUpgraded(address indexed newContract, uint256 zartcAmount);

    constructor(address _zartcToken) Ownable(msg.sender) {
        require(_zartcToken != address(0), "Invalid ZARTC token address");
        zartcToken = IERC20(_zartcToken);
    }

    // 用户下注（加密输入）
    function placeBet(
        externalEuint64 encryptedBetType,   // 改为 euint64
        externalEuint64 encryptedNumber,    // 改为 euint64
        externalEuint64 encryptedAmount,    // 改为 euint64
        bytes calldata inputProof
    ) external nonReentrant {
        // 验证加密输入并转换为内部类型
        euint64 internalAmount = FHE.fromExternal(encryptedAmount, inputProof);
        require(FHE.isInitialized(internalAmount), "Amount not initialized");

        euint64 internalBetType = FHE.fromExternal(encryptedBetType, inputProof);
        require(FHE.isInitialized(internalBetType), "Bet type not initialized");

        euint64 internalNumber = FHE.fromExternal(encryptedNumber, inputProof);
        require(FHE.isInitialized(internalNumber), "Number not initialized");

        // 如果是单号投注，验证数字（在加密域）
        ebool isSingleNumber = FHE.eq(internalBetType, FHE.asEuint64(uint64(BetType.SingleNumber)));
        ebool isValidNumber = FHE.le(internalNumber, FHE.asEuint64(uint64(MAX_NUMBER)));
        ebool isValidBet = FHE.or(FHE.not(isSingleNumber), isValidNumber); // Valid if not SingleNumber or number <= MAX_NUMBER
        require(FHE.isInitialized(isValidBet), "Invalid bet validation");

        // 存储下注
        roundBets[currentRound].push(Bet({
            player: msg.sender,
            amount: internalAmount,
            betType: internalBetType,
            number: internalNumber
        }));

        // 授权当前合约使用加密值
        FHE.allowThis(internalAmount);
        FHE.allowThis(internalBetType);
        FHE.allowThis(internalNumber);

        if (roundBets[currentRound].length == 1) {
            generateResult();
        }

        emit BetPlaced(
            msg.sender,
            currentRound,
            FHE.toBytes32(internalAmount),
            FHE.toBytes32(internalBetType),
            FHE.toBytes32(internalNumber)
        );
    }

    // 生成加密随机结果
    function generateResult() private {
        euint64 winningNumber = FHE.randEuint64(uint64(MAX_NUMBER + 1)); // 改为 euint64
        ebool isRed = FHE.asEbool(false);

        for (uint256 i = 0; i < redNumbers.length; i++) {
            ebool isMatch = FHE.eq(winningNumber, FHE.asEuint64(uint64(redNumbers[i])));
            isRed = FHE.or(isRed, isMatch);
        }

        roundResults[currentRound] = Result({
            number: winningNumber,
            isRed: isRed,
            fulfilled: true
        });

        // 允许前端用户解密
        FHE.allow(winningNumber, tx.origin);
        FHE.allow(isRed, tx.origin);

        // 允许本合约内部访问结果
        FHE.allowThis(winningNumber);
        FHE.allowThis(isRed);

        processBets(currentRound, winningNumber, isRed);
        currentRound++;

        emit ResultFulfilled(currentRound - 1, FHE.toBytes32(winningNumber), FHE.toBytes32(isRed));
    }

    // 计算加密奖励
    function processBets(uint256 round, euint64 winningNumber, ebool isRed) private {
        Bet[] memory bets = roundBets[round];
        for (uint256 i = 0; i < bets.length; i++) {
            Bet memory bet = bets[i];
            euint64 payout = FHE.asEuint64(0);

            // 单号投注
            ebool isSingleNumberWin = FHE.and(
                FHE.eq(bet.betType, FHE.asEuint64(uint64(BetType.SingleNumber))),
                FHE.eq(bet.number, winningNumber)
            );
            payout = FHE.select(isSingleNumberWin, FHE.mul(bet.amount, FHE.asEuint64(uint64(SINGLE_NUMBER_PAYOUT))), payout);

            // 红号投注
            ebool isRedWin = FHE.and(
                FHE.eq(bet.betType, FHE.asEuint64(uint64(BetType.Red))),
                isRed
            );
            euint64 redPayout = FHE.mul(bet.amount, FHE.asEuint64(uint64(GRID_PAYOUT)));
            payout = FHE.select(isRedWin, redPayout, payout);

            // 黑号投注（非红且非0）
            ebool isBlackWin = FHE.and(
                FHE.eq(bet.betType, FHE.asEuint64(uint64(BetType.Black))),
                FHE.and(FHE.not(isRed), FHE.ne(winningNumber, FHE.asEuint64(0)))
            );
            payout = FHE.select(isBlackWin, redPayout, payout);

            // 分配奖励
            if (FHE.isInitialized(payout)) {
                pendingRewards[bet.player] = FHE.add(pendingRewards[bet.player], payout);
                FHE.allowThis(payout);
                emit RewardDistributed(bet.player, FHE.toBytes32(payout));
            }
        }
    }

    // 领取奖励（标记为可解密，需前端处理）
    function claimReward() external nonReentrant {
        euint64 reward = pendingRewards[msg.sender];
        require(FHE.isInitialized(reward), "No rewards to claim");

        // 仅授权调用用户本人解密（而非全网公开）
        bytes32 rewardHandle = FHE.toBytes32(reward);
        FHE.allow(reward, msg.sender); // 👈 给用户临时权限
        pendingRewards[msg.sender] = FHE.asEuint64(0);

        emit RewardReadyForDecryption(msg.sender, rewardHandle);
    }

    // 查询用户下注（仅返回玩家地址和加密句柄）
    function getUserBets(address user, uint256 round) external view returns (address[] memory players, bytes32[] memory amountHandles, bytes32[] memory betTypeHandles, bytes32[] memory numberHandles) {
        Bet[] memory bets = roundBets[round];
        uint256 count = 0;
        for (uint256 i = 0; i < bets.length; i++) {
            if (bets[i].player == user) count++;
        }

        players = new address[](count);
        amountHandles = new bytes32[](count);
        betTypeHandles = new bytes32[](count);
        numberHandles = new bytes32[](count);

        uint256 index = 0;
        for (uint256 i = 0; i < bets.length; i++) {
            if (bets[i].player == user) {
                players[index] = bets[i].player;
                amountHandles[index] = FHE.toBytes32(bets[i].amount);
                betTypeHandles[index] = FHE.toBytes32(bets[i].betType);
                numberHandles[index] = FHE.toBytes32(bets[i].number);
                index++;
            }
        }
        return (players, amountHandles, betTypeHandles, numberHandles);
    }

    // 修改 ZARTC 代币地址（仅 owner）
    function updateZARTCToken(address newToken) external onlyOwner {
        require(newToken != address(0), "Invalid token address");
        address oldToken = address(zartcToken);
        zartcToken = IERC20(newToken);
        emit ZARTCTokenUpdated(oldToken, newToken);
    }

    // 升级合约（仅 owner）
    function upgradeContract(address newContract) external onlyOwner nonReentrant {
        require(newContract != address(0), "Invalid contract address");
        uint256 zartcBalance = zartcToken.balanceOf(address(this));
        if (zartcBalance > 0) {
            zartcToken.transfer(newContract, zartcBalance);
        }
        emit ContractUpgraded(newContract, zartcBalance);
    }
}
```
The full contract uses euint64, ebool, and other FHE-based types.


### 💻 Frontend Example (HTML + JavaScript)
```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8" />
  <title>🎰 FHE加密轮盘投注 - 提交加密投注</title>
  <!-- 引入 ethers.js -->
  <script src="https://cdnjs.cloudflare.com/ajax/libs/ethers/5.7.2/ethers.umd.min.js"></script>
  <style>
    body {
      font-family: "Segoe UI", Tahoma, Geneva, Verdana, sans-serif;
      margin: 2rem;
      background: #f9f9f9;
      color: #222;
    }
    h1 {
      color: #d6336c;
    }
    p {
      max-width: 600px;
      line-height: 1.6;
    }
    pre {
      background: #272822;
      color: #f8f8f2;
      padding: 1rem;
      border-radius: 8px;
      max-width: 600px;
      overflow-x: auto;
      white-space: pre-wrap;
      word-break: break-word;
      margin-top: 1rem;
    }
  </style>
  <script type="module">
    import {
      initSDK,
      createInstance,
      SepoliaConfig,
    } from "https://cdn.zama.ai/relayer-sdk-js/0.1.0-9/relayer-sdk-js.js";

    async function main() {
      const output = document.getElementById("output");

      try {
        output.textContent = "🦊 请求连接钱包...";
        if (!window.ethereum) throw new Error("请先安装 MetaMask");
        await window.ethereum.request({ method: "eth_requestAccounts" });

        const provider = new ethers.providers.Web3Provider(window.ethereum);
        const signer = provider.getSigner();
        const userAddress = await signer.getAddress();

        // 合约地址，请替换成你的部署地址
        const contractAddress = "0x59130C9d2d4a369c408b1B0B9E2Df97AD09ACc9B";

        // 合约 ABI，placeBet 参数类型与合约一致
        const abi = [
          "function placeBet(bytes32, bytes32, bytes32, bytes) external",
        ];
        const contract = new ethers.Contract(contractAddress, abi, signer);

        output.textContent = "🔧 初始化同态加密 SDK...";
        await initSDK();
        const instance = await createInstance(SepoliaConfig);

        // 示例投注参数（可根据业务修改）
        const betType = 0n; // 下注类型：0单号,1红,2黑
        const number = 7n;  // 下注数字，最大36
        const amount = 10000n; // 下注金额

        // 参数校验
        if (number > 36n) throw new Error("投注数字超过轮盘最大数36");
        if (betType > 2n) throw new Error("无效的投注类型");

        output.textContent = "🔐 加密投注参数中...";
        const input = instance.createEncryptedInput(contractAddress,userAddress);

        input.add64(0n);
        input.add64(0n);
         input.add64(1n);

        const { handles: values, inputProof: attestation } = await input.encrypt();

        // 转换加密句柄和证明为hex字符串，便于合约调用
        const hexHandles = values.map((v) => ethers.utils.hexlify(v));
        const hexAttestation = ethers.utils.hexlify(attestation);

        output.textContent = "🚀 发送交易至链上...";
        const tx = await contract.placeBet(
          hexHandles[0],
          hexHandles[1],
          hexHandles[2],
          hexAttestation,
          { gasLimit: 500000 }
        );

        await tx.wait();
        output.textContent = `✅ 交易成功！交易哈希：${tx.hash}`;

      } catch (err) {
        output.textContent = `❌ 出错了:\n${err.message}`;
        console.error(err);
      }
    }

    main();
  </script>
</head>
<body>
  <h1>🎰 FHE加密轮盘投注</h1>
  <p>
    这是一个使用同态加密技术保护隐私的轮盘投注示例。  
    通过加密您的投注数据，保证您的下注内容不会被公开。  
    请确保您已安装 MetaMask 并连接到 Sepolia 测试网络。
  </p>

  <pre id="output">加载中...</pre>
</body>
</html>
```
### ❌ Common Errors Encountered

❌ 出错了:
cannot estimate gas; transaction may fail or may require manual gas limit [ See: https://links.ethers.org/v5-errors-UNPREDICTABLE_GAS_LIMIT ] (reason="execution reverted", method="estimateGas", transaction={"from":"0xcA95Fe60f102F259D47B541c2dd5603554013309","to":"0x59130C9d2d4a369c408b1B0B9E2Df97AD09ACc9B","data":"0x479394f94ae53c75ae73c70f15d2b2953fc99332decaeaf1c5000000000000aa36a70500e2bc918ff79dddb32aff0d8b06b5548183c09d6786010000000000aa36a7050089d87189b67fe77db5435dbf90568ea086e2ddedae020000000000aa36a70500000000000000000000000000000000000000000000000000000000000000008000000000000000000000000000000000000000000000000000000000000000a303014ae53c75ae73c70f15d2b2953fc99332decaeaf1c5000000000000aa36a70500e2bc918ff79dddb32aff0d8b06b5548183c09d6786010000000000aa36a7050089d87189b67fe77db5435dbf90568ea086e2ddedae020000000000aa36a7050026d419f7d136a5d21d9bd6c060befd1be1785b27a06048e41de1409800f9ca84558df11369450c5c1f3be8c626439ce6cfd48d6604d4bebe5a3652955ec871621b0000000000000000000000000000000000000000000000000000000000","accessList":null}, error={"code":3,"message":"execution reverted","stack":"o@moz-extension://d30c7c56-e37a-4bb3-adfe-9bf6171f1b71/common-4.js:1:71368\nrequest@moz-extension://d30c7c56-e37a-4bb3-adfe-9bf6171f1b71/common-1.js:3:17766\n"}, code=UNPREDICTABLE_GAS_LIMIT, version=providers/5.7.2)

### ❓ Questions & Help Needed

1. how to fix it or where the code is wrong?


### 🙏 Any help, suggestions, or example repos would be greatly appreciated!

