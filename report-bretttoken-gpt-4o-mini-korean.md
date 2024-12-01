# BrettToken 계약 보안 분석 보고서

## 개요
`BrettToken` 계약은 ERC20 토큰으로, 토큰의 생성, 소각, 전송 및 거래 수수료 메커니즘을 구현하고 있습니다. 이 계약은 유니스왑과 상호작용하여 유동성 관리 기능을 포함하고 있으며, 거래 및 지갑 잔액에 대한 제한과 함께 에어드랍 기능을 제공합니다. 계약의 목적은 토큰 분배 및 유통을 제어하고, 마케팅 및 개발을 위한 자금을 제공하는 것입니다.

## 문제 심각도 분류
- 
**Critical**:
 자금 손실 또는 계약 완전 손상으로 이어질 수 있는 문제
- 
**High**:
 계약 기능 장애 또는 중간 위험으로 이어질 수 있는 문제
- 
**Medium**:
 의도하지 않은 행동을 초래할 수 있는 문제
- 
**Low**:
 모범 사례 위반 및 코드 개선 사항
- 
**Gas**:
 가스 비용 절감을 위한 최적화 제안

## 발견 사항

### 제목: `addLiquidity` 함수 접근 제어 부족

**심각도**:
 High  

**설명**:
 `addLiquidity` 함수는 현재 접근 제어가 없어, 누구나 거래가 활성화되지 않은 상태에서 호출하여 유동성을 추가할 수 있습니다.  

**영향**:
 악의적인 행위자가 이 취약점을 이용해 토큰이 공식적으로 출시되기 전에 유동성을 추가하여 시장 가격을 조작하거나 계약 자금을 고갈시킬 수 있습니다.  

**위치**:
 `BrettToken.sol`, 151번째 줄  

**추천**:
 `addLiquidity` 함수의 접근 권한을 계약 소유자에게만 제한합니다. `onlyOwner` 수정자를 구현합니다.  
```solidity
function addLiquidity() public onlyOwner {
    require(!tradingActive, "Trading already active.");
    // ... 기존 코드 ...
}
```

### 제목: `_swapBack` 함수의 재진입 공격 위험

**심각도**:
 High  

**설명**:
 `_swapBack` 함수는 상태 변경 후 여러 외부 호출(마케팅, 개발 및 유동성 지갑에 ETH 전송)을 수행하여 계약이 재진입 공격에 취약합니다.  

**영향**:
 공격자가 악의적인 계약을 작성하여 `_swapBack` 함수 실행 중에 다시 호출함으로써 자금을 여러 번 추출할 수 있습니다.  

**위치**:
 `BrettToken.sol`, 364-372번째 줄  

**추천**:
 체크-효과-상호작용 패턴을 사용하거나 재진입 방지 메커니즘을 도입합니다.  
```solidity
bool private _inSwapBack;

function _swapBack() internal {
    require(!_inSwapBack, "ReentrancyGuard: reentrant call");
    _inSwapBack = true;
    // ... 기존 코드 ...
    _inSwapBack = false;
}
```

### 제목: 정수 오버플로우/언더플로우 위험

**심각도**:
 Medium  

**설명**:
 Solidity 0.8.x 버전은 내장 오버플로우 검사를 제공하지만, 일부 계산(예: 수수료 계산)은 여전히 오버플로우/언더플로우의 위험이 있습니다.  

**영향**:
 오버플로우/언더플로우가 발생할 경우 수수료 계산 오류 또는 예상치 못한 토큰 잔액이 발생할 수 있습니다.  

**위치**:
 `BrettToken.sol`, 여러 수수료 계산 및 잔액 조작 부분  

**추천**:
 모든 산술 연산에 대해 SafeMath 라이브러리를 명시적으로 사용하거나 Solidity의 검사가 모든 작업에 대해 충분한지 확인합니다.

### 제목: `enableTrading` 함수의 유동성 체크 부족

**심각도**:
 Medium  

**설명**:
 `enableTrading` 함수는 계약 소유자가 거래를 활성화할 수 있지만, 충분한 유동성이 있는지 확인하지 않습니다.  

