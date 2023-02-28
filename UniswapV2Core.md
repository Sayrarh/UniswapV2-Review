# Uniswap V2 Core
The core contracts of Uniswap v2 are responsible for the fundamental functions of the protocol, such as the pricing algorithm and the token swaps themselves. The Uniswap V2 Core contracts consists of three contracts;

1. UniswapV2Factory
2. UniswapV2ERC20
3. UniswapV2Pair

## UniswapFactory Contract

```solidity
 function createPair(address tokenA, address tokenB) external returns (address pair) {
        require(tokenA != tokenB, 'UniswapV2: IDENTICAL_ADDRESSES');
        (address token0, address token1) = tokenA < tokenB ? (tokenA, tokenB) : (tokenB, tokenA);
        require(token0 != address(0), 'UniswapV2: ZERO_ADDRESS');
        require(getPair[token0][token1] == address(0), 'UniswapV2: PAIR_EXISTS'); // single check is sufficient
        bytes memory bytecode = type(UniswapV2Pair).creationCode;
        bytes32 salt = keccak256(abi.encodePacked(token0, token1));
        assembly {
            pair := create2(0, add(bytecode, 32), mload(bytecode), salt)
        }
        IUniswapV2Pair(pair).initialize(token0, token1);
        getPair[token0][token1] = pair;
        getPair[token1][token0] = pair; // populate mapping in the reverse direction
        allPairs.push(pair);
        emit PairCreated(token0, token1, pair, allPairs.length);
```

The createPair() in the UniswapV2 smart contract creates a new UniswapV2Pair contract for a given pair of tokens tokenA and tokenB.
<br/>
The function first checks that tokenA and tokenB are not the same and that token0 is not a zero address. Then it checks if a pair contract already exists for the given tokens using the getPair mapping, which maps pairs of tokens to their respective UniswapV2Pair contract addresses. If a pair contract already exists, the function throws an exception.
<br/>
If a new pair contract can be created, the function creates a new contract using the create2 function, which creates a new contract using the supplied bytecode and a generated salt value. The bytecode is the creation code for a UniswapV2Pair contract. The salt value is generated from the token addresses.
<br/>
After the new pair contract is created, the initialize function of the UniswapV2Pair contract is called with the two token addresses. This sets the two tokens for the new pair contract.
<br/>
The function then updates the getPair mapping to map the new pair contract address to the two token addresses, and also maps the two token addresses to the new pair contract address in reverse direction.

Finally, the function adds the new pair contract address to the allPairs array, which keeps track of all created pair contracts.

## UniswapV2ERC20 Contract

## UniswapV2Pair Contract

```solidity
uint public constant MINIMUM_LIQUIDITY = 10**3;
```

A minimum amount of liquidity tokens always exists, which is owned by address(0), to prevent instances of division by zero. This amount is referred to as MINIMUM_LIQUIDITY and is equal to one thousand tokens.

```solidity
 bytes4 private constant SELECTOR = bytes4(keccak256(bytes('transfer(address,uint256)')));

```

The constant SELECTOR is a bytes4 variable that stores the ABI selector for the transfer function of the ERC-20 token standard. This selector is used to facilitate the transfer of ERC-20 tokens between two token accounts.

```solidity
    address public factory;
    address public token0;
    address public token1;

    uint112 private reserve0;          
    uint112 private reserve1;        
    uint32  private blockTimestampLast;
    uint public price0CumulativeLast;
    uint public price1CumulativeLast;
    uint public kLast;
```
- factory: address of the contract that created this particular pool.
- token0: address of token X
- token1: address of token Y
The two variables 'token0' and 'token1' store the addresses of the two ERC-20 token contracts that can be traded within this pool.

- reserve0: reserve of token X prior to a swap     
- reserve1: reserve of token Y prior to a swap  
The two variables 'reserve0' and 'reserve1' are private variables that keep track of the reserves held for each token in the pool.         
- blockTimestampLast: is a private variable that stores the timestamp of the last block in which an exchange occurred. This is used to calculate the exchange rates between the two tokens over time.

