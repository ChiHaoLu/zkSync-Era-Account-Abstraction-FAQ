# zkSync-Era-Account-Abstraction-FAQ

Thanks a lot to **Matter Labs** for answering below questions! 

---

## Q3

### Question

1. How does this `minialAllowance` and send a transaction with `paymasterParams` work? When I send a transaction(e.g. mint token) with `paymasterParams` which has `ApprovalBased` and `minimalAllowance=1`, the operator will handle the approve token transaction first and then get into the target transaction lifecycle(verification phase -> pay fee with paymaster -> execution phase)?
1. You say “…though if it has smaller allowance than the `minimalAllowance`, he will reset it to exactly `minialAllowance`.”. How does this “reset“ be done? This question is related to the above question if the operator does the `approve` first, the operator will choose the correct `minialAllowance` before the `approve` transaction?
1. When I send a `mint` transaction that equipped the `paymasterParams` in SDK, the operator will bundle the `approve` call(which is used for the `transferFrom` function in the future) and target transaction call(e.g. `mint`) into one transaction (which means multi-call)?
1. We have known that two conditions of `postTransaction` being called are below, but how can the zkSync operator know if the Paymaster contract has implemented the function `postTransaction` or not, does it just simulate to call it or other way?
    - Paymaster contract has implemented the function `postTransaction`
    - there is no `out of gas` error after the user’s transaction executed
1. We are curious about the hard point of the refund implementation. If the paymaster knows how much ETH they should pay and how much they will get in the refund, they can refund `refund / prepaid`  ERC20 to user? 
    - `refundToUser = prechargeFromUser * refundedETH / prepaidETH`

### Answer

1. This all happens as one big transaction. In our system even EOAs are custom accounts (just built-in everywhere where no contract has been deployed yet). So during the prePaymaster step of the account abstraction (if we used EIP4337, then it would be part of the user validation of the tx) the user's account will itself grant the allowance.
2. The logic for our default account system contract can be found here: [DefaultAccount.sol](https://github.com/matter-labs/era-system-contracts/blob/main/contracts/DefaultAccount.sol). This particular logic is described here: [TransactionHelper.sol](https://github.com/matter-labs/era-system-contracts/blob/28dbd2dc43cfa5c15e732d7e6f75aadd171cc3fd/contracts/libraries/TransactionHelper.sol#L361). Looking into the code of the account code used for EOAs can give more insight in how this works. The order of methods called for each tx is decsibred in our [documentation](  https://era.zksync.io/docs/dev/developer-guides/aa.html)
3. (2. & 3. share the answer) 
4. We just call it. We don't need to know if the paymaster implements it. As was said in one of the previous responses that unlike EIP4337 the postTransaction is not guaranteed to be successful. If it is not implemented (or implemented, but panics), it is ignored.
5. This part is unfortunately somewhat hard in the current version of zkSync. In order to keep user interfaces simpler it was decided to keep only one gasLimit (unlike EIP4337 which has 3 types of gasLimit dedicated to verification/postop, transaction itself, prevalidation (calldata costs, etc)). In EIP4337 postOp receives verificationGasLimit and so the paymaster can know *for sure* how much he will be refunded. This is not the case for zkSync, since postTransaction uses gas from the same "bucket" which the paymaster will receive the refund from. Paymaster only receives the *maximal* amount it will receive, but it needs to be self-aware of the amount of gas it will spend during the postTransaction itself, since the actual refunded gas will depend on it. That is why implementing ERC20 refund is not fully trivial right now.

---

## Q2
### Question
1. When we calculate the amount of ERC-20 which exchanges for ETH, why do We use `minimalAllowance` instead of `maximum`?
```javascript
  // Encoding the "ApprovalBased" paymaster flow's input
  const paymasterParams = utils.getPaymasterParams(PAYMASTER_ADDRESS, {
    type: "ApprovalBased",
    token: TOKEN_ADDRESS,
    // set minimalAllowance as we defined in the paymaster contract
    minimalAllowance: ethers.BigNumber.from(1),
    // empty bytes as testnet paymaster does not use innerInput
    innerInput: new Uint8Array(),
  });
```
2. Who pays for paymaster execution gas (ETH)? Is it the paymaster itself, the bootloader, or the user who triggered the transaction? My guess is that the paymaster pays the execution gas (ETH) by itself (use the ETH on the paymaster contract itself), and it will cost the same value of ERC-20 from the User account when we exchange any ERC20 token for ETH (`utils.getPaymasterParams`).
3. I see the paymaster has not supported the refund yet, so who will lose this refund gas when this feature has not been kept? The Bootloader, Paymaster, or User? I think this question is related to the problem2 above, who covers the gas cost of paymaster operations?
4. If an `out of gas` error occurs after a transaction is executed, who is the party responsible for paying the gas for that  failed execution?
5. If `postTransaction` run failed, is there any continued behavior? For example, in 4337 the `postOp` will run again if the `postOp` failed in the first time.

### Answer
1. minimalAllowance is called minimal, because this is the minimal allowance the account will ensure to give to the paymaster. If the account has already set a larger allowance, he won't reset it to a smaller, though if it has smaller allowance than the minimalAllowance, he will reset it to exactly minimalAllowance.
1. Paymaster pays with its own ETH (meaning, it must be funded). When it comes to the amount of ERC20s, this depends on paymaster implementation. Anything can be coded there (including doing an actual swap, fetching prices from an oracle, not using users tokens at all, etc). In the example she/he gave, this is how the testnet paymaster we deployed works.
1. Paymaster will receive the refund (it does so even now). It is just hard to implement the easy refund for the user (i.e. Paymaster refunding the ERC20 equivalent of the refund it has received). Besides, moving the ERC20 back to user will cost quite some gas too, so even if it was easy to support the refund for the paymaster, not sure if they would be large.
1. Paymaster
1. No, unlike EIP4337, where if the postOp fails, the users' transaction will be reverted also (and the postOp will retry), we ignore failed postOp (we don't keep this invariant that the postOp must be executed)

---

## Q1
### Question
1. I found deployment needs 712 type transaction, if we want to do normal transactions (e.g. transfer, swap), it also needs 712?
1. Will the operator or API check if the transaction is 712-type or not?
1. When we deploy a contract, MUST we call the ContractDeployer? Can I implement a CREATE2 opcode in my contract, and use it to deploy another contract, instead of calling ContractDeployer to deploy the contract?

### Answer
1. No, zkSync also supports 2718 & 1559. [REF](https://era.zksync.io/docs/dev/developer-guides/transactions/transactions.html#transaction-types)
1. Yes, but it processes multiple transaction types too if you aren't deploying a contract. 👆
1. Yes, contractDeployer must be invoked when deploying a contract, via a deployment script or from another contract like a factory. [REF](https://era.zksync.io/docs/dev/tutorials/custom-aa-tutorial.html#the-factory) 
