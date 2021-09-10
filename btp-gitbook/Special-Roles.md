### Special Authorization Roles in Contract Upgradeability

____

The Blockchain Transmission Protocol (BTP) entirely relies on a set of three types of smart contracts, `BSH`, `BMC`, and `BMV`, on each connected network. In these contracts, there are some tasks/functions restricting who is allowable to access and to change/update crucial state variables. Hence, an ownership role is proposed to take responsibility for managing and maintaining services, features, and crucial data of a contract. In addition, Contract Upgradeability is an additional feature that the ICON Foundation proposes to provide developers ease-to-modify smart contracts for some cases, i.e., fixing bugs, adding additional features during the development of their projects. Thus, this task is also required a special authorizer to operate.

This document helps to differentiate some special roles in BTP Network with a feature of Contract Upgradeability. If you need to know what Contract Upgradeability is and how to upgrade a contract, please check out this link [here](./Readme.md).

In BTP Network with a feature of Contract Upgradeability, there is a total of three special authorization roles:

- `Owner`: this is a special role of a smart contract. As proposed by the ICON Foundation, each of the contracts can be managed by one or multiple `Owners`.

| Feature 	| Action 	| Description 	|
|-	|-	|-	|
| Ownership 	| Add Owner 	| - Current Owner can add a new Owner <br> - Initial Owner is an account that deployed a contract 	|
|  	| Remove Owner 	| - Current Owner can remove another Owner <br> - The last Owner cannot remove him/herself (Must have at least 1 owner) 	|	|

- `Proxy Admin`: is a contract, generated by Openzeppelin Plugin, which manages to upgrade `Proxy` contracts.
- `Owner` of `Proxy Admin`: there is only one `Owner` in the `Proxy Admin` contract. To differentiate this role from the above ones, `Owner` of `Proxy Admin` contract will be called `Admin of Upgradeability` or `Operator`.

### Operator vs Proxy Admin vs Owners

____

Assume, we have a scenario as follows: `Operator A` deploys Contract1, Contract2, and Contract3 with upgradeability feature using Openzeppelin Plugin Library

<p align="center">
  <img src="./images/Deploy-Case.png" />
</p>

- After deployment, there are seven contracts being generated
  - `Proxy Admin`: a contract that manages to upgrade Proxy contracts
  - Three `TransparentUpgradeableProxy` (a.k.a `Proxy`): the contracts that all Users would directly interact with instead
  - And three logic contracts: `Contract1`, `Contract2`, and `Contract3`

<p align="center">
  <img src="./images/Explanation.png" />
</p>

- **Note that:**
   - `Proxy Admin` is a contract
   - There's only one `Proxy Admin` contract to manage an upgradeability of three `Proxy` contracts
   - `Proxy Admin` contract is owned by `Operator A`. To upgrade contracts, only `Operator A` is allowed to perform this action.
   - Meanwhile, `Operator A` also has an `Owner` role of `Contract1`, `Contract2`, and `Contract3`

- `Operator A` calls `Proxy` contracts to add more owners
   - Contract2: `Operator A` calls to add an `Owner22`.
   - Contract3: `Operator A` calls to add `Owner32` and `Owner33`

<p align="center">
  <img src="./images/Add-Owners.png" />
</p>

- **Note that:**
   - `Contract2` is owned by `Operator A` and `Owner22`
   - `Contract3` is owned by `Operator A`, `Owner32`, and `Owner33`
   - `Owner22`, `Owner32`, and `Owner33` are distinctive owners of `Contract2` and `Contract3` even though `Operator A` has an ownership role in these two contracts. Thus, `Owner22` does not have an ownership role in `Contract3` and the same on others.

- `Owner22` calls `Proxy` contract to remove `Operator A` while `Owner33` calls another `Proxy` contract to remove `Owner32`

<p align="center">
  <img src="./images/Remove-Owners.png" />
</p>

- `Operator B` deploys Contract4 with upgradeability feature using Openzeppelin Plugin Library

<p align="center">
  <img src="./images/New-ProxyAdmin.png" />
</p>

- **Note that:**
   - There are two `Proxy Admin` contracts - `Proxy Admin` manages an upgradeability of {`Contract1`,`Contract2`, and `Contract3`} and `Proxy Admin 2` manages an upgradeability of {`Contract4`}
   - `Proxy Admin` is owned by `Operator A` while `Operator B` is an owner of `Proxy Admin 2`


- `Operator A` transfers upgradeable management of `Contract3` to `Proxy Admin 2` via using Openzeppelin Util `changeProxyAdmin(ProxyAddr, newProxyAdminAddr)` 
   - `ProxyAddr`: An address of `Proxy` contract that needs to update a new **Proxy Admin**
   - `newProxyAdminAddr`: an address of a new **Proxy Admin** contract
   - For example: `changeProxyAdmin(ProxyContract3.address, ProxyAdmin2.address)` in this case

