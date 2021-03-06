---
id: mapping-assets
title: Mapping Assets using POS
description: Build your next blockchain app on Matic.
keywords:
  - docs
  - matic
image: https://matic.network/banners/matic-network-16x9.png
---

### Introduction

Assets can be transferred in between root chain & child chain. Let's be first clear regarding nomenclature

- **Root chain/ Base chain/ Parent chain/ Layer 1** :: all are same, referring to either Goerli or Ethereum Mainnet
- **Child chain/ Layer 2** :: refers to either Matic Mumbai or Matic Matic Mainnet

For assets i.e. ERC20, ERC721, ERC1155 to be transferrable in between chains, we need to be following certain guidelines

- Assets must have required predicate contracts deployed
- Asset contract need to deployed on root chain
- Modified version of asset contract needs to be deployed on child chain
- Then they need to be mapped by calling [`RootChainManager.mapToken(...)`](https://github.com/maticnetwork/pos-portal/blob/c50e4144d90fcd63aa3d5600b11ccfff9b395fcf/contracts/root/RootChainManager/RootChainManager.sol#L165), which can only be performed by certain accounts

> For mapping i.e. the final step, make sure you check [below](#request-submission)

### Walk through

Here we're going to modify child smart contract, _given root smart contract_, for making it mapping eligible.

#### Root Token Contract

Let's moidfy [this](https://github.com/maticnetwork/pos-portal/blob/master/contracts/child/ChildToken/ChildERC20.sol) smart contract & use it as our root token contract.

```js title="RootERC20.sol"
pragma solidity 0.6.6;

import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract RootERC20 is ERC20,
{
    constructor(string memory name, string memory symbol, uint8 decimals) public ERC20(name, symbol) {
        
        _setupDecimals(decimals);
        _mint(msg.sender, 10 ** 27); // minting 10^9 MyToken, on root chain
    
    }

}
```

Lets say we've just deployed this on Goerli Testnet at `0x...`.

#### Child Token Contract

Now we need to add two functions in above defined smart contract i.e. {`deposit`, `withdraw`}.

##### Why ?

For transferring assets from root chain to child chain, we need to call [`RootChainManager.depositFor(...)`](https://github.com/maticnetwork/pos-portal/blob/c50e4144d90fcd63aa3d5600b11ccfff9b395fcf/contracts/root/RootChainManager/RootChainManager.sol#L205), which will eventually ask [`StateSender.syncState(...)`](https://github.com/maticnetwork/pos-portal/blob/c50e4144d90fcd63aa3d5600b11ccfff9b395fcf/contracts/root/StateSender/IStateSender.sol#L4), to transfer this asset from root chain to child chain, by emitting [`StateSynced`](https://github.com/maticnetwork/pos-portal/blob/c50e4144d90fcd63aa3d5600b11ccfff9b395fcf/contracts/root/StateSender/DummyStateSender.sol#L29) event. 

But before that make sure you've approved [`RootChainManagerProxy`](https://github.com/maticnetwork/static/blob/e9604415ee2510146cb3030c83d7dbebff6444ad/network/testnet/mumbai/index.json#L52) to spend equal amount of token(s), so that it can call `RootERC20.transferFrom` & start deposit. 

Once this event is emitted, our Heimdal Nodes, which keep monitoring root chain periodically, will pick up `StateSynced` event & perform call to `onStateReceive` function of target smart contract. Here our target smart contract is nothing but [`ChildChainManager.onStateReceive`](https://github.com/maticnetwork/pos-portal/blob/c50e4144d90fcd63aa3d5600b11ccfff9b395fcf/contracts/child/ChildChainManager/ChildChainManager.sol#L48).

`deposit` method which we're going to add in our smart contract, is going to be called by [`ChildChainManagerProxy`](https://github.com/maticnetwork/static/blob/e9604415ee2510146cb3030c83d7dbebff6444ad/network/testnet/mumbai/index.json#L90) & can only be called by this one.

`withdraw` method to be called on child smart contract, which will be check pointed & published on root chain as Merkel Root Proof, which then can be finally exitted by calling [`RootChainManager.exit`](https://github.com/maticnetwork/pos-portal/blob/c50e4144d90fcd63aa3d5600b11ccfff9b395fcf/contracts/root/RootChainManager/RootChainManager.sol#L279), while submitting proof.

- Token minting happens in `deposit` method.
- Tokens to be burnt in `withdraw` method.

These rules need to followed to keep balance of assets between two chains, otherwise it'll be assets created from thin air.

> Note: No token minting in constructor of child token contract.

#### Implementation

As we now know, why we need to implement `deposit` & `withdraw` methods in child token contract, we can proceed for implementing it.

```js title="ChildERC20.sol"
pragma solidity 0.6.6;

import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/math/SafeMath.sol";

contract ChildERC20 is ERC20,
{
    using SafeMath for uint256;

    constructor(string memory name, string memory symbol, uint8 decimals) public ERC20(name, symbol) {
        
        _setupDecimals(decimals);
        // can't mint here, because minting in child chain smart contract's constructor not allowed
        // _mint(msg.sender, 10 ** 27);
    
    }

    function deposit(address user, bytes calldata depositData) external {
        uint256 amount = abi.decode(depositData, (uint256));

        // `amount` token getting minted here & equal amount got locked in RootChainManager
        _totalSupply = _totalSupply.add(amount);
        _balances[msg.sender] = _balances[msg.sender].add(amount);
        
        emit Transfer(address(0), msg.sender, amount);
    }

    function withdraw(uint256 amount) external {
        _balances[msg.sender] = _balances[msg.sender].sub(amount, "ERC20: burn amount exceeds balance");
        _totalSupply = _totalSupply.sub(amount);
        
        emit Transfer(msg.sender, address(0), amount);
    }

}
```

But one thing you might notice, `deposit` function, in our implementation can be called by anyone, which must not happen. So we're going to make sure it can only be called by [`ChildChainManagerProxy`](https://github.com/maticnetwork/static/blob/e9604415ee2510146cb3030c83d7dbebff6444ad/network/testnet/mumbai/index.json#L90).

```js title="ComplaintChildERC20.sol"
pragma solidity 0.6.6;

import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/math/SafeMath.sol";

contract ChildERC20 is ERC20,
{
    using SafeMath for uint256;
    // keeping it for checking, whether deposit being called by valid address or not
    address public childChainManagerProxy;
    address deployer;

    constructor(string memory name, string memory symbol, uint8 decimals, address _childChainManagerProxy) public ERC20(name, symbol) {
        
        _setupDecimals(decimals);
        childChainManagerProxy = _childChainManagerProxy;
        deployer = msg.sender;
        // can't mint here, because minting in child chain smart contract's constructor not allowed
        // _mint(msg.sender, 10 ** 27);
    
    }

    // being proxified smart contract, most probably childChainManagerProxy contract's address
    // is not going to change ever, but still, lets keep it 
    function updateChildChainManager(address newChildChainManagerProxy) external {
        require(newChildChainManagerProxy != address(0), "Bad ChildChainManagerProxy address");
        require(msg.sender == deployer, "You're not allowed");

        childChainManagerProxy = newChildChainManagerProxy;
    }

    function deposit(address user, bytes calldata depositData) external {
        require(msg.sender == childChainManagerProxy, "You're not allowed to deposit");

        uint256 amount = abi.decode(depositData, (uint256));

        // `amount` token getting minted here & equal amount got locked in RootChainManager
        _totalSupply = _totalSupply.add(amount);
        _balances[msg.sender] = _balances[msg.sender].add(amount);
        
        emit Transfer(address(0), msg.sender, amount);
    }

    function withdraw(uint256 amount) external {
        _balances[msg.sender] = _balances[msg.sender].sub(amount, "ERC20: burn amount exceeds balance");
        _totalSupply = _totalSupply.sub(amount);
        
        emit Transfer(msg.sender, address(0), amount);
    }

}
```

This updated implementation can be used for mapping.

Steps :

- First deploy root token on root chain i.e. {Goerli, Ethereum Mainnet}
- Modify root token by adding `deposit` & `withdraw` functions & deploy corresponding child token on child chain i.e. {Matic Mumbai, Matic Mainnet}
- Then submit a mapping request, to be resolved by team.

### Request Submission

Please go through [this](/docs/develop/ethereum-matic/submit-mapping-request).
