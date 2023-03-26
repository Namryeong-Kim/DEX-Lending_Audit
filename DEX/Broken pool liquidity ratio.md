#Problem - Broken pool liquidity ratio
## 최영현

### 설명

문제 발생 - addLiquidity function

```solidity
function addLiquidity(uint256 tokenXAmount, uint256 tokenYAmount, uint256 minimumLPTokenAmount) external returns (uint LPTokenAmount){
        require(tokenXAmount > 0 && tokenYAmount > 0);
        (uint256 reserveX, ) = amount_update();
        (, uint256 reserveY) = amount_update();

        if(totalSupply() == 0){ LPTokenAmount = tokenXAmount * tokenYAmount / 10**18;}
        
        else{ LPTokenAmount = totalSupply() * tokenXAmount / reserveX;}
        //console.log("add total:", totalSupply());
        require(minimumLPTokenAmount <= LPTokenAmount);
```

addLiquidity에서 imbalance한 token amount를 제공하더라도 그대로 pool에 add하여 LP token을 발행함

```solidity
function testAddLiquidity() external {
        uint firstLPReturn = dex.addLiquidity(1000 ether, 1000 ether, 0); //lp = 1000
        emit log_named_uint("firstLPReturn", firstLPReturn);

				uint secondLPReturn = dex.addLiquidity(4000 ether, 100 ether, 0);
        emit log_named_uint("secondLPReturn", secondLPReturn);
        
				(bool success, ) = address(dex).call(abi.encodeWithSelector(dex.addLiquidity.selector, 4000 ether, 100 ether, 0));
        assertTrue(!success, "error");
}
```
<img width="544" alt="스크린샷 2023-03-26 오후 8 40 54" src="https://user-images.githubusercontent.com/127647051/227773283-017aff86-a48f-4e61-a8b9-929e324cbf14.png">

→ second addLiquidity에서 LP token 발행시, 무조건 X토큰을 기준으로 발행함. Y토큰에 적은양을 amount로 주더라도 X토큰에 대한 LP token을 받을 수 있으며, 해당 값이 totalSupply로 mint됨

→ removeLiquidity시, Y토큰에 대해 더 많은 amount를 획득할 수 있게됨

```solidity
function testAddLiquidity() external {
        uint firstLPReturn = dex.addLiquidity(1000 ether, 1000 ether, 0); //lp = 1000
        emit log_named_uint("firstLPReturn", firstLPReturn);

        uint secondLPReturn = dex.addLiquidity(4000 ether, 100 ether, 0);
        emit log_named_uint("secondLPReturn", secondLPReturn);

        (uint x, uint y) = dex.removeLiquidity(secondLPReturn,0, 0); //lp = 1000
        emit log_named_uint("x", x);
        emit log_named_uint("y", y);

        assertEq(x, 4000 ether);
        assertEq(y, 100 ether);
    }
```
<img width="546" alt="스크린샷 2023-03-26 오후 8 42 03" src="https://user-images.githubusercontent.com/127647051/227773319-50f7f597-119e-4257-bd2c-bf45ddc8dea0.png">

### 파급력

- Critical → 공격자는 적은 양의 token을 ‎pool에 제공하여 본인이 제공한 것보다 많은 amount token을 회수할 수 있음

### 해결방안

- addLiquidity에 비율에 따른 token amount를 add할 수 있도록 추가하기

---

## 황준태

### 설명

문제 발생 - addLiquidity function

```solidity
if(totalSupply_ ==0 ){ //if first supply 
        LPTokenAmount = _sqrt(tokenXAmount*tokenYAmount);
    }
    else{// calculate over the before
        //pool 비율 깨짐
        liqX = _mul(tokenXAmount ,totalSupply_)/amountX;
        liqY = _mul(tokenYAmount ,totalSupply_)/amountY;
        LPTokenAmount = (liqX<liqY) ? liqX:liqY; 
    }
    require(LPTokenAmount >= minimumLPTokenAmount, "Less LP Token Supply");
											...
tokenX.transferFrom(msg.sender, address(this), tokenXAmount); 
tokenY.transferFrom(msg.sender, address(this), tokenYAmount);
```

