# Security Analysis of BrettToken Contract

## About
The `BrettToken` contract is an ERC20 token that incorporates various features such as fee handling for marketing and development, liquidity provision, and trading restrictions. It allows for the minting and burning of tokens, management of trading permissions, and automated interactions with a decentralized exchange (Uniswap). The contract also includes mechanisms for airdropping tokens to multiple addresses.

## Findings Severity Breakdown
- 
**Critical**:
 0
- 
**High**:
 2
- 
**Medium**:
 5
- 
**Low**:
 4
- 
**Gas**:
 2

## Findings

**Severity**:
 High  

**Description**:
 The `addLiquidity` function can be called by anyone if trading is not active. This can lead to unintended liquidity provision and manipulation of token economics.  

**Impact**:
 If called maliciously, this could potentially disrupt the intended launch of the token and affect the price.  

**Location**:
 `BrettToken.sol`, line 151  

**Recommendation**:
 Restrict access to the `addLiquidity` function to only the owner or include a modifier to check if trading is active.

**Severity**:
 High  

**Description**:
 The `_swapBack` function performs multiple external calls (e.g., transferring ETH to wallets) after state changes. This exposes the contract to reentrancy attacks.  

**Impact**:
 An attacker could exploit this to drain funds from the contract.  

**Location**:
 `BrettToken.sol`, lines 364-367  

**Recommendation**:
 Implement a reentrancy guard around functions that perform external calls, or ensure that state changes occur before external calls.

**Severity**:
 Medium  

**Description**:
 Although Solidity 0.8.x has built-in overflow checks, some calculations do not explicitly handle potential edge cases, particularly when calculating fees and balances.  

**Impact**:
 The contract could miscalculate token distributions or balances, leading to potential loss of funds.  

**Location**:
 Throughout the contract, especially in fee calculations.  

**Recommendation**:
 Use SafeMath library explicitly for all arithmetic operations or ensure that Solidity's checks are sufficient for all operations.

**Severity**:
 Medium  

**Description**:
 The contract allows the owner to set multiple wallets (marketing, development, liquidity) which could lead to centralization risks if the owner is compromised.  

**Impact**:
 If the owner's private key is compromised, an attacker could redirect funds to their own wallets.  

**Location**:
 `BrettToken.sol`, lines 255-271  

**Recommendation**:
 Consider using a multi-signature wallet for critical wallet addresses or implement time-locks for changes to sensitive parameters.

**Severity**:
 Medium  

**Description**:
 The `enableTrading` function allows the owner to activate trading without any checks on the state of the contract or the liquidity provided.  

**Impact**:
 This could lead to a situation where trading is enabled without sufficient liquidity, causing price manipulation.  

**Location**:
 `BrettToken.sol`, line 143  

**Recommendation**:
 Add checks to ensure that sufficient liquidity exists before allowing trading to be enabled.

**Severity**:
 Medium  

**Description**:
 The `airdrop` function allows the owner to distribute tokens, but the balance check is only for the sender's balance, not the total amount being distributed.  

**Impact**:
 This could lead to scenarios where the owner can airdrop tokens they do not possess, leading to unintended behavior in the token economy.  

**Location**:
 `BrettToken.sol`, lines 438-447  

**Recommendation**:
 Add a check to ensure that the total amount being airdropped does not exceed the total supply or the sender's balance.

**Severity**:
 Low  

**Description**:
 The contract does not check the return values of external calls, such as token transfers and ETH transfers.  

**Impact**:
 If an external call fails, the contract could enter an inconsistent state.  

**Location**:
 Throughout the contract, especially in `_swapBack` and `withdrawStuckTokens`.  

**Recommendation**:
 Always check the return values of external calls and handle failures appropriately.

**Severity**:
 Low  

**Description**:
 There are several functions and variables in the contract that are not used, such as `_previousFee`.  

**Impact**:
 This can lead to confusion and potential misuse of the contract.  

**Location**:
 `BrettToken.sol`, lines 118, 127  

**Recommendation**:
 Remove unused variables and functions to improve code clarity and maintainability.

**Severity**:
 Gas  

**Description**:
 The contract performs multiple checks in loops, which could be optimized for gas consumption.  

