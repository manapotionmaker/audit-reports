
# Nouns security review by maᴎa
***review commit hash* - [23d64ac7093f504ad4731bc4cf8d41b2c2943657](https://github.com/nounsDAO/token-buyer/tree/23d64ac7093f504ad4731bc4cf8d41b2c2943657)**

# High:
### [H-01] - Upper bound price range does not fall within Chanlink's range

#### Description:
PriceLowerBound is set to $100 * 10^{18}, and priceUpperBound is set to $100,000 * 10^{18} which translate to $100 and $100K respectively.

This is the Chainlink oracle PriceFeed is using: 0x5f4eC3Df9cbd43714FE2740f5E3616155c5b8419 which uses this aggregator: 0x37bC7498f4FF12C19678ee8fE19d713b87F6a9e6. minAnswer and maxAnswer values from this aggregator are: $1 * {10^8}$ and $10,000 * {10 ^ 8}$ which translate to $1 and $10K. Chainlink will never report prices outside this range.

The upper bound of $100K set in PriceFeed is not useful as the check priceWad > priceUpperBound is always false:

```solidity
if (priceWAD < priceLowerBound || priceWAD > priceUpperBound) {
    revert InvalidPrice(priceWAD);
}
```

#### Impact:
If the market price moves above the upper bound specified by Chainlink, TokenBuyer will allow trades in the loss for Nouns DAO until the staleness check kicks in. staleAfter is set to 1 hour currently.

#### Lines affected:
https://github.com/nounsDAO/token-buyer/blob/23d64ac7093f504ad4731bc4cf8d41b2c2943657/src/PriceFeed.sol#L93

#### Remedial recommendation & POC:
Set priceUpperBound to a value < $10,000 * 10^{18}$. Chainlink reports a new price in two scenarios:

1) The price moves beyond a certain deviation threshold.
2) The price has not been reported within a certain amount of time (aka heartbeat).

# Low:
### [L-01] - Avoid Floating pragma

#### Description:
Contracts should be deployed with the same compiler version and flags that they have been tested the most with. Locking the pragma helps ensure that contracts do not accidentally get deployed using, for example, the latest compiler which may have higher risks of undiscovered bugs.

```solidity
pragma solidity ^0.8.15;
```

#### Impact:
Possible unintended consequences due to version updates.

#### Lines affected:
- https://github.com/nounsDAO/token-buyer/blob/23d64ac7093f504ad4731bc4cf8d41b2c2943657/src/Payer.sol#L16
- https://github.com/nounsDAO/token-buyer/blob/23d64ac7093f504ad4731bc4cf8d41b2c2943657/src/PriceFeed.sol#L16
- https://github.com/nounsDAO/token-buyer/blob/23d64ac7093f504ad4731bc4cf8d41b2c2943657/src/TokenBuyer.sol#L16

#### Remedial recommendation & POC:
Fix the pragma version for example `pragma solidity ^0.8.15;`
```diff
- pragma solidity ^0.8.15;
+ pragma solidity 0.8.15;
```



### [L-02] - setAdmin does not check for zero address

#### Description:
The `setAdmin(address newAdmin)` function does not check if the new owner is a zero address.

```solidity
function setAdmin(address newAdmin) external onlyAdminOrOwner {  
    emit AdminSet(admin, newAdmin);  
  
    admin = newAdmin;  
}
```

#### Impact:
This could have unintended results such as accidentally setting the zero address as admin. 

#### Lines affected:
https://github.com/nounsDAO/token-buyer/blob/23d64ac7093f504ad4731bc4cf8d41b2c2943657/src/TokenBuyer.sol#L388-L392

#### Remedial recommendation & POC:
Fix the pragma version for example `pragma solidity ^0.8.15;`
```solidity
function setAdmin(address newAdmin) external onlyAdminOrOwner {
	require(newAdmin != address(0), "Invalid address");
	
    emit AdminSet(admin, newAdmin);  
    
    admin = newAdmin;  
}
```


# Gas:
### [G-01] - WithdrawPaymentToken makes two calls to the Ownable contract

#### Description:
It is more gas efficient to use the `msg.sender` function in stead of checking the external ownable contract.

```solidity
address to = owner();
```

#### Impact:
From gas used: 
transaction cost	26687 gas 
execution cost	5623 gas 

To gas used:
transaction cost	26515 gas 
execution cost	5451 gas 

#### Lines affected:
https://github.com/nounsDAO/token-buyer/blob/23d64ac7093f504ad4731bc4cf8d41b2c2943657/src/Payer.sol#L123

#### Remedial action:

```diff
- address to = owner();
+ address to = msg.sender;
```


### [G-02] - Storage variable update in while loop
#### Description:
The `payBackDebt(uint256 amount)` function updates the `totalDebt` storage variable in a while loop. This could potentially see multiple mstore updates inside a single transaction. 

`totalDebt -= amount;`

#### Impact:
Increase gas usage due to multiple mstore updates.

#### Lines affected:
https://github.com/nounsDAO/token-buyer/blob/23d64ac7093f504ad4731bc4cf8d41b2c2943657/src/Payer.sol#L139-L184

#### Remedial recommendation & POC:
Consider caching `totalDebt` and updating the debt after the total debt has been calculated.

```solidity
    function payBackDebt(uint256 amount) external {

        uint256 public debtToPayBack;

        // While there are tokens left, and debt entries exist
        while (amount > 0 && !queue.empty()) {

                <...>

            if (amount < _debtAmount) {
            
                 <...>
                 
                debtToPayBack += amount;

                 <...>

                return;

            } else {

                <...>

                // Update total debt
                debtToPayBack += _debtAmount;
                
                <...>
            }

        }
		// minus total debt after loop
        totalDebt -= debtToPayBack;
    }
```