- tokenXAmount, tokenYAmount를 pool 비율에 맞게 넣지 않고 있음
    - liqX, liqY 중 작은 값으로 LPTokenAmount를 발행하고 있지만, msg.sender로부터 받아오는 tokenAmount들은 사용자가 input으로 준 것 그대로 받아옴 → LPTokenAmount와 맞지 않는 비율

테스트

```solidity
function testAddLiquidity() external {
        uint firstLPReturn = dex.addLiquidity(1000 ether, 1000 ether, 0); //lp = 1000
        emit log_named_uint("firstLPReturn", firstLPReturn);

        uint secondLPReturn = dex.addLiquidity(1000 ether, 2000 ether, 0); //lp = 1000
        emit log_named_uint("secondLPReturn", secondLPReturn);

        (uint x, uint y) = dex.removeLiquidity(secondLPReturn,0, 0); 
        emit log_named_uint("x", x);
        emit log_named_uint("y", y);

        assertEq(x, 1000 ether);
        assertEq(y, 2000 ether);
    }
```
<img width="541" alt="스크린샷 2023-03-26 오후 8 42 39" src="https://user-images.githubusercontent.com/127647051/227773331-89307783-9a33-4c6a-8486-0caa87ce81e1.png">

- 더 많이 add한 token에 대해 더 적은 양이 remove됨

### 파급력

- Informational → 사용자가 pool에 제공한 liquidity보다 적은 양을 회수하므로 사용자가 손해보는 구조

### 해결방안

- addLiquidity에 비율에 따른 token amount를 add할 수 있도록 추가하기

---

## 임나라

### 설명

문제 발생 - addLiquidity function

```solidity
if (totalSupply() == 0) {
    LPTokenAmount = tokenXAmount * tokenYAmount;
} else {
    LPTokenAmount = tokenXAmount * totalSupply() / tokenXpool;
}
```

- tokenXAmount, tokenYAmount를 pool 비율에 맞게 넣지 않고 있음
    - addLiquidity에서 imbalance한 token amount를 제공하더라도 그대로 pool에 add하여 LP token을 발행함
    - 최초 pool에 liquidity제공 후, LPTokenAmount를 tokenXAmount를 기준으로만 구함
    - 비율에 맞지 않는 liquidity 제공 시, Y토큰에 적은양을 amount로 주더라도 X토큰에 대한 LP token을 받을 수 있으며, 해당 값이 totalSupply로 mint됨

테스트

```solidity
function testAddLiquidity() external {
        uint firstLPReturn = dex.addLiquidity(1000 ether, 1000 ether, 0); //lp = 1000
        emit log_named_uint("firstLPReturn", firstLPReturn);
        assertEq(firstLPReturn, 1000 ether);
        
        uint secondLPReturn = dex.addLiquidity(4000 ether, 100 ether, 0);
        emit log_named_uint("secondLPReturn", secondLPReturn);

        (uint x, uint y) = dex.removeLiquidity(secondLPReturn,0, 0);
        emit log_named_uint("x", x);
        emit log_named_uint("y", y);

        assertEq(x, 4000 ether);
        assertEq(y, 100 ether);
    }
```
<img width="691" alt="스크린샷 2023-03-26 오후 8 43 18" src="https://user-images.githubusercontent.com/127647051/227773372-0cf034ca-e456-4b7c-b16e-8c4fe3eac538.png">
→ removeLiquidity시, Y토큰에 대해 더 많은 amount를 획득할 수 있게됨

### 파급력

- Critical → 공격자는 적은 양의 token을 ‎pool에 제공하여 본인이 제공한 것보다 많은 amount token을 회수할 수 있음

### 해결방안

- addLiquidity에 비율에 따른 token amount를 add할 수 있도록 추가하기

---

## 김영운

### 설명

문제 발생 - addLiquidity function

