### EVM Part

### ✅ Issue explanation: gasFee could increase during the cross-chain call, causing the mismatch

**Core problem**:

Actual cost for cross-chain call could be higher than gasZRC20 burned causing accounting error for deposited gas token to TSS.

When we initiate the call from the Zeta to Ethereum for example the following situation could occur. The fn() determines that we need to pay 10 tokens in gas based on the current gwei price, so the 10 tokens is charged from the user on the zeta as a ‘fee’. However, during the call the gwei price on ethereum is increased, so right now the correct value charged by user would be 20 tokens. However, they were already charged as 10, so there is leftover of 10 tokens for the protocol. It is the problem, because on the Ethereum the cost of 20 tokens will be taken from the TSS wallet to execute the tx, which would break the following invariant → “*gas cost is incurred on **TSS** which is also a reserve for deposited gas token from users*”

Over time, this imbalance may prevent users from withdrawing their tokens from **ZEVM** and potentially leading to financial losses.

**What steps lead auditor to find this bug?**:

The auditor carefully examined the cross chain flow, and found out that the fee incurred is based on the gasPrice which fluctuates. And if this fluctuate it could cause the misbalance in the amount that is paid. If we paid amount A, which is heavily dependant on the current gasPrice, it could easily be turned into the amount A + 1.

**What question he posed that lead him to this bug?**

- Does the gasFee dependant on the current gasPrice?
- What if the during the fly the gasPrice increases/decreases?

**How to find it next time**:

Every time when i notice, that during the cross chain tx the fee is based on the current gasPrice, it should be a red flag. Since, the balances could mismatch, due to gas deviations.

### ✅ Issue explanation: flood the system with the 0 value transfers

**Core problem**:

This simple, but obvious bug is wrapped around the potential idea of flooding the entry of the token deposits with the 0 value deposits, which would flood the The fungible address, which is an Externally Owned Account (EOA). Since EOA could process the deposits sequentially, it could cause the using the the resources excessively and preventing from working as intended. 

### ✅ Issue explanation: **GatewayZEVM withdrawals misses amount Input Validation**

**Core problem**:

The problem here is that the cross chain mechanism is lack of the input validation related to the amount being processed. If for example we would like to deposit 100e18 tokens on zeta to be minted on the ethereum, the mismatch could occur because on the Ethereum there might be no such amount. A lot of bugs could occur from it, such as the gas tokens amount couldn’t be returned

### ✅ Issue explanation: onRevert doesn’t verify the sender

**Core problem**:

Nice one! Overall, on the ZEVM, every contract deployed must implement 2 callbacks, `onRevert` and `onCrossChainCall` .However, the problem here is that the `onRevert` is called only when the call on the dst chain revert, and we should wrapped the funds/everything back. But the problem is that the `onRevert` , which has the predefined RevertOptions struct doesn’t validate the onRevert input call is actually the call that belongs to the this contract(which implements the onRevert) 

```solidity
struct RevertOptions {
>>  address revertAddress;
    bool callOnRevert;
    address abortAddress;
    bytes revertMessage;
    uint256 onRevertGasLimit;
}
```

 What i mean by that and what could happened. 

1. I am attacker, i send the call from zeta to ethereum, but since i have the freedom of setting any revertAddress i could set the address as some contract on zeta that implements the `onRevert` callback and i can exploit somehow.
2. Assume the vulnerable address, rely on the onRevert logic, and made some actions in case it is triggered. And we as an attacker could triggered this logic and nothing would prevent us from doing it.

Eventually, to be able to resolve this issue we need to add the sender, that would prevent from triggering the onRevert of the contracts that we want

```solidity
struct RevertContext {
+    bytes sender;
     address asset;
     uint64 amount;
     bytes revertMessage;
```

### ✅ Issue explanation: Pausable modifier is missed on the zrc20.withdraw

**Core problem**:

The core problem here is that the devs simply forget to put the `whenNotPaused` modifier on the function withdraw in the zrc20 which is the native token of the ZETA chain. It means that once the Gateway’s will be stopped the users could still withdraw the tokens via the function in the native token.