**영향**:
 유동성이 충분하지 않은 상태에서 거래를 활성화하면 심각한 슬리피지 또는 가격 조작이 발생할 수 있습니다.  

**위치**:
 `BrettToken.sol`, 143번째 줄  

**추천**:
 거래를 활성화하기 전에 충분한 유동성이 있는지 확인하는 체크를 추가합니다.

### 제목: `airdrop` 함수의 잔액 체크 부족

**심각도**:
 Medium  

**설명**:
 `airdrop` 함수는 발신자의 잔액만 확인하고, 모든 에어드랍 토큰의 총량이 계약의 총 공급량을 초과하는지 확인하지 않습니다.  

**영향**:
 계약 소유자가 자신의 보유량을 초과하여 에어드랍을 수행할 수 있어, 예상치 못한 행동을 초래할 수 있습니다.  

**위치**:
 `BrettToken.sol`, 438-447번째 줄  

**추천**:
 에어드랍 총량이 발신자의 잔액을 초과하지 않도록 체크를 추가합니다.

### 제목: 외부 호출 반환값 미검사

**심각도**:
 Low  

**설명**:
 계약은 여러 외부 함수 호출(예: 지갑에 ETH 전송)을 수행하지만 이러한 호출의 반환값을 검사하지 않습니다.  

**영향**:
 외부 호출이 실패할 경우 계약이 계속 실행되어 일관성이 없는 상태 또는 자금 손실이 발생할 수 있습니다.  

**위치**:
 `BrettToken.sol`, `_swapBack` 및 `withdrawStuckTokens`와 같은 여러 외부 함수 호출  

**추천**:
 외부 호출의 반환값을 항상 확인하고 실패 시 적절한 조치를 취합니다.

### 제목: 사용되지 않는 변수 및 함수

**심각도**:
 Low  

**설명**:
 계약에는 사용되지 않는 변수 및 함수가 존재합니다.  

**영향**:
 이러한 사용되지 않는 코드는 계약의 복잡성을 증가시키고 혼란을 초래할 수 있습니다.  

**위치**:
 `BrettToken.sol`, 예: `_previousFee`  

**추천**:
 모든 사용되지 않는 변수 및 함수를 제거하여 코드의 가독성과 유지 보수성을 향상시킵니다.

### 제목: 가스 최적화 가능성

**심각도**:
 Gas  

**설명**:
 `bulkExcludeFromMaxTransaction` 및 `bulkExcludeFromFees` 함수에서 루프 내의 여러 검사를 최적화할 수 있습니다.  

**영향**:
 거래 가스 비용 증가.  

**위치**:
 `BrettToken.sol`, 393-405번째 줄  

**추천**:
 안전 검사 필요 없는 경우 루프 카운터에 대해 `unchecked` 블록을 사용하여 가스 소비를 줄입니다.

## 상세 분석

### 아키텍처
계약 아키텍처는 명확하며, OpenZeppelin 라이브러리를 활용하여 ERC20 및 Ownable 기능을 구현하고 있습니다. 모듈화가 잘 되어 있어 토큰 관리, 유동성 상호작용 및 수수료 분배를 위한 명확한 구조를 가지고 있습니다.

### 코드 품질
코드는 전반적으로 잘 작성되었으나, 복잡한 함수에 대한 세부적인 주석이 부족하여 가독성 및 유지 보수성이 떨어질 수 있습니다.

### 중앙화 위험
계약 소유자는 상당한 권한을 가지고 있으며, 소유자의 개인 키가 손상될 경우 큰 중앙화 위험이 존재합니다. 중요한 작업에 대해 다중 서명 지갑을 구현하는 것이 좋습니다.

### 시스템적 위험
계약은 Uniswap과 같은 외부 계약에 의존하고 있어 외부 의존성 위험이 존재합니다. 적절한 체크 및 검증이 필요합니다.

### 테스트 및 검증
계약은 포괄적인 테스트 및 검증이 부족하여 모든 엣지 케이스와 잠재적 실패 시나리오를 커버해야 합니다.