- price0CumulativeLast: keeps a record of the cumulative price of token0 every time there is a change in 'reserve0'
- price1CumulativeLast: is a variable that maintains the cumulative price of token1 each time there is a change in 'reserve1'. These variables are used to calculate the average exchange rate between the two tokens over a given period.
- kLast: holds the value of the product of 'reserve0' and 'reserve1'(reserve0 * reserve1) as of immediately after the most recent liquidity event. In a pair exchange, the exchange rate between token0 and token1 is determined by maintaining a constant value for the product of the two reserves during trades, and 'kLast' tracks this value. <br/>
Also, whenever a liquidity provider deposits or withdraws tokens, the value of 'kLast' changes. Additionally, due to the 0.3% market fee, 'kLast' also increases slightly.

```solidity
    uint private unlocked = 1;
    modifier lock() {
        require(unlocked == 1, 'Tamswap: LOCKED');
        unlocked = 0;
        _;
        unlocked = 1;
    }
```

The 'lock' modifier ensures that the function it modifies can only be executed when 'unlocked' is equal to 1.

If 'unlocked' is indeed equal to 1, the modifier changes 'unlocked' to 0, executes the function, and then changes 'unlocked' back to 1. This is to prevent multiple instances of the function from executing simultaneously and interfering with each other(in the case of a reentry attack).
If 'unlocked' is not equal to 1, the require statement throws an error with the message "Tamswap: LOCKED" and the function cannot be executed.

```solidity
 function getReserves() public view returns (uint112 _reserve0, uint112 _reserve1, uint32 _blockTimestampLast) {
        _reserve0 = reserve0;
        _reserve1 = reserve1;
        _blockTimestampLast = blockTimestampLast;
    }
```
The function 'getReserves' is a public view function that returns the values of 'reserve0', 'reserve1', and 'blockTimestampLast' as a tuple of types 'uint112', 'uint112', and 'uint32', respectively.

```solidity
    function _safeTransfer(address token, address to, uint value) private {
        (bool success, bytes memory data) = token.call(abi.encodeWithSelector(SELECTOR, to, value));
        require(success && (data.length == 0 || abi.decode(data, (bool))), 'UniswapV2: TRANSFER_FAILED');
    }
```
The function '_safeTransfer' is a private function that attempts to transfer a certain 'value' amount of an ERC-20 token ('token') to a specified recipient ('to') using the transfer function's selector ('SELECTOR'). The function then verifies that the transfer was successful and reverts with the error message "UniswapV2": TRANSFER_FAILED" if it was not.


```solidity
  event Mint(address indexed sender, uint amount0, uint amount1);
    event Burn(address indexed sender, uint amount0, uint amount1, address indexed to);
    event Swap(
        address indexed sender,
        uint amount0In,
        uint amount1In,
        uint amount0Out,
        uint amount1Out,
        address indexed to
    );
    event Sync(uint112 reserve0, uint112 reserve1);
```
- Mint: This event is emitted when a liquidity provider deposits tokens into the pool, creating new liquidity.
- Burn: This event is emitted when a liquidity provider withdraws tokens from the pool, removing liquidity.
- Swap: This event is emitted when a swap occurs between the two tokens in the pool.
- Sync: This event is emitted when the reserves of the two tokens are updated.


```solidity
    constructor() {
        factory = msg.sender;
    }
```

The 'constructor' function sets the address of the contract that deployed the UniswapV2 contract as the 'factory' variable.

```solidity
function initialize(address _token0, address _token1) external {
        require(msg.sender == factory, 'UniswapV2: FORBIDDEN'); // sufficient check
        token0 = _token0;
        token1 = _token1;
    }
```
The 'initialize' function is to be called externally by the factory contract. It takes two addresses as arguments, which represent the two ERC-20 tokens that will be traded in the pool.

The function first checks that the caller of the function is indeed the factory contract, using the 'msg.sender' variable. If this check fails, the function will revert and the transaction will not be executed.