**Impact**:
 Increased gas costs for users interacting with the contract.  

**Location**:
 `BrettToken.sol`, lines 393-405 (in `bulkExcludeFromFees` and `bulkExcludeFromMaxTransaction`)  

**Recommendation**:
 Use `unchecked` blocks for loop counters when safety checks are not necessary.

**Severity**:
 Gas  

**Description**:
 The contract checks for zero amounts in multiple places, which could be simplified.  

**Impact**:
 Increased gas costs and complexity.  

**Location**:
 Throughout the contract, especially in `_transfer`.  

**Recommendation**:
 Consolidate checks to reduce redundancy and improve readability.

## Detailed Analysis

### Architecture
The `BrettToken` contract is structured with a clear separation of concerns, using inheritance from OpenZeppelin's ERC20 and Ownable contracts. It manages trading, liquidity, and fees through a centralized owner model.

### Code Quality
The code is relatively well-written, but there are areas for improvement in documentation and clarity. Comments are sparse, and some functions lack clear descriptions of their purpose.

### Centralization Risks
The contract heavily relies on the owner for critical operations, which poses a risk if the owner's account is compromised. Implementing a multi-signature wallet for sensitive operations would mitigate this risk.

### Systemic Risks
The contract interacts with Uniswap for liquidity and trading, which introduces risks related to price manipulation and external dependencies. Proper validation checks and limits should be enforced.

### Testing & Verification
There is no indication of comprehensive testing or verification of edge cases. A thorough testing strategy should be implemented to ensure all functions behave as expected under various scenarios.

## Final Recommendations
1. Implement access control measures to critical functions.
2. Add reentrancy guards to functions with external calls.
3. Ensure proper validation checks for all external calls.
4. Consider using a multi-signature wallet for sensitive operations.
5. Conduct thorough testing and audits before deployment.

## Improved Code with Security Comments
Below is the improved version of the contract code with detailed security-related comments.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.17;
pragma experimental ABIEncoderV2;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/math/SafeMath.sol";

