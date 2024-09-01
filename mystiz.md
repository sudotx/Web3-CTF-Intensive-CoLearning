---
timezone: Asia/Taipei
---

# Mystiz

1. 自我介紹

Software Engineer @google. CTF since 2017. Blog [here](https://mystiz.hk/about-me/).

2. 你認為你會完成本次殘酷學習嗎？

Can't say yes, but I hope this could get me motivated.

## Notes

<!-- Content_START -->

### 2024.08.28

1. 設定 Foundry 環境。https://book.getfoundry.sh/getting-started/installation
2. 把 Damn Vulnerable DeFi、EthTaipei CTF 2023 跟 MetaTrust CTF 2023 的題目先打包下來備用。
    * 把 EthTaipei CTF 2023 的答案刪掉。
    * MetaTrust CTF 2023 repo 要轉換成 Foundry 的模式，到解題的時候再把它們轉換過去。

目標：完成 Damn Vulnerable DeFi + EthTaipei CTF 2023 及 MetaTrust CTF 2023。

### 2024.08.29

Progress

* Damn Vulnerable DeFi (2/18)
* EthTaipei CTF 2023 (0/5)
* MetaTrust CTF 2023 (0/22)

#### 📚 Reading: EIP for Flash Loans

Reference: https://eips.ethereum.org/EIPS/eip-3156

> A flash loan is a smart contract transaction in which a lender smart contract lends assets to a borrower smart contract with the condition that the assets are returned, plus an optional fee, before the end of the transaction.

#### 🔨 Foundry debugging

```solidity
pragma solidity =0.8.25;

import {IERC3156FlashBorrower} from "@openzeppelin/contracts/interfaces/IERC3156.sol";

contract ExploitContract is IERC3156FlashBorrower {
    function onFlashLoan(
        address initiator,
        address token,
        uint256 amount,
        uint256 fee,
        bytes calldata data
    ) external override returns (bytes32) {
        // Take the money from receiver
        

        return keccak256("IERC3156FlashBorrower.onFlashLoan");
    }
}
```

```bash
# We can use -vvvv to make the log super-verbose
forge test --match-test test_unstoppable -vvvv
```

This is what it looks when `vault.flashLoan(exploit, address(token), 1e18, bytes("00"));` is called, when my exploit contract is given above.

![](Writeup/mystiz/images/20240829-dvd-unstoppable-1.png)

#### 🏁 Damn Vulnerable DeFi: Unstoppable

**Time used: ~1h 55m**

The goal of the challenge is to pause the vault. One way to trigger is to make the `vault.flashLoan` raise an exception -- thus the vault will be stopped when `isSolved` is called.

```solidity
try vault.flashLoan(this, asset, amount, bytes("")) { /* omitted */ } catch { /* pauses the vault here! */ }
// called in src/unstoppable/UnstoppableMonitor.sol:checkFlashLoan(100e18) (line 41)
// called in test/unstoppable/Unstoppable.t.sol:_isSolved() (line 106)
```

To make this happen, we can make `convertToShares(totalSupply) != balanceBefore`. In that's the case, the transaction will be reverted -- and thus raising an exception. We can simply transfer 1 DVT to the vault to halt the contract.

#### 🏁 Damn Vulnerable DeFi: Naive Receiver

**Time used: ~5h 35m**

The goal of the challenge is to drain the WETH from the `NaiveReceiverPool` and the `FlashLoanReceiver` contracts (which initially had 1000 WETH and 10 WETH).

Ideas:

1. We can call `pool.flashLoan(receiver, address(weth), 0, bytes(""));` and it will take 1 WETH away from `FlashLoanReceiver` each time.
1. We can use `BasicForwarder` to make `_msgSender()` to be the `NaiveReceiverPool`. However, it will always append `request.from` (you need its private key). We can bypass by calling `Multicall.multicall` from `BasicForwarder.execute`. In that way, the appended `request.from` will not be used.

### 2024.08.30

Progress

* Damn Vulnerable DeFi (4/18)
* EthTaipei CTF 2023 (0/5)
* MetaTrust CTF 2023 (0/22)

#### 🏁 Damn Vulnerable DeFi: Truster

**Time used: ~1h 40m**

The goal of the challenge is to drain the token from `TrusterLenderPool`. Additionally `flashLoan` is protected by the re-entrancy guard.

Function definition for `Address.functionCall`: https://github.com/OpenZeppelin/openzeppelin-contracts/blob/release-v5.1/contracts/utils/Address.sol#L62-L64

Since `token` is a ERC20 token, why not use the `approve` method? This allows our attacking contract to spend money on behalf of the pool, thus we can transfer funds out of the pool afterwards.

#### 📚 Reading: Re-entrancy attack

References:

- https://medium.com/mr-efacani-teatime/%E9%96%92%E8%81%8A%E5%8A%A0%E5%AF%86%E8%B2%A8%E5%B9%A3%E6%9C%80%E6%99%AE%E9%81%8D%E7%9A%84%E6%94%BB%E6%93%8A%E6%89%8B%E6%B3%95-re-entrancy-attack-ea63e90da7a7
- https://solidity-by-example.org/hacks/re-entrancy/
- https://github.com/pcaversaccio/reentrancy-attacks?tab=readme-ov-file (⭐ Collection of re-entrancy attacks)

#### 🏁 Damn Vulnerable DeFi: Side Entrance

**Time used: ~45m**

We create an `ExploitContract` to drain the funds from `SideEntranceLenderPool`. To start with, we call `flashLoan` to get 1000 ETH. We then deposit the money to the pool. Since the pool's balance is unchanged, it is considered repayed. The only difference is, we have 1000 ETH deposited to the pool.

After that, we can simply withdraw the amount to the recovery wallet.

### 2024.08.31

Progress

* Damn Vulnerable DeFi (5/18)
* EthTaipei CTF 2023 (0/5)
* MetaTrust CTF 2023 (0/22)

#### 🏁 Damn Vulnerable DeFi: The Rewarder

**Time used: ~1h 15m**

Important: We are also rewarded. `0x44E97aF4418b7a17AABD8090bEA0A471a366305C` appeared on line 755 in both files. We are the 188th entry.

To (almost) drain the reward distributor, we can repeatedly send the same claim in the same transaction. This is because `_setClaimed` will be called once.

<!-- Content_END -->