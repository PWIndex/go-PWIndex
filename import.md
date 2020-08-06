Hackers use the same scheme to attack Opyn contracts to attack the **PW INDEX** ecosystem

```
function exercise(
        uint256 oTokensToExercise,
        address payable[] memory vaultsToExerciseFrom
) public payable {
        for (uint256 i = 0; i < vaultsToExerciseFrom.length; i++) {
            address payable vaultOwner = vaultsToExerciseFrom[i];
            require(
                hasVault(vaultOwner),
                "Cannot exercise from a vault that doesn't exist"
            );
            Vault storage vault = vaults[vaultOwner];
            if (oTokensToExercise == 0) {
                return;
            } else if (vault.oTokensIssued >= oTokensToExercise) {
                _exercise(oTokensToExercise, vaultOwner);
                return;
            } else {
                oTokensToExercise = oTokensToExercise.sub(vault.oTokensIssued);
                _exercise(vault.oTokensIssued, vaultOwner);
            }
        }
        require(
            oTokensToExercise == 0,
            "Specified vaults have insufficient collateral"
        );
    }
```

The attacker only used 272ETH and finally got 467ETH

The key point is the exercise function of the contract. From the figure above, we can see that the USDC is sent to the attacker contract by calling transfer twice in the exercise function. Next, we will switch to the exercise function for specific analysis.



You can see that the exercise function allows multiple vaultsToExerciseFrom to be passed in, and then calls the _exercise function through the for loop to process each vaultsToExerciseFrom. Now we cut into the _exercise function for specific analysis

```
function _exercise(
        uint256 oTokensToExercise,
        address payable vaultToExerciseFrom
) internal {
        // 1. before exercise window: revert
        require(
            isExerciseWindow(),
            "Can't exercise outside of the exercise window"
        );

        require(hasVault(vaultToExerciseFrom), "Vault does not exist");

        Vault storage vault = vaults[vaultToExerciseFrom];
        require(oTokensToExercise > 0, "Can't exercise 0 oTokens");
        // Check correct amount of oTokens passed in)
        require(
            oTokensToExercise <= vault.oTokensIssued,
            "Can't exercise more oTokens than the owner has"
        );
        // Ensure person calling has enough oTokens
        require(
            balanceOf(msg.sender) >= oTokensToExercise,
            "Not enough oTokens"
        );

        // 1. Check sufficient underlying
        // 1.1 update underlying balances
        uint256 amtUnderlyingToPay = underlyingRequiredToExercise(
            oTokensToExercise
        );
        vault.underlying = vault.underlying.add(amtUnderlyingToPay);

        // 2. Calculate Collateral to pay
        // 2.1 Payout enough collateral to get (strikePrice * oTokens) amount of collateral
        uint256 amtCollateralToPay = calculateCollateralToPay(
            oTokensToExercise,
            Number(1, 0)
        );

        // 2.2 Take a small fee on every exercise
        uint256 amtFee = calculateCollateralToPay(
            oTokensToExercise,
            transactionFee
        );
        totalFee = totalFee.add(amtFee);

        uint256 totalCollateralToPay = amtCollateralToPay.add(amtFee);
        require(
            totalCollateralToPay <= vault.collateral,
            "Vault underwater, can't exercise"
        );

        // 3. Update collateral + oToken balances
        vault.collateral = vault.collateral.sub(totalCollateralToPay);
        vault.oTokensIssued = vault.oTokensIssued.sub(oTokensToExercise);

        // 4. Transfer in underlying, burn oTokens + pay out collateral
        // 4.1 Transfer in underlying
        if (isETH(underlying)) {
            require(msg.value == amtUnderlyingToPay, "Incorrect msg.value");
        } else {
            require(
                underlying.transferFrom(
                    msg.sender,
                    address(this),
                    amtUnderlyingToPay
                ),
                "Could not transfer in tokens"
            );
        }
        // 4.2 burn oTokens
        _burn(msg.sender, oTokensToExercise);

        // 4.3 Pay out collateral
        transferCollateral(msg.sender, amtCollateralToPay);

        emit Exercise(
            amtUnderlyingToPay,
            amtCollateralToPay,
            msg.sender,
            vaultToExerciseFrom
        );

    }
```

1. In line 6 of the code, first check whether it is within the insurance period. This is naturally affirmative
2. In line 11 of the code, check whether vaultToExerciseFrom has created a vault. Note that this is only to check whether a vault has been created.
3. Check the incoming oTokensToExercise value in lines 14, 16, and 21 of the code, and you can see that the attacker passed in 0x1443fd000, which is obviously passable.
4. Next, calculate the amount of ETH that needs to be consumed on line 28 of the code
5. Calculate the amount and handling fee to be paid in lines 35 and 41 of the code
6. Next, determine whether the underlying is an ETH address in line 59 of the code. Underlying is assigned in line 31 of the above code. Since isETH is true, it will enter the if logic instead of the else logic. In if In logic, both amtUnderlyingToPay and msg.value are user-controllable
7. The oTokensToExercise is then burned, and the transferCollateral function is called to transfer USDC to the caller of the exercise function
The above key points are steps 2 and 6, so we only need to make sure that the incoming vaultToExerciseFrom has created a vault, and make amtUnderlyingToPay equal to msg.value, and these related parameters are all we can control, so the attack idea It's obvious.

8. The vaultToExerciseFrom passed by the attacker are:
0xe7870231992ab4b1a01814fa0a599115fe94203f
0x076c95c6cd2eb823acc6347fdf5b3dd9b83511e4


It has been verified that both addresses have created a vault

10. At this time, since the oToken created by the attacker is 0xa21fe800 (2720000000), and vault.oTokensIssued is 2720000000 less than 5440000000, so the else logic in the exercise function will be used. At this time, oTokensToExercise is 0xa21fe800 (2720000000), then the above code is 60th The line msg.value == amtUnderlyingToPay is definitely established

11. Since vaultsToExerciseFrom passes in two addresses, the for loop will execute the _exercise function twice, so transfer USDC to the attacker contract twice.


**The attack process is as follows**

1. The attacker uses the contract to first call the contract's createERC20CollateralOption function to create oToken

2. The attack contract calls the exercise function, passing in the address of the created vault

3. Call the _exercise function twice through the for loop logic execution in the exercise function

4. The exercise function calls the transferCollateral function to transfer USDC to the function caller (because the for loop calls the _exercise function twice, the transferCollateral function will also be executed twice)

5. The attack contract calls the removeUnderling function to transfer out the previously passed ETH

6. In the end, the attacker got back the previously invested ETH and additional USDC


The PW INDEX open source community has offered a reward to remedy the contract loopholes, and will conduct a two-week period of vulnerability repair data to improve. All communities are requested to submit open source work.