```solidity
else {
	liquidityX = _div(_mul(tokenXAmount, totalLiquidity), tokenXReserve);
	liquidityY = _div(_mul(tokenYAmount, totalLiquidity), tokenYReserve);
}

lpTokenCreated = (liquidityX < liquidityY) ? liquidityX : liquidityY;
```

- tokenXAmount, tokenYAmount를 pool 비율에 맞게 넣지 않고 있음
    - addLiquidity에서 imbalance한 token amount를 제공하더라도 그대로 pool에 add하여 LP token을 발행함
    - liquidityX, liquidityY 중 작은 값으로 lpTokenCreated를 발행하고 있지만, msg.sender로부터 받아오는 tokenAmount들은 사용자가 input으로 준 것 그대로 받아옴

테스트

```solidity
function testAddLiquidity() external {
        uint firstLPReturn = dex.addLiquidity(1000 ether, 1000 ether, 0);
        emit log_named_uint("firstLPReturn", firstLPReturn);
        assertEq(firstLPReturn, 1000 ether);
        
        uint secondLPReturn = dex.addLiquidity(4000 ether, 100 ether, 0);
        emit log_named_uint("secondLPReturn", secondLPReturn);

        (uint x, uint y) = dex.removeLiquidity(secondLPReturn,0, 0);
        emit log_named_uint("x", x);
        emit log_named_uint("y", y);

        assertEq(x, 4000 ether);
        assertEq(y, 100 ether);
    }
```

<img width="610" alt="스크린샷 2023-03-26 오후 8 44 14" src="https://user-images.githubusercontent.com/127647051/227773397-49a092ac-c33a-42f7-8d25-c6c8c8d2a8df.png">

- 더 많이 add한 token에 대해 더 적은 양이 remove됨

### 파급력

- Informational → 사용자가 pool에 제공한 liquidity보다 적은 양을 회수하므로 사용자가 손해보는 구조

### 해결방안

- addLiquidity에 비율에 따른 token amount를 add할 수 있도록 추가하기

---

## 김지우

### 설명

문제 발생 - addLiquidity function

```solidity
uint liquidityX=liquiditySum*tokenXAmount/X;
uint liquidityY=liquiditySum*tokenYAmount/Y;
lpToken=(liquidityX<liquidityY)?liquidityX:liquidityY;
```

- tokenXAmount, tokenYAmount를 pool 비율에 맞게 넣지 않고 있음
    - addLiquidity에서 imbalance한 token amount를 제공하더라도 그대로 pool에 add하여 LP token을 발행함
    - liquidityX, liquidityY 중 작은 값으로 lpTokenCreated를 발행하고 있지만, msg.sender로부터 받아오는 tokenAmount들은 사용자가 input으로 준 것 그대로 받아옴

테스트

```solidity
function testAddLiquidity() external {
        uint firstLPReturn = dex.addLiquidity(1000 ether, 1000 ether, 0);
        emit log_named_uint("firstLPReturn", firstLPReturn);
        assertEq(firstLPReturn, 1000 ether);
        
        uint secondLPReturn = dex.addLiquidity(4000 ether, 100 ether, 0);
        emit log_named_uint("secondLPReturn", secondLPReturn);

        (uint x, uint y) = dex.removeLiquidity(secondLPReturn,0, 0);
        emit log_named_uint("x", x);
        emit log_named_uint("y", y);

        assertEq(x, 4000 ether);
        assertEq(y, 100 ether);
    }
```

![스크린샷 2023-03-26 오후 6.01.34.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/41684cbe-271b-46e5-b41d-667ebc67d99f/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2023-03-26_%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE_6.01.34.png)

- 더 많이 add한 token에 대해 더 적은 양이 remove됨

### 파급력

- Informational → 사용자가 pool에 제공한 liquidity보다 적은 양을 회수하므로 사용자가 손해보는 구조

### 해결방안

- addLiquidity에 비율에 따른 token amount를 add할 수 있도록 추가하기

---

## 서준원

### 설명

문제 발생 - addLiquidity function

```solidity
if(totalSupply() == 0){
    token_liquidity_L = sqrt(reserve_x * reserve_y);
    token_amount = sqrt((tokenXAmount * tokenYAmount));
} else{
    token_amount = (tokenXAmount * 10 ** 18 * totalSupply() / reserve_x) / 10 ** 18; //비율 깨짐
}
```