If the check passes, the 'token0' and 'token1' variables are set to the corresponding addresses of the ERC-20 tokens.

```solidity
 // update reserves and, on the first call per block, price accumulators
    function _update(uint balance0, uint balance1, uint112 _reserve0, uint112 _reserve1) private {
        require(balance0 <= type(uint112).max && balance1 <= type(uint112).max, 'UniswapV2: OVERFLOW');
        uint32 blockTimestamp = uint32(block.timestamp % 2**32);
        uint32 timeElapsed = blockTimestamp - blockTimestampLast; // overflow is desired
        if (timeElapsed > 0 && _reserve0 != 0 && _reserve1 != 0) {
            // * never overflows, and + overflow is desired
            price0CumulativeLast += uint(UQ112x112.encode(_reserve1).uqdiv(_reserve0)) * timeElapsed;
            price1CumulativeLast += uint(UQ112x112.encode(_reserve0).uqdiv(_reserve1)) * timeElapsed;
        }
        reserve0 = uint112(balance0);
        reserve1 = uint112(balance1);
        blockTimestampLast = blockTimestamp;
        emit Sync(reserve0, reserve1);
    }
```
This is a private function which updates the reserves and price accumulators for the pool.It is called whenever there are new funds deposited or withdrawn by the liquidity providers or when tokens are swapped by traders. The function takes in four arguments: 
- balance0 and balance1: which represent the current balances of token0 and token1 in the pool. Gotten balanceOf(address(this))
- _reserve0 and _reserve1, which represent the previous known reserves of the pool.

The function first checks that the values of balance0 and balance1 are not greater than the maximum value of an unsigned 112-bit integer (i.e., type(uint112).max) to avoid overflow issues.
Then it calculates the time elapsed since the last block using blockTimestampLast, updates price0CumulativeLast and price1CumulativeLast with the new cumulative prices based on the time elapsed and the previous reserves, and sets the new reserves and block timestamp. Finally, it emits a Sync event with the new reserve values.

```solidity
// if fee is on, mint liquidity equivalent to 1/6th of the growth in sqrt(k)
    function _mintFee(uint112 _reserve0, uint112 _reserve1) private returns (bool feeOn) {
        address feeTo = IUniswapV2Factory(factory).feeTo();
        feeOn = feeTo != address(0);
        uint _kLast = kLast; // gas savings
        if (feeOn) {
            if (_kLast != 0) {
                uint rootK = Math.sqrt(uint(_reserve0).mul(_reserve1));
                uint rootKLast = Math.sqrt(_kLast);
                if (rootK > rootKLast) {
                    uint numerator = totalSupply.mul(rootK.sub(rootKLast));
                    uint denominator = rootK.mul(5).add(rootKLast);
                    uint liquidity = numerator / denominator;
                    if (liquidity > 0) _mint(feeTo, liquidity);
                }
            }
        } else if (_kLast != 0) {
            kLast = 0;
        }
    }

```

The _mintFee() is marked as private and takes two arguments of type uint112. It returns a bool value that indicates whether the fee is enabled or not.
<br/>
The function first retrieves the feeTo address from the IUniswapV2Factory contract and checks if it is not address(0), indicating that the fee is enabled(on). It also saves the previous value of the k constant (stored in _kLast) to reduce gas costs.
<br/>
If the fee is enabled and _kLast is not zero, the function calculates the current value of the k constant as the square root of the product of _reserve0 and _reserve1. It also calculates the previous value of k as the square root of _kLast.
<br/>
If the current value of k is greater than the previous value of k, the function calculates the amount of liquidity tokens that should be minted as a fee. The liquidity amount is proportional to the increase in the square root of k.
<br/>
If the calculated liquidity amount is greater than zero, the function mints that amount of liquidity tokens and sends them to the feeTo address.
<br/>
Else If the fee is not enabled and _kLast is not zero, the function resets _kLast to zero.