```solidity

function withdraw(bytes memory to, uint256 amount) external override returns (bool) {
   (address gasZRC20, uint256 gasFee) = withdrawGasFee();
        if (!IZRC20(gasZRC20).transferFrom(msg.sender, FUNGIBLE_MODULE_ADDRESS, gasFee)) {   
            revert GasFeeTransferFailed();
        }
        _burn(msg.sender, amount);
        emit Withdrawal(msg.sender, to, amount, gasFee, PROTOCOL_FLAT_FEE);                
        return true;
    }

```

**How to find it next time**:

Carefully examined the all the opportunities of the deposit/withdraw functions and whether there is no mismatch. Also, it is important to note that the `withdraw` is come from the legacy contract, so the dev simply forget about setting the pausable functionality here because it comes from the v1.

### ✅ Issue explanation: **Forcing GAS_LIMIT in native token transfers can make Smart Contracts go out-of-gas**

**Core problem**:

Nice one! The thing here was that the ZetaChain retrieves the `gasLimit` for each token it transfers. However, the bug is about the idea that having strict `gasLimit` for ether(native token) should be discouraged. Why? Because every time we send eth to receiver, the fallback could be triggered which could consume more gas than the ZetaChain provided. 

```solidity
    function withdraw( ... ) ... {
        if (receiver.length == 0) revert ZeroAddress();
        if (amount == 0) revert InsufficientZRC20Amount();

>>      uint256 gasFee = _withdrawZRC20(amount, zrc20);
    }
    ...
    function _withdrawZRC20(uint256 amount, address zrc20) internal returns (uint256) {
        // Use gas limit from zrc20
>>      return _withdrawZRC20WithGasLimit(amount, zrc20, IZRC20(zrc20).GAS_LIMIT());
    }

```

**What steps lead auditor to find this bug?**:

Auditor carefully understood the token behaviour. When the zrc20 will be transferred to Mainnet as a native token the problem could occur. He brainstorm the question what if the hard coded limit will be not enough if the receiving contract would trigger the fallback.

**What question he posed that lead him to this bug?**

- On the dst chain, the native token will be transferred to the receiver?
- What if the receiver is the smart contract with the callback? The gasLimit could not be enough?!

**How to find it next time**:

We must +- the similar conditions → hardcoded gasLimit and opportunity to trigger the callback, it would result in a juicy bug.

### ✅ Issue explanation: **Message transfereing with onRevert() can't get recovered**

**Core problem**:

I guess the core problem was in the off-chain logic, which was oos, however we still could grasp the pitfalls of this flow. Anyway, we have the revertLogic on failed cross-chain-calls, it looks like this

```solidity
struct RevertOptions {
>>  address revertAddress;
    bool callOnRevert;
    address abortAddress;
    bytes revertMessage;
>>  uint256 onRevertGasLimit;
}
```

There is a `onRevertGasLimit`, and we pass it once the msg fails, and we need to deliver it back. However, the problem is that this variable works only during the failed CCTX call with the assets being transferred. However, the same RevertOptions struct is passed when we make the simple cross-chain-call.

```solidity
    function call(
        address receiver,
        bytes calldata payload,
>>      RevertOptions calldata revertOptions
    ) ... {
        if (receiver == address(0)) revert ZeroAddress();
        emit Called(msg.sender, receiver, payload, revertOptions);
    }
```

The dev stated that if the asset-CCTX call failed, the `onRevertGasLimit` would be taken as a piece from the assets being transferred. But, what if we have no assets, because we make the simple call? Yes, we end up in the bad situation.

**What steps lead auditor to find this bug?**:

Auditor found this bug via the communication with the dev. It is the lesson for me. Once i see some strange param such as gasLimit, that someone need to pay (me or protocol), i need to ask about it immediately, because on this contest i ‘assumed’ that it would be paid from the protocol pocket, that’s why i missed it.  

**What question he posed that lead him to this bug?**

