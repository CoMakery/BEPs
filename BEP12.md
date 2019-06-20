# BEP12: Introduce customized validation scripts and transfer memo validation
## Summary
This BEP describes a new feature that enables users to customize validation scripts, and introduces the first validation script for a transaction memo. 
## Abstract
In some circumstances, users may want to specify some additional validations on some transactions. 

Taking exchange for example, for Bitcoin and Ethereum, exchanges create unique addresses for each client. It costs them too much effort to manage secret files and collect tokens to cold wallet. So now, for new blockchain platform, exchanges tend to use a single address for client deposit and require clients to input memo to identify client account information. 

However, this user experience change causes a lot of problems for both exchange customer support and clients. Clients may be panic if their tokens are missing and exchange supports have to verify the client deposit transactions manually. 

For all transfer transactions which are sending deposit tokens to exchanges, it would be nice if Binance chain can reject those who have no valid memo. Thus clients won’t be panic for losing tokens and exchanges supports won’t suffer from the heavy working load.

## Status
This BEP is WIP.
## Motivation
Currently, only exchanges can benefit from this BEP. However, this BEP proposes an important infrastructure for customized validation functions. In the future, more amazing features will be built on it. 
## Specificaion
### Add Verification Flags into AppAccount
```
type AppAccount struct {
  auth.BaseAccount    `json:"base"`
  Name                string          `json:"name"`
  FrozenCoins         sdk.Coins       `json:"frozen"`
  LockedCoins         sdk.Coins       `json:"locked"`
  Flags               uint64          `json:”flags”`
}
```
We will add a new field named “flags” into “AppAccount”. Its data type is 64bit unsigned int. Each bit will represent a validation script, which means an account can specify at most 64 validation scripts. By default, all accounts’ flags are zero. Users can send transactions to update their account flags.
### New Transaction to Update Account Flags
Parameters for Updating Account Flags

|       | Type           | Description | 
|-------|----------------|-------------|
| From  | sdk.AccAddress | Address of target account |
| Flags | uint64         | New account flags | 

Users are free to set their account flags to any values. There is not limitation for account flags. However, setting a bit which has no bonded validation script to 1 will not have any effect, unless a new validation script is bonded to it in the future. Besides, the account flags changes will take effect since the next transaction. 
 
### Memo validation
Firstly, this validation script will check the following conditions:

- The transaction type is send.
- The address is the receiving address.

Then the validation script will ensure that the transaction memo is not empty and the memo only contains digital characters. This is the pseudocode:

```
func memoValiation(addr, tx) error {
if tx.Type != “send” {
    return nil
}
if ! isReceiver(tx, addr) {
   return nil
}
if  tx.memo.length == 0 {
    return err(“tx memo is empty”)
}
if !isAllDigital(tx.memo) {
    return err(“tx memo contains non digital character”)
}
return nil
}
```

### Validations Entrance
“AnteHandler” is the entrance of customized validation scripts. To minimize the impact on performance, the following methods will be taken:
Only in check mode, the validation scritps can be called. 
To reduce the cost of calling a script, account flags will be checked before calling validation scripts.

### Impact to Upgrade
For the initial version, we just need to make a decision on which height binance chain will support the new transaction to update account flags.

In the future, more validation scripts will be supported and existing validation scripts might need to update, which means we must take scalability into consideration. 

Steps to add a new validation script:

- Implement a new validation script and call it from entrance.
- Specify since which height the new validation script will take effect.

Steps to update an existing validation script:

- Add updated validation code.
- Specify since which height the new code will take effect.


## License
All the content is licensed under [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