<b>Note that Uniswap uses the Automated Market Maker (AMM) formula, which uses the constant product invariant k to determine token prices and liquidity. The function calculates the amount of liquidity to mint as a fee based on changes in k, which reflects changes in the product of the token reserves.</b>

```solidity
// this low-level function should be called from a contract which performs important safety checks
    function mint(address to) external lock returns (uint liquidity) {
        (uint112 _reserve0, uint112 _reserve1,) = getReserves(); // gas savings
        uint balance0 = IERC20(token0).balanceOf(address(this));
        uint balance1 = IERC20(token1).balanceOf(address(this));
        uint amount0 = balance0.sub(_reserve0);
        uint amount1 = balance1.sub(_reserve1);

        bool feeOn = _mintFee(_reserve0, _reserve1);
        uint _totalSupply = totalSupply; // gas savings, must be defined here since totalSupply can update in _mintFee
        if (_totalSupply == 0) {
            liquidity = Math.sqrt(amount0.mul(amount1)).sub(MINIMUM_LIQUIDITY);
           _mint(address(0), MINIMUM_LIQUIDITY); // permanently lock the first MINIMUM_LIQUIDITY tokens
        } else {
            liquidity = Math.min(amount0.mul(_totalSupply) / _reserve0, amount1.mul(_totalSupply) / _reserve1);
        }
        require(liquidity > 0, 'UniswapV2: INSUFFICIENT_LIQUIDITY_MINTED');
        _mint(to, liquidity);

        _update(balance0, balance1, _reserve0, _reserve1);
        if (feeOn) kLast = uint(reserve0).mul(reserve1); // reserve0 and reserve1 are up-to-date
        emit Mint(msg.sender, amount0, amount1);
    }

```

The mint function, which is an external function that is only callable by other contracts and is marked as lock. It takes a single argument 'to' of type address and returns a uint value representing the amount of liquidity tokens that were minted.
<br/>
The function first retrieves the token reserves and balances using the getReserves() function and the balanceOf function of the IERC20 interface. It calculates the amounts of each token that were deposited into the contract by subtracting the current reserves from the current balances.
<br/>
It then calls the _mintFee() function to determine whether a fee should be charged for this transaction and to calculate the amount of liquidity tokens that should be minted as a fee.
<br/>
If the total supply of liquidity tokens is zero, the function calculates the initial amount of liquidity tokens to mint as the square root of the product of the token amounts minus a constant MINIMUM_LIQUIDITY. It then mints MINIMUM_LIQUIDITY tokens and sends them to address 0x0 (effectively burning them). This step is necessary to ensure that the liquidity pool always has non-zero liquidity.
<br/>
If the total supply of liquidity tokens is non-zero, the function calculates the amount of liquidity tokens to mint based on the proportional value of each deposited token to the total value of the pool. The amount of liquidity tokens to mint is the minimum of the amounts calculated based on each deposited token.
<br/>
The function then mints the calculated amount of liquidity tokens and sends them to the specified recipient address. It updates the token reserves and calls the _update function to update the accumulated price history. If a fee was charged for this transaction, the function updates the previous value of the k constant.
<br/>
Finally, the function emits a Mint event with the details of the tokens deposited and the amount of liquidity tokens minted. If no liquidity tokens were minted, the function throws an error with the message "UniswapV2: INSUFFICIENT_LIQUIDITY_MINTED".