- `onRevertGasLimit` - who pays for it? User or protocol?
- What if i provide 0 value? How it would be handled?
- If the protocol use it, what if it would be not enough? During the callback for example>

**How to find it next time**:

Ask more questions about ambiguous parameters. Don’t ‘**assume**’ that it would be payed/executed by someone.

### ✅  Issue explanation: **Performing a second CCTX during a current CCTX will fail due to nonReentrant**

**Core problem**:

The thing here is about **nonReentrant** modifier. Assume the user initiate the cctx call on the Mainnet, this tx reaches the ZetaChain, and once it reaches the dst chain, the cctx call is triggered there, to deliver the tokens/some information back to the Mainnet. In such scenario the cross-chain-call would revert due to **nonReentrant** modifier, which would prevent from the execution. Overall, due to the `nonReentrant` modifier, such "ping-pong" communication patterns are not possible on `GatewayEVM`. In contrast, `GatewayZEVM` does not have this restriction, allowing such protocols to function.

**What steps lead auditor to find this bug?**:

Overall, the auditor imagined the scenario “what if”, what if the user/dapp send the cctx but then decides to atomically sends it back? He compared the 2 Gateways and found out that the intended functionality could be broken.

**What question he posed that lead him to this bug?**

1. What if we would send the cctx to the dst chain and then immediately send it back? Would it proceed?
2. What if the dapp would require to ping pong the message via cross-chain? Would it work?
3. Is the `nonReenterant` modifier is needed?

**How to find it next time**:

Some dapp/user would require very strange conditions during the cross-chain. And one of such condition is ping-pong communication, and we must ensure that it wouldn’t be blocked via some modifiers/checks.

### ✅ Issue explanation: **data availability fee isn’t taken into an account during the fee calculation**

**Core problem**:

We must remember that if we communicate with the Optimism/Base chains we need take into an account the data availability fee (which is basically the price of storing the data on L2)

 

```solidity
Optimism Transaction Fee = [ Fee Scalar * L1 Gas Price * (Calldata + Fixed Overhead) ] + [ L2 Gas Price * L2 Gas Used ]
```

As we can see the first component, `[ Fee Scalar * L1 Gas Price * (Calldata + Fixed Overhead) ]`, is the data availability fee and the second component, `[ L2 Gas Price * L2 Gas Used ]`, is the execution cost. 

However, based on the function that calculates the gas that needs to be charged we could see that no data availability cost is charged.

```solidity
    function withdrawGasFeeWithGasLimit(uint256 gasLimit) public view override returns (address, uint256) {
    /**
     * @dev Withdraws gas fees with specified gasLimit
     * @return returns the ZRC20 address for gas on the same chain of this ZRC20, and calculates the gas fee for
     * withdraw()
     */
    function withdrawGasFeeWithGasLimit(uint256 gasLimit) public view override returns (address, uint256) {
        address gasZRC20 = ISystem(SYSTEM_CONTRACT_ADDRESS).gasCoinZRC20ByChainId(CHAIN_ID);
        if (gasZRC20 == address(0)) revert ZeroGasCoin();

        uint256 gasPrice = ISystem(SYSTEM_CONTRACT_ADDRESS).gasPriceByChainId(CHAIN_ID);
        if (gasPrice == 0) {
            revert ZeroGasPrice();
        }
        uint256 gasFee = gasPrice * gasLimit + PROTOCOL_FLAT_FEE;
        return (gasZRC20, gasFee);
    }
    }
```

 Eventually, the off-chain entity that deliver the cctx would be charged more that needed → resulting in kind of griefing.

**What steps lead auditor to find this bug?**:

Carefully examined the behaviour of the chains that would be integrated with the Zeta Protocol. Understanding of the data availability cost

**What question he posed that lead him to this bug?**

- Does the protocol plan to be integrated with the OP Stack chains?
- Is the DA cost is taken into account, is it accounted during the fee calculation?
- Are there any other chains that impose the DA cost?

**How to find it next time**:

