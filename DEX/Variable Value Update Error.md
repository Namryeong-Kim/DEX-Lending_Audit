# Problem - Variable Value Update Error
## 권준우

### 설명

문제 발생 - 전체

```solidity
mapping(address => uint256) public LPToken_balances;
mapping(address => uint256) public balances;

function addLiquidity(uint256 tokenXAmount, uint256 tokenYAmount, uint256 minimumLPTokenAmount) external returns (uint256 LPTokenAmount){
								    ...
    uint256 xBalance = balances[address(tokenX)];
    uint256 yBalance = balances[address(tokenY)];
										...
}

function swap(uint256 tokenXAmount, uint256 tokenYAmount, uint256 tokenMinimumOutputAmount) public returns (uint256 outputAmount){
						        ...
    uint256 xBalance = tokenX.balanceOf(address(this));
    uint256 yBalance = tokenY.balanceOf(address(this)); 
    
    if(tokenXAmount != 0){
										...
        yBalance -= outputAmount ;
        xBalance += tokenXAmount;
						        ...
    }
    else{
						        ...
        xBalance -= outputAmount;
        yBalance += tokenYAmount;
						        ...
    }
    return outputAmount;
}

function removeLiquidity(uint256 LPTokenAmount, uint256 minimumTokenXAmount, uint256 minimumTokenYAmount) external returns (uint,uint){
								    ...
    uint xBalance = (LPTokenAmount*tokenX.balanceOf(address(this)))/totalLiquidity;
    uint yBalance = (LPTokenAmount*tokenY.balanceOf(address(this)))/totalLiquidity;
								    ...
    balances[address(tokenX)] -= xBalance;
    balances[address(tokenY)] -= yBalance;
    
    LPToken_balances[msg.sender] -= LPTokenAmount;
    totalLiquidity -= LPTokenAmount;
    return (xBalance,yBalance);
}
```

- addLiquidity로 유동성 제공 후 swap 시 balances[address(tokenX)], balances[address(tokenY)]를 업데이트 하지 않음
    - swap 후 addLiquidity 수행시, swap으로 변동된 pool liquidity의 비율이 아닌 이전 비율로 계산이 됨
- swap후 removeLiquidity에서 LPTokenAmount에 대한 xBalance, yBalance를 뺄 수 없음 → DEX가 망가짐

테스트

```solidity
function testSwap2() external {
  dex.addLiquidity(3000 ether, 3000 ether, 0);
  dex.swap(1000 ether, 0, 0);
  dex.removeLiquidity(dex.LPToken_balances(address(this)),0,0);
}
```

<img width="708" alt="스크린샷 2023-03-26 오후 10 47 42" src="https://user-images.githubusercontent.com/127647051/227780117-cc3d6e3a-b794-4847-a3c3-4f81777849ac.png">

- 시나리오
    - addLiquidity로 x, y token에 각각 3000 ether씩 유동성을 제공함
        
        → 3000 ether의 LP token 받음
        
    - x token에 대해 1000 ether swap 수행
        
        → balances[address(tokenX)], balances[address(tokenY)]가 4000 ether, 2250 ether로 업데이트 되어야 하나 업데이트 되지 않음(여전히 balances[address(tokenX)], balances[address(tokenY)]는 각각 3000 ether임)
        
    - 3000 ether의 LP token에 대해 모두 removeLiquidity 수행
        
        → removeLiquidity에서 가져온 xBalance, yBalance는 
        
        ```solidity
        uint xBalance = (LPTokenAmount*tokenX.balanceOf(address(this)))/totalLiquidity;
        uint yBalance = (LPTokenAmount*tokenY.balanceOf(address(this)))/totalLiquidity;
        ```
        
        로부터 가져오므로 4000 ether, 2250 ether가 됨
        
        그러나 balances[address(tokenX)], balances[address(tokenY)]는 3000 ether로 업데이트 되지 않은 상태이기 때문에 Arithmetic over/underflow가 발생함
        

### 파급력

- Critical → 해당 DEX에서 addLiquidity후 swap을 수행하게 되면 removeLiquidity를 정상적으로 수행할 수 없음
    
    공격자가 swap으로 pool liquidity 비율을 깨면 DEX를 망가뜨릴 수 있음
    

### 해결방안

- 함수에서 사용되는 xBalance, yBalance 모두 tokenX.balanceOf(address(this)), tokenY.balanceOf(address(this))로 업데이트하기