```solidity
// this low-level function should be called from a contract which performs important safety checks
    function burn(address to) external lock returns (uint amount0, uint amount1) {
        (uint112 _reserve0, uint112 _reserve1,) = getReserves(); // gas savings
        address _token0 = token0;                                // gas savings
        address _token1 = token1;                                // gas savings
        uint balance0 = IERC20(_token0).balanceOf(address(this));
        uint balance1 = IERC20(_token1).balanceOf(address(this));
        uint liquidity = balanceOf[address(this)];

        bool feeOn = _mintFee(_reserve0, _reserve1);
        uint _totalSupply = totalSupply; // gas savings, must be defined here since totalSupply can update in _mintFee
        amount0 = liquidity.mul(balance0) / _totalSupply; // using balances ensures pro-rata distribution
        amount1 = liquidity.mul(balance1) / _totalSupply; // using balances ensures pro-rata distribution
        require(amount0 > 0 && amount1 > 0, 'UniswapV2: INSUFFICIENT_LIQUIDITY_BURNED');
        _burn(address(this), liquidity);
        _safeTransfer(_token0, to, amount0);
        _safeTransfer(_token1, to, amount1);
        balance0 = IERC20(_token0).balanceOf(address(this));
        balance1 = IERC20(_token1).balanceOf(address(this));

        _update(balance0, balance1, _reserve0, _reserve1);
        if (feeOn) kLast = uint(reserve0).mul(reserve1); // reserve0 and reserve1 are up-to-date
        emit Burn(msg.sender, amount0, amount1, to);
    }
```
The burn function allows Liquidity providers to remove liquidity from the pool and receive back their proportionate share of the underlying tokens(liquidity + accummulated rewards).
<br/>
The function first retrieves the current reserves of the pool using the getReserves function, and the balances of the two tokens using the balanceOf function of the IERC20 interface. It then calculates the amount of liquidity tokens to burn based on the user's balance of the liquidity token.
<br/>
The function then calls _mintFee to calculate and mint all accumulated fees that may be applicable. It then calculates the amounts of token0 and token1 that the user is entitled to receive back based on their proportionate share of the pool, and transfers those amounts to the user using the _safeTransfer function.
<br/>
After transferring the tokens, the function updates the reserves of the pool using the _update function, and sets the value of kLast if applicable. Finally, the function emits a Burn event to indicate that liquidity has been removed from the pool.


```solidity
// this low-level function should be called from a contract which performs important safety checks
    function swap(uint amount0Out, uint amount1Out, address to, bytes calldata data) external lock {
        require(amount0Out > 0 || amount1Out > 0, 'UniswapV2: INSUFFICIENT_OUTPUT_AMOUNT');
        (uint112 _reserve0, uint112 _reserve1,) = getReserves(); // gas savings
        require(amount0Out < _reserve0 && amount1Out < _reserve1, 'UniswapV2: INSUFFICIENT_LIQUIDITY');

        uint balance0;
        uint balance1;
        { // scope for _token{0,1}, avoids stack too deep errors
        address _token0 = token0;
        address _token1 = token1;
        require(to != _token0 && to != _token1, 'UniswapV2: INVALID_TO');
        if (amount0Out > 0) _safeTransfer(_token0, to, amount0Out); // optimistically transfer tokens
        if (amount1Out > 0) _safeTransfer(_token1, to, amount1Out); // optimistically transfer tokens
        if (data.length > 0) ITamswapCallee(to).tamswapCall(msg.sender, amount0Out, amount1Out, data);
        balance0 = IERC20(_token0).balanceOf(address(this));
        balance1 = IERC20(_token1).balanceOf(address(this));
        }
        uint amount0In = balance0 > _reserve0 - amount0Out ? balance0 - (_reserve0 - amount0Out) : 0;
        uint amount1In = balance1 > _reserve1 - amount1Out ? balance1 - (_reserve1 - amount1Out) : 0;
        require(amount0In > 0 || amount1In > 0, 'UniswapV2: INSUFFICIENT_INPUT_AMOUNT');
        { // scope for reserve{0,1}Adjusted, avoids stack too deep errors
        uint balance0Adjusted = balance0.mul(1000).sub(amount0In.mul(3));
        uint balance1Adjusted = balance1.mul(1000).sub(amount1In.mul(3));
        require(balance0Adjusted.mul(balance1Adjusted) >= uint(_reserve0).mul(_reserve1).mul(1000**2), 'UniswapV2: K');
        }

        _update(balance0, balance1, _reserve0, _reserve1);
        emit Swap(msg.sender, amount0In, amount1In, amount0Out, amount1Out, to);
    }
```