DA cost must be charged during the cctx fee calculation if we work with OP Stack chains and potentially some others in the future.

### ✅ Issue explanation: tssAddress must be allowed to be re-set

**Core problem**:

From the first glance, very simple issue, but it is what it is → missed.

The **tssAddress** is an important address that is basically the core of the Zeta protocol functionality, however there is no method that allow to reset it. That’s why we need to ensure it, because still it is the threshold siganture address that could become buggy or compromised.

### ✅ Issue explanation: if the user-overpay the gasFee isn’t refunded back.

**Core problem**:

Classical issue in the cross-chains protocols, when the fee is charged from the user. If the fee is over-charged it must be ensured that the remaining is refunded, in case the user could loose the money since it is difficult to predict the exact input amount of the fees.

```solidity
function call(
        bytes memory receiver,
        address zrc20,
        bytes calldata message,
        uint256 gasLimit,
        RevertOptions calldata revertOptions
    )
        external
        nonReentrant
        whenNotPaused
    {
        if (receiver.length == 0) revert ZeroAddress();
        if (message.length == 0) revert EmptyMessage();

        (address gasZRC20, uint256 gasFee) = IZRC20(zrc20).withdrawGasFeeWithGasLimit(gasLimit);
        if (!IZRC20(gasZRC20).transferFrom(msg.sender, FUNGIBLE_MODULE_ADDRESS, gasFee)) {
            revert GasFeeTransferFailed();
        }

        emit Called(msg.sender, zrc20, receiver, message, gasLimit, revertOptions);
    }
```

For example here, the user passes the gasLimit, based on what the gasFee is later calculated, however, there is no refundAddress implemented that would receive the unspent amount back.

**What steps lead auditor to find this bug?**:

Popular cross-chain protocols, such as [Wormhole](https://wormhole.com/docs/tutorials/messaging/cross-chain-contracts/#feature-receive-refunds) implements the refund logic. So when we deal with the fees calculation we need to take a lot of conditions take into an account,  that’s why it is the most buggy edge.

**What question he posed that lead him to this bug?**

1. Does the fee refunded to the user?
2. What if the user provided more that intended, will it be simply lost?
3. Could the user overpay? What would happen in such case?

**How to find it next time**:

Ensure that the user’s gasfee is refunded in case it is over payed.

### ✅ Issue explanation: **Lack of Context in Arbitrary Call Execution Leading to Potential Unauthorized Actions**

**Core problem**:

Very interesting issue that i was thinking long enough. When we initiate the cctx from the zeta to evmChain, on the gateway we have the peace of the code that executes the arbitrary call to the dst entity from the TSS(trusted address) via Gateway, it looks like this

```solidity
function _execute(address destination, bytes calldata data) internal returns (bytes memory) {
        (bool success, bytes memory result) = destination.call{ value: msg.value }(data);
        if (!success) revert ExecutionFailed();

        return result;
    }
```

However, the problem is: assume the dst contract is some dApp that heavily rely on this call, and it implements some important functionality based on it. However, there is no context related to this call! What i mean by that is:

1. Yes, the dst contract could check that the msg.sender is the Gateway.sol, but it is still not enough, because any user could encode any data into the `result` field and initiate an cross-chain transfer(from the zeta chain)
2. The main point is, that there should be come context added, so the dapp that is integrated with the Gateway/Zeta will not be abused by providing the malicious call, where only the msg.sender could be checked.

**What steps lead auditor to find this bug?**:

The auditor think in a way, how the external entity that relies on the Zeta could be damaged, because by nature this arbitrary call wouldn’t harm the protocol itself. While i concentrated solely how it would damage the protocol itself, i should think in a way how the receiver would suffer. And because actually could initiate the call, it would cause a troubl 

**What question he posed that lead him to this bug?**

1. Who are expected receiver of this call?
2. What information is encoded into the call?
3. What if the receiver would rely on it, but the malicious sender from the source chain could forge the call, what would happen?

**How to find it next time**:

Try to think more in a way, how the call could harm the receiver, not the protocol itself. Lack of the context could also be very damageable, take as an example the Layer Zero, where it is taken into an account.

### ✅ Issue explanation: **When user withdraw ZETA on ZEVM, gas slippage cannot be set**

**Core problem**:

This bug was discovered via the communication with the dev team, during the cctx communication on withdrawing zeta from the ZetaChain, no gasFee is paid , consequently they stated that during the cctx communication, *the protocol will automatically reduce the amount in the crosschain-tx (this is not represented at the smart contract level), swap the ZETA for the gas tokens of the external chain with a in-protocol Uniswap pool and burn the tokens to pay for the fees*. And it lead to the bug, that the slippage which is come from the user isn’t set. 

'''
@project Why doesn't external calling via transfer ZETA need to pay fees like calling via call? Additionally, why doesn't withdraw ZETA need to pay gas fees?
Yes we should add comment on this. The protocol will automatically reduce the amount in the crosschain-t (this is not represented at the smart contract level), swap the ZETA for the gas tokens of the external chain with a in-protocol Uniswap pool and burn the tokens to pay for the fees.
'''

For example:

1. Alice performs a withdraw to transfer ZETA from ZetaChain to Ethereum, where the amount is 10 ZETA.
2. Alice expects that 1 ZETA will be used as the `gasFee` required for the withdrawal.
3. However, when Alice's transaction is executed, the gas price has increased, and the exchange rate between ZETA and ETH has decreased, requiring 2 ZETA to complete the operation.
4. Since the contract does not have a parameter for Alice to set her expected amount to be received, Alice ultimately receives 8 ZETA instead of the expected 9 ZETA.

**What steps lead auditor to find this bug?**:

It wasn’t clear enough why the gas isn’t paid. And a lot of people assumed that it is a bug that must could abused, while in the reality it was a design decision, that covers under the hood the high vulnerability. So, the simple question should be ask! Many questions should be asked to the dev.

**How to find it next time**:

If some part is very obvious, but very-very obvious that could result in the med/high it is better to ask the dev, since you don’t loose nothing, maybe it is a design decision, that only you will be aware about, thus find a unique.

### ✅ Issue explanation: **Attacker can create a fake onRevert call**

**Core problem**:

Very interesting finding. Remember we have the option to execute the arbitrary call? Here the attacker could exploit this peace of the code to encode the call to the dapp/contract that has the `onRevert()` method. This would allow the attacker to potentially steal the funds that are stored in the contract.

1. Make an Omnichain call `Zeta -> EVM`.
2. We made the receiver one of the `Universal Apps` on the destination `EVM` chain that supports `onRevert()`.
3. Call a function selector `onRevert()`, and add `RevertContext` as parameter input.
4. ZetaClients process the message.
5. They made that arbitrary call to the `Univeral App`.
6. The GatewayEVM, which is a trusted entity, calls the `onRevert()` function on the Universal App, passing the fraudulent data.
7. The Universal App, believing this is a legitimate recovery request (because it trusts GatewayEVM), executes its recovery logic and transfers tokens back to the attacker, even though these tokens were never truly transferred in the first place.

**What steps lead auditor to find this bug?**:

Carefully examined how to harm the dapp/protocol that is integrated with the protocol.

**What question he posed that lead him to this bug?**

1. `onRevert` and `onCrossChain` call is called on the dst entity, are there any opportunity to call it from the untrusted entity?
2. What if we could call these methods from the different person?

### ✅ Issue explanation: **Incorrect protocol behaviour when making empty-payload cross-chain calls**

**Core problem**:

Interesting! We have 2 ways of deposit function. First we make the simple deposit, while the second one we make the depositAndCall, and the only way how the off-chain node would differentiate them is via the payload field (first one is empty, while second one must be non-empty)

```solidity
function deposit(
    address receiver,
    RevertOptions calldata revertOptions
)
    external
    payable
    whenNotPaused
    nonReentrant
{
    if (msg.value == 0) revert InsufficientETHAmount();
    if (receiver == address(0)) revert ZeroAddress();

    (bool deposited,) = tssAddress.call{ value: msg.value }("");

    if (!deposited) revert DepositFailed();

    emit Deposited(msg.sender, receiver, msg.value, address(0), "", revertOptions);
}
```

```solidity
function depositAndCall(
    address receiver,
    bytes calldata payload,
    RevertOptions calldata revertOptions
)
    external
    payable
    whenNotPaused
    nonReentrant
{
    if (msg.value == 0) revert InsufficientETHAmount();
    if (receiver == address(0)) revert ZeroAddress();

    (bool deposited,) = tssAddress.call{ value: msg.value }("");

    if (!deposited) revert DepositFailed();

    emit Deposited(msg.sender, receiver, msg.value, address(0), payload, revertOptions);
}
```

However, the problem is that assume that the user would like to call the contract on the dst chain with `onCrossChainCall` with an empty bytes payload, but he couldn’t succeed because it would be evaluate as an “simple deposit”, thus it will be never executed, which violates the rules.

<aside>
💡

This is because if now a user wants to deposit assets and call the `onCrossChainCall` function with an empty payload, the observers will think it is a simple deposit and omit the call on zetachain, calling `GatewayZEVM::deposit` instead of `GatewayZEVM::depositAndCall`.

</aside>

**What steps lead auditor to find this bug?**:

He take a look on the event that is emmited and found out that would we end up if we would not provide the payload during the call - rule violation. Simply, take a look on the event.
`emit Deposited(msg.sender, receiver, msg.value, address(0), "", revertOptions);`

**What question he posed that lead him to this bug?**

1. What if we call the contract on the dst chain with an empty payload? How it would be handled?

### ✅ Issue explanation: **The GatewayEVM contract executing arbitrary calldata can be exploited by malicious users to steal other users' fund**

**Core problem**:

The problem is that if the user which often interact with the Gateway could give a huge allowance, the attacker could exploit it via the arbitrary call.

- **Attack Prerequisites**
    - The user `victim` has existing allowances to the `GatewayEVM` being the `spender`. This is the only prerequisite for the attack to work.
    - This prerequisite is actually highly likely to be true in reality. The reason is that, the `GatewayEVM` contract has functions such as `depositAndCall()` and `deposit()` for ERC20 transfer, both of which require the user to have some pre-existing allowances in place to call these functions. Therefore, it should be quite common to see the frequent users of this protocol approving the `GatewayEVM` to spend their ERC20s; and, very often the users tend to approve for a higher amount in reality, in order to reduce repetitive approvals if they are frequent users.
- **Attack Scenario**
    - Once the attack prerequisite is met, then the attack scenario is rather simple. First, the `attacker` initiated a call on the zetachain, the gateway contract on zetachain emitted the event `Called`, with the arbitrary address `receiver` being an ERC20 address and the calldata `message` being the `transferFrom()` related calldata.
    - Then, the `TSS` on the connected EVM chain passed this info to `GatewayEVM` to execute it.
    - Because the `GatewayEVM` is the `spender` for potentially many users on many different ERC20s, it can successfully execute the trxn like `transferFrom(victim, beneficiary, amount)`. The `beneficiary` is just the `attacker`'s address on the connected chain.
    - If this scenario plays out, mass stealing can happen.

### ✅ Issue explanation: **Unwhitelisted tokens will block withdrawal on EVM chains**

**Core problem**:

The problem that on the ZEVM there still could exist a pair between the zeta and erc20 tokens, which circulate, while in the meantime the erc20 token is actually unwhitelisted on the EVM chains. It could lead to the problems, because the user could withdraw this token from the ZEVM but can’t deposit it back.

```solidity
function withdraw(
    address to,
    address token,
    uint256 amount
)
    external
    nonReentrant
    onlyRole(WITHDRAWER_ROLE)
    whenNotPaused
{
    if (!whitelisted[token]) revert NotWhitelisted();

    IERC20(token).safeTransfer(to, amount);

    emit Withdrawn(to, token, amount);
}
```