- tokenXAmount, tokenYAmount를 pool 비율에 맞게 넣지 않고 있음
    - addLiquidity에서 imbalance한 token amount를 제공하더라도 그대로 pool에 add하여 LP token을 발행함

테스트

```solidity
function testAddLiquidity() external {
        uint firstLPReturn = dex.addLiquidity(1000 ether, 1000 ether, 0);
        emit log_named_uint("firstLPReturn", firstLPReturn);
        assertEq(firstLPReturn, 1000 ether);
        
        uint secondLPReturn = dex.addLiquidity(4000 ether, 100 ether, 0);
        emit log_named_uint("secondLPReturn", secondLPReturn);

        (uint x, uint y) = dex.removeLiquidity(secondLPReturn,0, 0);
        emit log_named_uint("x", x);
        emit log_named_uint("y", y);

        assertEq(x, 4000 ether);
        assertEq(y, 100 ether);
    }
```
<img width="703" alt="스크린샷 2023-03-26 오후 8 46 18" src="https://user-images.githubusercontent.com/127647051/227773497-cbc151da-b771-4653-b662-0926bfa8415c.png">

→ second addLiquidity에서 LP token 발행시, 무조건 X토큰을 기준으로 발행함. Y토큰에 적은양을 amount로 주더라도 X토큰에 대한 LP token을 받을 수 있으며, 해당 값이 totalSupply로 mint됨

→ removeLiquidity시, Y토큰에 대해 더 많은 amount를 획득할 수 있게됨

### 파급력

- Critical → 공격자는 적은 양의 token을 ‎pool에 제공하여 본인이 제공한 것보다 많은 amount token을 회수할 수 있음

### 해결방안

- addLiquidity에 비율에 따른 token amount를 add할 수 있도록 추가하기

---

## 권준우

### 설명

문제 발생 - addLiquidity function

```solidity
else {
  liquidityX = (totalLiquidity*tokenXAmount) / xBalance;
  liquidityY = (totalLiquidity*tokenYAmount) / yBalance;
  LPTokenAmount = (liquidityX < liquidityY) ? liquidityX : liquidityY;
  require(LPTokenAmount >= minimumLPTokenAmount);
  totalLiquidity += LPTokenAmount;
}
```



- tokenXAmount, tokenYAmount를 pool 비율에 맞게 넣지 않고 있음
    - addLiquidity에서 imbalance한 token amount를 제공하더라도 그대로 pool에 add하여 LP token을 발행함
    - liquidityX, liquidityY 중 작은 값으로 lpTokenCreated를 발행하고 있지만, tokenAmount들은 사용자가 input으로 준 것 그대로 받아옴

테스트

```solidity
function testAddLiquidity() external {
        uint firstLPReturn = dex.addLiquidity(1000 ether, 1000 ether, 0);
        emit log_named_uint("firstLPReturn", firstLPReturn);
        assertEq(firstLPReturn, 1000 ether);
        
        uint secondLPReturn = dex.addLiquidity(4000 ether, 100 ether, 0);
        emit log_named_uint("secondLPReturn", secondLPReturn);

        (uint x, uint y) = dex.removeLiquidity(secondLPReturn,0, 0);
        emit log_named_uint("x", x);
        emit log_named_uint("y", y);

        assertEq(x, 4000 ether);
        assertEq(y, 100 ether);
    }
```

<img width="572" alt="스크린샷 2023-03-26 오후 8 47 19" src="https://user-images.githubusercontent.com/127647051/227773550-ea9958ff-df3e-40a1-bffe-8f0efefb9fcb.png">

- 더 많이 add한 token에 대해 더 적은 양이 remove됨

### 파급력

- Informational → 사용자가 pool에 제공한 liquidity보다 적은 양을 회수하므로 사용자가 손해보는 구조

### 해결방안

- addLiquidity에 비율에 따른 token amount를 add할 수 있도록 추가하기