The swap function allows traders to swap tokens by specifying the output amounts and the recipient of the tokens.

The function first performs safety checks to ensure that the output amounts are not zero and that there is enough liquidity in the pool to complete the trade. It then retrieves the current reserves and token balances, and transfers the output amounts to the specified recipient.
<br/>
If the data argument is provided, it calls the uniswapCall function of the recipient contract, passing the input and output amounts and the msg.sender as arguments.
<br/>
Next, the function calculates the input amounts required to complete the trade, based on the remaining balance of each token in the pool. It then checks that there is enough input liquidity to complete the trade, and updates the reserves with the new balances.
<br/>
Finally, the function emits a Swap event with information about the input and output amounts and the recipient of the tokens.


```solidity
 // force balances to match reserves
    function skim(address to) external lock {
        address _token0 = token0; // gas savings
        address _token1 = token1; // gas savings
        _safeTransfer(_token0, to, IERC20(_token0).balanceOf(address(this)).sub(reserve0));
        _safeTransfer(_token1, to, IERC20(_token1).balanceOf(address(this)).sub(reserve1));
    }

```

The skim function transfers the excess balance of each token (i.e., the difference between the contract's balance and its recorded reserve) to the specified to address. This is useful when the contract's balance has been slightly increased due to rounding errors or fees.

```solidity
  // force reserves to match balances
    function sync() external lock {
        _update(IERC20(token0).balanceOf(address(this)), IERC20(token1).balanceOf(address(this)), reserve0, reserve1);
    }
```

The sync function updates the reserves to match the actual balances. This is useful when the recorded reserves have become out of sync with the actual balances due to some external operation, such as a deposit or withdrawal of tokens. The function calls _update with the current balances and the current reserves, effectively resetting the reserve values.



## Example: How the contract works in practice
```
A user Sayrarh wants to provide liquidity for a pair contract that already exists e.g WETH/USDC pair.
- Sayrarh holds 500WETH and 1000USDC

- token0 = address of WETH contract
- token1 = address of USDC contract
- assuming the previous reserve of the WETH/USDC pool is 10/200 respectively.
- reserve0 = 10 WETH (current WETH token reserve)
- reserve1 = 200 USDC (current USDC token reserve)


- assuming the blockTimestampLast = 1677419978
- uint32 blockTimestamp = uint32(block.timestamp % 2**32)
- blockTimestamp = uint32(1677420235 % 4294967296) = 1677420235
- timeElapsed = 1677420235 - 1677419978 = 257

- kLast = reserve0 * reserve1 = 10 * 200 = 2000

////mint() function////
- balance0 = 510 WETH (total amount of WETH tokens in the contract, note that 500WETH tokens provided by sayrarh was sent in by the periphery contract)
- balance1 = 1200 USDC (total amount of USDC tokens in the contract,note that 1000USDC tokens provided by sayrarh was sent in by the periphery contract)

- amount0 = 510(WETH) - 10(WETH) = 500(WETH)
- amount1 = 1200(USDC) - 200(USDC) = 1000(USDC)
To get the amount of liquidity that was actually provided by sayrarh

bool feeOn = _mintFee(_reserve0, _reserve1);
If the feeTo address is not address(0) and _kLast is not zero
 {
        uint rootK = Math.sqrt(uint(_reserve0).mul(_reserve1));
        uint rootKLast = Math.sqrt(_kLast);
            if (rootK > rootKLast) {
                uint numerator = totalSupply.mul(rootK.sub(rootKLast));
                 uint denominator = rootK.mul(5).add(rootKLast);
                 uint liquidity = numerator / denominator;
                if (liquidity > 0) _mint(feeTo, liquidity);
     }

rootK = Math.sqrt(uint(10)*200) = 4000000
rootKLast = Math.sqrt(2000) = 4000000
price0cummulativeLast = 
price1cummulativeLast =

```



- Uniswap charges 1/6 protocol fee out of 0.3% trader fee