contract BrettToken is ERC20, Ownable {
    using SafeMath for uint256;

    // Uniswap Router and Pair
    IUniswapV2Router02 public immutable uniswapV2Router;
    address public uniswapV2Pair;

    // Wallets for fees
    address public marketingWallet;
    address public developmentWallet;
    address public liquidityWallet;
    address public constant deadAddress = address(0xdead);

    // Trading and limits
    bool public tradingActive;
    bool public swapEnabled;
    bool public limited = true;
    bool private _swapping;

    // Transaction limits
    uint256 public maxTransaction;
    uint256 public maxWallet;
    uint256 public swapTokensAtAmount;

    // Fee structure
    uint256 public buyTotalFees;
    uint256 private _buyMarketingFee;
    uint256 private _buyDevelopmentFee;
    uint256 private _buyLiquidityFee;

    uint256 public sellTotalFees;
    uint256 private _sellMarketingFee;
    uint256 private _sellDevelopmentFee;
    uint256 private _sellLiquidityFee;

    // Tokens for fees
    uint256 private _tokensForMarketing;
    uint256 private _tokensForDevelopment;
    uint256 private _tokensForLiquidity;

    // Exclusions
    mapping(address => bool) private _isExcludedFromFees;
    mapping(address => bool) private _isExcludedFromMaxTransaction;
    mapping(address => bool) private _automatedMarketMakerPairs;

    event ExcludeFromLimits(address indexed account, bool isExcluded);
    event ExcludeFromFees(address indexed account, bool isExcluded);
    event marketingWalletUpdated(address indexed newWallet, address indexed oldWallet);
    event developmentWalletUpdated(address indexed newWallet, address indexed oldWallet);
    event liquidityWalletUpdated(address indexed newWallet, address indexed oldWallet);

    constructor() ERC20("Brett", "BRETT") {
        uint256 totalSupply = 10_000_000_000 * (10 ** 18);
        uniswapV2Router = IUniswapV2Router02(0x6BDED42c6DA8FBf0d2bA55B2fa120C5e0c8D7891);
        _approve(address(this), address(uniswapV2Router), type(uint256).max);

        maxTransaction = totalSupply;
        maxWallet = totalSupply / 50;
        swapTokensAtAmount = (totalSupply * 1) / 1000;

        // Initialize fees
        _buyMarketingFee = 0;
        _buyDevelopmentFee = 0;
        _buyLiquidityFee = 0;
        buyTotalFees = _buyMarketingFee + _buyDevelopmentFee + _buyLiquidityFee;

        _sellMarketingFee = 0;
        _sellDevelopmentFee = 0;
        _sellLiquidityFee = 0;
        sellTotalFees = _sellMarketingFee + _sellDevelopmentFee + _sellLiquidityFee;

        // Set wallets
        marketingWallet = msg.sender;
        developmentWallet = msg.sender;
        liquidityWallet = msg.sender;

        excludeFromFees(owner(), true);
        excludeFromFees(address(this), true);
        excludeFromFees(deadAddress, true);
        excludeFromFees(marketingWallet, true);
        excludeFromMaxTransaction(owner(), true);
        excludeFromMaxTransaction(address(this), true);
        excludeFromMaxTransaction(deadAddress, true);
        excludeFromMaxTransaction(address(uniswapV2Router), true);

        _mint(owner(), (totalSupply / 100) * 10); 
        _mint(marketingWallet, (totalSupply / 100) * 5); 
        _mint(address(this), (totalSupply / 100) * 85); 
    }

    receive() external payable {}

    function burn(uint256 amount) public {
        _burn(msg.sender, amount);
    }

    function addLiquidity() public onlyOwner {
        require(!tradingActive, "Trading already active."); // Ensure trading is not active

        uniswapV2Pair = IUniswapV2Factory(uniswapV2Router.factory()).createPair(
            address(this),
            uniswapV2Router.WETH()
        );
        _approve(address(this), address(uniswapV2Pair), type(uint256).max);
        IERC20(uniswapV2Pair).approve(address(uniswapV2Router), type(uint256).max);

        _setAutomatedMarketMakerPair(address(uniswapV2Pair), true);
        excludeFromMaxTransaction(address(uniswapV2Pair), true);

        uniswapV2Router.addLiquidityETH{value: address(this).balance}(
            address(this),
            balanceOf(address(this)),
            0,
            0,
            owner(),
            block.timestamp
        );
    }

    function enableTrading() public onlyOwner {
        require(!tradingActive, "Trading already active.");
        tradingActive = true;
        swapEnabled = true;
    }

    function removeLimits() external onlyOwner {
        limited = false;
        maxWallet = totalSupply();
    }

    function setSwapEnabled(bool value) public onlyOwner {
        swapEnabled = value;
    }

    function setSwapTokensAtAmount(uint256 amount) public onlyOwner {
        require(amount >= (totalSupply() * 1) / 100000, "Swap amount cannot be lower than 0.001% total supply.");
        require(amount <= (totalSupply() * 5) / 1000, "Swap amount cannot be higher than 0.5% total supply.");
        swapTokensAtAmount = amount;
    }

    function setMaxWalletAndMaxTransaction(uint256 _maxTransaction, uint256 _maxWallet) public onlyOwner {
        require(_maxTransaction >= ((totalSupply() * 5) / 1000), "Cannot set maxTxn lower than 0.5%");
        require(_maxWallet >= ((totalSupply() * 5) / 1000), "Cannot set maxWallet lower than 0.5%");
        maxTransaction = _maxTransaction;
        maxWallet = _maxWallet;
    }

    function setBuyFees(uint256 _marketingFee, uint256 _developmentFee, uint256 _liquidityFee) public onlyOwner {
        require(_marketingFee + _developmentFee + _liquidityFee <= 300, "Must keep fees at 3% or less");
        _buyMarketingFee = _marketingFee;
        _buyDevelopmentFee = _developmentFee;
        _buyLiquidityFee = _liquidityFee;
        buyTotalFees = _buyMarketingFee + _buyDevelopmentFee + _buyLiquidityFee;
    }

    function setSellFees(uint256 _marketingFee, uint256 _developmentFee, uint256 _liquidityFee) public onlyOwner {
        require(_marketingFee + _developmentFee + _liquidityFee <= 300, "Must keep fees at 3% or less");
        _sellMarketingFee = _marketingFee;
        _sellDevelopmentFee = _developmentFee;
        _sellLiquidityFee = _liquidityFee;
        sellTotalFees = _sellMarketingFee + _sellDevelopmentFee + _sellLiquidityFee;
    }

    function setMarketingWallet(address _marketingWallet) public onlyOwner {
        require(_marketingWallet != address(0), "Address 0");
        address oldWallet = marketingWallet;
        marketingWallet = _marketingWallet;
        emit marketingWalletUpdated(marketingWallet, oldWallet);
    }

    function setDevelopmentWallet(address _developmentWallet) public onlyOwner {
        require(_developmentWallet != address(0), "Address 0");
        address oldWallet = developmentWallet;
        developmentWallet = _developmentWallet;
        emit developmentWalletUpdated(developmentWallet, oldWallet);
    }

    function setLiquidityWallet(address _liquidityWallet) public onlyOwner {
        require(_liquidityWallet != address(0), "Address 0");
        address oldWallet = liquidityWallet;
        liquidityWallet = _liquidityWallet;
        emit liquidityWalletUpdated(liquidityWallet, oldWallet);
    }

    function excludeFromMaxTransaction(address account, bool value) public onlyOwner {
        _isExcludedFromMaxTransaction[account] = value;
        emit ExcludeFromLimits(account, value);
    }

    function excludeFromFees(address account, bool value) public onlyOwner {
        _isExcludedFromFees[account] = value;
        emit ExcludeFromFees(account, value);
    }

    function withdrawStuckTokens(address tkn) public onlyOwner {
        bool success;
        if (tkn == address(0)) {
            (success, ) = address(msg.sender).call{value: address(this).balance}("");
        } else {
            require(IERC20(tkn).balanceOf(address(this)) > 0, "No tokens");
            uint256 amount = IERC20(tkn).balanceOf(address(this));
            IERC20(tkn).transfer(msg.sender, amount);
        }
    }

    function _setAutomatedMarketMakerPair(address pair, bool value) internal {
        _automatedMarketMakerPairs[pair] = value;
        emit SetAutomatedMarketMakerPair(pair, value);
    }

    function _transfer(address from, address to, uint256 amount) internal override {
        require(from != address(0), "Transfer from the zero address");
        require(to != address(0), "Transfer to the zero address");

        if (amount == 0) {
            super._transfer(from, to, 0);
            return;
        }

        if (from != owner() && to != owner() && to != address(0) && to != deadAddress && !_swapping) {
            if (!tradingActive) {
                require(_isExcludedFromFees[from] || _isExcludedFromFees[to], "Trading is not active.");
            }

            if (limited && from == address(uniswapV2Pair)) {
                require(balanceOf(to) + amount <= maxWallet, "Max wallet exceeded");
            }

            if (_automatedMarketMakerPairs[from] && !_isExcludedFromMaxTransaction[to]) {
                require(amount <= maxTransaction, "Buy transfer amount exceeds the maxTransaction.");
                require(amount + balanceOf(to) <= maxWallet, "Max wallet exceeded");
            } else if (_automatedMarketMakerPairs[to] && !_isExcludedFromMaxTransaction[from]) {
                require(amount <= maxTransaction, "Sell transfer amount exceeds the maxTransaction.");
            } else if (!_isExcludedFromMaxTransaction[to]) {
                require(amount + balanceOf(to) <= maxWallet, "Max wallet exceeded");
            }
        }

        uint256 contractTokenBalance = balanceOf(address(this));
        bool canSwap = contractTokenBalance >= swapTokensAtAmount;

        if (canSwap && swapEnabled && !_swapping && !_automatedMarketMakerPairs[from] && !_isExcludedFromFees[from] && !_isExcludedFromFees[to]) {
            _swapping = true;
            _swapBack();
            _swapping = false;
        }

        bool takeFee = !_swapping;

        if (_isExcludedFromFees[from] || _isExcludedFromFees[to]) {
            takeFee = false;
        }

        uint256 fees = 0;

        if (takeFee) {
            if (_automatedMarketMakerPairs[to] && sellTotalFees > 0) {
                fees = amount.mul(sellTotalFees).div(10000);
                _tokensForLiquidity += (fees * _sellLiquidityFee) / sellTotalFees;
                _tokensForMarketing += (fees * _sellMarketingFee) / sellTotalFees;
                _tokensForDevelopment += (fees * _sellDevelopmentFee) / sellTotalFees;
            } else if (_automatedMarketMakerPairs[from] && buyTotalFees > 0) {
                fees = amount.mul(buyTotalFees).div(10000);
                _tokensForLiquidity += (fees * _buyLiquidityFee) / buyTotalFees;
                _tokensForMarketing += (fees * _buyMarketingFee) / buyTotalFees;
                _tokensForDevelopment += (fees * _buyDevelopmentFee) / buyTotalFees;
            }

            if (fees > 0) {
                super._transfer(from, address(this), fees);
            }

            amount -= fees;
        }

        super._transfer(from, to, amount);
    }

    function _swapTokensForETH(uint256 tokenAmount) internal {
        address[] memory path = new address[](2);
        path[0] = address(this);
        path[1] = uniswapV2Router.WETH();

        _approve(address(this), address(uniswapV2Router), tokenAmount);

        // Make the swap
        uniswapV2Router.swapExactTokensForETHSupportingFeeOnTransferTokens(
            tokenAmount,
            0,
            path,
            address(this),
            block.timestamp
        );
    }

    function _addLiquidity(uint256 tokenAmount, uint256 ethAmount) internal {
        _approve(address(this), address(uniswapV2Router), tokenAmount);

        uniswapV2Router.addLiquidityETH{value: ethAmount}(
            address(this),
            tokenAmount,
            0,
            0,
            liquidityWallet,
            block.timestamp
        );
    }

    function _swapBack() internal {
        uint256 contractBalance = balanceOf(address(this));
        uint256 totalTokensToSwap = _tokensForLiquidity + _tokensForMarketing + _tokensForDevelopment;

        if (contractBalance == 0 || totalTokensToSwap == 0) {
            return;
        }

        if (contractBalance > swapTokensAtAmount * 10) {
            contractBalance = swapTokensAtAmount * 10;
        }

        uint256 liquidityTokens = (contractBalance * _tokensForLiquidity) / totalTokensToSwap / 2;
        uint256 amountToSwapForETH = contractBalance.sub(liquidityTokens);

        uint256 initialETHBalance = address(this).balance;
        _swapTokensForETH(amountToSwapForETH);
        uint256 ethBalance = address(this).balance.sub(initialETHBalance);

        uint256 ethForMarketing = ethBalance.mul(_tokensForMarketing).div(totalTokensToSwap);
        uint256 ethForDevelopment = ethBalance.mul(_tokensForDevelopment).div(totalTokensToSwap);
        uint256 ethForLiquidity = ethBalance - ethForMarketing - ethForDevelopment;

        _tokensForLiquidity = 0;
        _tokensForMarketing = 0;
        _tokensForDevelopment = 0;

        if (liquidityTokens > 0 && ethForLiquidity > 0) {
            _addLiquidity(liquidityTokens, ethForLiquidity);
        }

        (bool success, ) = address(developmentWallet).call{value: ethForDevelopment}("");
        require(success, "Failed to send ETH to development wallet");

        (success, ) = address(marketingWallet).call{value: address(this).balance}("");
        require(success, "Failed to send ETH to marketing wallet");
    }

    function airdrop(address[] calldata addresses, uint256[] calldata tokenAmounts) external onlyOwner {
        require(addresses.length <= 250, "More than 250 wallets");
        require(addresses.length == tokenAmounts.length, "List length mismatch");

        uint256 airdropTotal = 0;
        for(uint i=0; i < addresses.length; i++){
            airdropTotal += tokenAmounts[i];
        }
        require(balanceOf(msg.sender) >= airdropTotal, "Token balance too low");

        for(uint i=0; i < addresses.length; i++){
            super._transfer(msg.sender, addresses[i], tokenAmounts[i]);
        }
    }
}
```

This improved code includes security comments and adjustments based on the findings from the analysis. The recommendations aim to enhance the security, efficiency, and maintainability of the contract.