```javascript
const Contract1 = artifacts.require('Contract1');
const Contract2 = artifacts.require('Contract2');
const Contract3 = artifacts.require('Contract3');
const ProxyAdmin = artifacts.require('ProxyAdmin');
const { deployProxy, upgradeProxy, admin } = require('@openzeppelin/truffle-upgrades');
const { assert } = require('chai');
const truffleAssert = require('truffle-assertions');

contract('Upgradeable contracts', (accounts) => {
    let c1, c2, c3, newProxyAdmin, adminInstance;

    it('Deploy Proxy Contracts', async() => {
        var _uri = 'https://1234.iconee/';
        var _native_coin = 'PARA';
        c1 = await deployProxy(Contract1, [_uri, _native_coin]);
        c2 = await deployProxy(Contract2, [_uri, _native_coin]);
        c3 = await deployProxy(Contract3, [_uri, _native_coin]);
    });

    it('Create a new Proxy Admin', async() => {
        //  ProxyAdmin.sol is provided by Openzeppelin
        newProxyAdmin = await ProxyAdmin.new();
    });

    it('Change Proxy Admin', async() => {
        await admin.changeProxyAdmin(c3.address, newProxyAdmin.address);
    });
});
```

<p align="center">
  <img src="./images/change-ProxyAdmin.png" />
</p>

<p align="center">
  <img src="./images/Transfer-ProxyAdmin.png" />
</p>

- We have a final result as following:
   - `Proxy Admin` manages an upgradeability of {`Contract1`, and `Contract2`}
   - `Proxy Admin 2` manages an upgradeability of {`Contract3`, and `Contract4`}
   - `Proxy Admin` contract is owned by `Operator A`. To upgrade `Contract1` and `Contract2`, only `Operator A` has the authorization role to perform this action.
   - Meanwhile, `Operator B` has an ownership role of `Proxy Admin 2`. Thus, only `Operator B` has the authorization role to upgrade `Contract3` and `Contract4`
   - `Operator A` is authorized to manage logic/feature implementation of `Contract1` and `Contract3`
   - In `Contract3`, `Owner33` has the same role as `Operator A`. These two clients collaborate to manage the `Contract3`
   - `Owner22` is authorized to manage logic/feature implementation of `Contract2`
   - And in the `Contract4`, `Operator B` has an authorization role to manage this contract (full management - upgradeability and logic/feature implementation)

### Frequently Asked

____

*Question: Is Proxy Admin a single account that can upgrade a smart contract?*

**Answer:** No, `Proxy Admin` is a contract, generated by Openzeppelin, which manages an upgradeability of one or multiple `Proxy` contracts

____

*Question: Can the Proxy Admin role be transferred?*

**Answer:** There's no Proxy Admin role. Instead, that's a contract. Thus, it can be replaced (not transferred)

____

*Question: Is it correct that Owners are accounts who own a specific component such as BMC and BSH?*

**Answer:** Yes, the `Owner` is an account, which has a special authorization to perform/to access some crucial features/logic implementation of a specific contract. Please be aware that each contract, i.e., `BSH` and `BMC`, implements its own logic to manage this ownership role while the `Proxy Admin` contract, created by Openzeppelin, uses a library ([Ownable.sol](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/access/Ownable.sol)). Hence, transferring ownership role of logic implementation contract does not affect an ownership role of `Proxy Admin` contract and vice versa

____

*Question: Proxy Admin exists per smart contract. Does it mean a proxy admin of BSH and BMC can be different?*

**Answer:** It's not completely correct. It depends on a specific circumstance. Please take a look at the example above

____

*Question: Owner exists per component. If BSH consists of BSHCore and BSHPeriphery contracts, does it mean that BSHCore's Owners and BSHPeriphery's Owners are always the same?*

**Answer:** It's not really correct. As our design, `BSHCore` has some functions that restrict only the `Owner` to access while `BSHPeriphery` does not have this restriction. Thus, the `BSHCore` contract has the ownership feature, but `BSHPeriphery` does not. In addition, `BSHCore` would manage a list of `Owners` in a case that `BSHPeriphery` needs this feature as well in the future.

____

*Question: Is the one, who deploys a contract, an initial proxy admin, and an initial owner?*

**Answer:** It's correct for an ownership role of a contract. It means that **the one who deploys contracts is the initial owner of logic contracts and the** `Proxy Admin` **contract**

____

*Question: Where is the method to transfer the proxy admin?*

**Answer:**  Openzeppelin provides two methods `changeProxyAdmin()` and `transferProxyAdminOwnership()`

- `changeProxyAdmin()` would replace Proxy Admin of a Proxy contract with another one
- `transferProxyAdminOwnership()` would transfer an ownership role from the initial Owner to another one

____

*Question: Is it right that only one account (Operator A in the above example) can manage the Proxy Admin contract?*

**Answer:** Yes, it's correct

____

*Question: Is it right that ownership of `Proxy Admin` can be transferred by calling transferOwnership()?*

**Answer:** In order to transfer this ownership, OpenZeppelin provides a utility script to support it. It is recommended to use their provided method `transferProxyAdminOwnership()` instead

____

*Question: Could it be many `Proxy Admin` contracts that manage the same `Proxy` contract?*

**Answer:** No, it's not correct. One `Proxy Admin` can manage many `Proxy` contracts. However, one `Proxy` contract can be managed by only one `Proxy Admin`

____

*Question: Does it imply that a client/an account, which can upgrade BMC, can also upgrade BSH?*

**Answer:** If these contracts are deployed by one **Operator**, it's likely that only one `Proxy Admin` manages multiple `Proxy` contracts. Thus, only the `Owner` of that `Proxy Admin`, which is **Operator**, has the ability to upgrade `Proxy` contracts in case of not having any prior transferring ownerships