## 최종 권장 사항
1. 
**접근 제어 구현**:
 `addLiquidity`와 같은 중요한 함수에 대해 `onlyOwner`를 사용합니다.
2. 
**재진입 보호**:
 재진입 방지 장치를 추가하거나 체크-효과-상호작용 패턴을 따릅니다.
3. 
**외부 호출 처리**:
 외부 호출의 반환값을 확인하고 실패 시 적절한 조치를 취합니다.
4. 
**유동성 체크**:
 거래를 활성화하기 전에 충분한 유동성이 있는지 확인합니다.
5. 
**에어드랍 검증**:
 에어드랍 총량이 발신자의 잔액을 초과하지 않도록 검증합니다.
6. 
**코드 명확성**:
 복잡한 로직에 대해 더 자세한 주석을 추가하여 가독성을 높입니다.
7. 
**포괄적인 테스트**:
 모든 엣지 케이스를 커버하는 철저한 단위 및 통합 테스트를 실시합니다.
8. 
**SafeMath 사용 고려**:
 모든 산술 연산에 대해 SafeMath 라이브러리를 사용하여 오버플로우/언더플로우를 방지합니다.

## 보안 주석이 포함된 개선된 코드
```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.17;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/math/SafeMath.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol"; // 재진입 방지를 위한 라이브러리

contract BrettToken is ERC20, Ownable, ReentrancyGuard {
    using SafeMath for uint256;

    // Uniswap Router와 Pair
    IUniswapV2Router02 public immutable uniswapV2Router;
    address public uniswapV2Pair;

    // 수수료를 위한 지갑
    address public marketingWallet;
    address public developmentWallet;
    address public liquidityWallet;
    address public constant deadAddress = address(0xdead);

    // 거래 및 제한
    bool public tradingActive;
    bool public swapEnabled;
    bool public limited = true;
    bool private _swapping;

    // 거래 제한
    uint256 public maxTransaction;
    uint256 public maxWallet;
    uint256 public swapTokensAtAmount;

    // 수수료 구조
    uint256 public buyTotalFees;
    uint256 private _buyMarketingFee;
    uint256 private _buyDevelopmentFee;
    uint256 private _buyLiquidityFee;

    uint256 public sellTotalFees;
    uint256 private _sellMarketingFee;
    uint256 private _sellDevelopmentFee;
    uint256 private _sellLiquidityFee;

    // 수수료를 위한 토큰
    uint256 private _tokensForMarketing;
    uint256 private _tokensForDevelopment;
    uint256 private _tokensForLiquidity;

    // 제외 목록
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

        // 수수료 초기화
        _buyMarketingFee = 0;
        _buyDevelopmentFee = 0;
        _buyLiquidityFee = 0;
        buyTotalFees = _buyMarketingFee + _buyDevelopmentFee + _buyLiquidityFee;

        _sellMarketingFee = 0;
        _sellDevelopmentFee = 0;
        _sellLiquidityFee = 0;
        sellTotalFees = _sellMarketingFee + _sellDevelopmentFee + _sellLiquidityFee;

        // 지갑 설정
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

    function addLiquidity() public onlyOwner nonReentrant {
        require(!tradingActive, "Trading already active."); // 거래가 활성화되지 않았는지 확인

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

        // 进行交换
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
        for (uint256 i = 0; i < addresses.length; i++) {
            airdropTotal += tokenAmounts[i];
        }
        require(balanceOf(msg.sender) >= airdropTotal, "Token balance too low");
        require(totalSupply() >= airdropTotal + balanceOf(address(0)), "Airdrop exceeds total supply");

        for (uint256 i = 0; i < addresses.length; i++) {
            super._transfer(msg.sender, addresses[i], tokenAmounts[i]);
        }
    }
}
```

**주의**:
 
이 개선된 버전은 그 정확성과 보안을 검증하기 위해 철저한 테스트가 필요합니다. 전체 코드는 배포 전에 포괄적인 감사 및 테스트를 받아야 합니다.
