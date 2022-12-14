# solidity-app
application of solidity

## ERC20

学习完`ERC20`代币标准，便可以发行代币了。

`ERC20`是一项以太坊代币标准，是从EIP-20提案经过以太坊社区不断讨论验证后通过而来的，是由Vitalik Buterin于2015年提出，是以太坊的第20号代币标准。详细标准参考如下地址：

https://github.com/ethereum/EIPs/blob/master/EIPS/eip-20.md

代币规范，也就是接口。后续需要根据接口实现具体功能。接口的方法修改为external修饰符。

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;

interface IERC20 {
    
    /**
     * Returns the total token supply.
     * 返回总的代币数量
     */
    function totalSupply() external view returns (uint256);

    /**
     * Returns the account balance of another account with address _owner.
     * 返回某个账户的当前代币余额
     */
    function balanceOf(address _owner) external view returns (uint256 balance);

    /**
     * Transfers _value amount of tokens to address _to, and MUST fire the Transfer event. 
     * The function SHOULD throw if the message caller's account balance does not have enough tokens to spend.
     * 转账函数
     */
    function transfer(address _to, uint256 _value) external returns (bool success);

    /**
     * Transfers _value amount of tokens from address _from to address _to, and MUST fire the Transfer event.
     * The transferFrom method is used for a withdraw workflow, allowing contracts to transfer tokens on your behalf. 
     * This can be used for example to allow a contract to transfer tokens on your behalf and/or to charge fees in sub-currencies. 
     * The function SHOULD throw unless the _from account has deliberately authorized the sender of the message via some mechanism.
     * Note Transfers of 0 values MUST be treated as normal transfers and fire the Transfer event.
     * 授权转账
     */
    function transferFrom(address _from, address _to, uint256 _value) external returns (bool success);

    /**
     * Allows _spender to withdraw from your account multiple times, up to the _value amount. 
     * If this function is called again it overwrites the current allowance with _value.
     * 授权
     */
    function approve(address _spender, uint256 _value) external returns (bool success);

    /**
     * Returns the amount which _spender is still allowed to withdraw from _owner.
     * 返回_owner授权给_spender的额度
     */
    function allowance(address _owner, address _spender) external view returns (uint256 remaining);

    /**
     * MUST trigger when tokens are transferred, including zero value transfers.
     */
    event Transfer(address indexed _from, address indexed _to, uint256 _value);

    /**
     * MUST trigger on any successful call to approve(address _spender, uint256 _value).
     */
    event Approval(address indexed _owner, address indexed _spender, uint256 _value);
}
```

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
import "./IERC20.sol";
pragma solidity ^0.8.0;
contract ERC20 is IERC20 {

    //public类型的状态变量，会自动生成相对应的get方法；添加override表示生成的方法会重写继承自父合约与变量同名的函数,也可以自己自助生成相应的方法
    mapping (address => uint256) private balance;

    mapping (address => mapping (address => uint256)) private allow;

    //代币总量
    uint256 private total;
    //代币的名称
    string public name;
    //代币的代号
    string public symbol;

    uint8 public decimals = 18;

    address owner;

    function totalSupply() external override view returns (uint256){
        return total;
    }

    function balanceOf(address _owner) external override view returns (uint256){
        return balance[_owner];
    }

    function allowance(address _owner, address _spender) external override view returns (uint256){
        return allow[_owner][_spender];
    }



    constructor(string memory _name, string memory _symbol) {
        name = _name;
        symbol = _symbol;
        owner = msg.sender;
    }

    function transfer(address _to, uint256 _value) external override returns (bool){
        balance[msg.sender] -= _value;
        balance[_to]  += _value;
        emit Transfer(msg.sender, _to, _value);
        return true;
    }

    function approve(address _spender, uint256 _value) external override returns (bool success){
        allow[msg.sender][_spender]  = _value;
        emit Approval(msg.sender, _spender,  _value);
        return true;
    }

    function transferFrom(address _from, address _to, uint256 _value) external override returns (bool){
        allow[_from][msg.sender] -= _value;
        balance[_from] -= _value;
        balance[_to] += _value;
        emit Transfer(_from, _to, _value);
        return true;
    }

    function mint(uint amount) public{
        require(msg.sender == owner, "no permission");
        balance[msg.sender] += amount;
        total += amount;
        emit Transfer(address(0), msg.sender, amount);
    }

    function burn(uint amount) public{
        balance[msg.sender] -= amount;
        total -= amount;
        emit Transfer(msg.sender, address(0), amount);
    }
}
```

![image-20221112111451365](README.assets/image-20221112111451365.png)

## Faucet

实现了一个简易的代币水龙头。用户每隔24h可以获取一次代币。

我们的业务逻辑如下：首先编写代币合约，发布代币。接下来在Faucet水龙头合约中，我们mint一定数量的代币。向外提供一个`acquireFaucet`的方法。EOA账号可以通过该方法每隔24h获取一次代币。

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;
import "./IERC20.sol";
import "./ERC20.sol";
contract Faucet {

    uint256 public allowedAmount = 100000000000000000000;

    address public tokenAddress;

    uint256 constant ONE_DAY = 86400;

    //记录每个地址领取代币的时间，后面可以设定每隔24h可以领取一次
    mapping (address => uint) acquiredAddress;

    //每当每个地址领取了一次代币，便触发当前事件
    event sendToken(address indexed _receiver, uint256 indexed _amount);

    constructor(address _tokenAddress) {
        tokenAddress = _tokenAddress;
        ERC20 token = ERC20(tokenAddress);
        token.mint(100000000000000000000000000);

    }


    function acquireFaucet() external {
        uint number = acquiredAddress[msg.sender];
        uint nowTime = block.timestamp;
        IERC20 token = IERC20(tokenAddress);
        require(token.balanceOf(address(this)) >= allowedAmount, "Faucet empty");
        if(number != 0){
            require(nowTime - number >= ONE_DAY, "Please try again after 24 hours from your original request.");
        }
        token.transfer(msg.sender, allowedAmount);
        acquiredAddress[msg.sender] = block.timestamp;
        emit sendToken(msg.sender, allowedAmount);
        //number为0，直接领取
    }
}
```

![image-20221112213159364](README.assets/image-20221112213159364.png)

![image-20221112213303430](README.assets/image-20221112213303430.png)

![image-20221112213419601](README.assets/image-20221112213419601.png)

![image-20221112213525087](README.assets/image-20221112213525087.png)

![image-20221112213613549](README.assets/image-20221112213613549.png)

![image-20221112213700196](README.assets/image-20221112213700196.png)

![image-20221112213759012](README.assets/image-20221112213759012.png)

## Airdrop

空投。说白了就是白嫖。也是项目方用来进行营销的一种手段。项目方通过智能合约给EOA账号发放指定数量的代币。依然需要用到前面我们编写的ERC20和IERC20这两个合约。

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;
import "./IERC20.sol";
contract Airdrop {
    
    /**
     * _token表示代币合约地址
     * _airdropList表示需要空投的地址列表
     * _amounts表示一共需要空投代币的数量
     */
    function airdropTokens(address _token, address[] calldata _airdropList, uint256[] calldata _amounts) external{
        //地址列表和数量列表的长度应该相同
        require(_airdropList.length  == _amounts.length, "length of address do not equal with amounts");
        IERC20 token = IERC20(_token);
        uint amounts = checkSum(_amounts);
        //require(token.allowance(msg.sender, address(this)) >= amounts, "Approve ERC20 Token");
        for (uint i = 0; i < _airdropList.length; i++) {
            token.transferFrom(msg.sender, _airdropList[i], _amounts[i]);
        }
    }

    /**
     * 计算一共需要空投的代币总量
     */
    function checkSum(uint256[] calldata _amounts) internal pure returns (uint number){
        for (uint i = 0; i < _amounts.length; i++) {
            number += _amounts[i];
        }
    }
}
```

![image-20221113200936792](README.assets/image-20221113200936792.png)

![image-20221113201113845](README.assets/image-20221113201113845.png)

## ERC721

`BTC`以及`ETH`这类代币可以称之为同质化代币。因为挖出来的每一枚代币和其他代币没有什么不同，是同质化的。但是在实际生活中，很多物品却不是同质化的，比如艺术品等。因此以太坊社区提出了`ERC721`标准，用来抽象非同质化物品。

<a href='https://eips.ethereum.org/all'>EIP</a> 

`EIP`全称 `Ethereum Imporvement Proposals`(以太坊改进建议), 是以太坊开发者社区提出的改进建议。

`ERC`全称 Ethereum Request For Comment (以太坊意见征求稿), 用以记录以太坊上应用级的各种开发标准和协议。如`ERC20(Token Standard)`以及`ERC721(Non-Fungible Token Standard)`。

`EIP`包含`ERC`。

在介绍ERC之前，我们先梳理一下`type`的用法。

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;
import "./IERC165.sol";
contract TypeDemo {
    
    //type(整形).max/min可以返回最大/最小值
    function max() external pure returns (uint){
        return type(uint).max;
    }

    function mim() external pure returns (uint){
        return type(uint).min;
    }

    //type(Contract):name(合约名称)、creationCode(创建字节码)、runtimeCode(运行字节码)
    //type(Interface).interfaceId(获取一个接口的interfaceId 如果接口只有一个方法 则为方法的selector，否者为所有方法的selector的异或后结果)
    //下面使用了三种方式来获取interfaceId，结果均是一致的
    function typeId() external pure returns (bytes4 a, bytes4 b, bytes4 c){
        a = type(IERC165).interfaceId;
        b = bytes4(keccak256('supportsInterface(bytes4)'));
        c = IERC165.supportsInterface.selector;
    }
}
```

![image-20221113215821399](README.assets/image-20221113215821399.png)

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;
interface IERC165 {
    
    /**
     * EIP-165:Standard Interface Detection.检验某个合约有没有实现该接口。如何校验呢？
     * The interface identifier for this interface is 0x01ffc9a7. You can calculate this by running bytes4(keccak256('supportsInterface(bytes4)'));
     * or using the Selector contract above.
     */
    function supportsInterface(bytes4 interfaceId) external view returns (bool);
}
```

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;
import "./IERC165.sol";
contract ERC165 is IERC165{

  /**
   * 0x01ffc9a7 ===
   *   bytes4(keccak256('supportsInterface(bytes4)'))
   */
    bytes4 private constant ERC165_InterfaceId = 0x01ffc9a7;

    mapping (bytes4 => bool) supportedInterfaces;

    constructor() {
        registerInterface(ERC165_InterfaceId);
    }

    function registerInterface(bytes4 _interfaceId) internal{
        require(_interfaceId != 0xffffffff);
        supportedInterfaces[_interfaceId] = true;
    }

    //特别注意：定长数组属于值类型，不属于引用类型，所以参数位置不需要添加memory
    function supportsInterface(bytes4 _interfaceId) external override view returns (bool){
        return supportedInterfaces[_interfaceId];
    }
}
```



```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;
import "./IERC165.sol";
/**
 * @title ERC-721 Non-Fungible Token Standard
 * @dev see https://github.com/ethereum/EIPs/blob/master/EIPS/eip-721.md
 */
interface IERC721 is IERC165 {
    
    /// @dev This emits when ownership of any NFT changes by any mechanism.
    ///  This event emits when NFTs are created (`from` == 0) and destroyed
    ///  (`to` == 0). Exception: during contract creation, any number of NFTs
    ///  may be created and assigned without emitting Transfer. At the time of
    ///  any transfer, the approved address for that NFT (if any) is reset to none.
    /// 转账事件，转出地址from，转入地址to，以及tokenId
    event Transfer(address indexed from, address indexed to, uint256 indexed tokenId);

    /// @dev This emits when the approved address for an NFT is changed or
    ///  reaffirmed. The zero address indicates there is no approved address.
    ///  When a Transfer event emits, this also indicates that the approved
    ///  address for that NFT (if any) is reset to none.
    ///  授权事件，记录授权地址owner，被授权地址approved和tokenid
    event Approval(address indexed owner, address indexed approved, uint256 indexed tokenId);

    /// @dev This emits when an operator is enabled or disabled for an owner.
    ///  The operator can manage all NFTs of the owner.
    ///  批量授权事件，记录批量授权的发出地址owner，被授权地址operator和授权与否的approved
    event ApprovalForAll(address indexed owner, address indexed operator, bool approved);

    /// @notice Count all NFTs assigned to an owner
    /// @dev NFTs assigned to the zero address are considered invalid, and this
    ///  function throws for queries about the zero address.
    /// @param owner An address for whom to query the balance
    /// @return balance The number of NFTs owned by `_owner`, possibly zero
    /// 返回某个地址所拥有的所有的NFT数量
    function balanceOf(address owner) external view returns (uint256 balance);

    /// @notice Find the owner of an NFT
    /// @dev NFTs assigned to zero address are considered invalid, and queries
    ///  about them do throw.
    /// @param tokenId The identifier for an NFT
    /// @return owner The address of the owner of the NFT
    /// 返回某个tokenId所属的主人地址
    function ownerOf(uint256 tokenId) external view returns (address owner);

    /// @notice Transfers the ownership of an NFT from one address to another address
    /// @dev Throws unless `msg.sender` is the current owner, an authorized
    ///  operator, or the approved address for this NFT. Throws if `_from` is
    ///  not the current owner. Throws if `_to` is the zero address. Throws if
    ///  `_tokenId` is not a valid NFT. When transfer is complete, this function
    ///  checks if `_to` is a smart contract (code size > 0). If so, it calls
    ///  `onERC721Received` on `_to` and throws if the return value is not
    ///  `bytes4(keccak256("onERC721Received(address,address,uint256,bytes)"))`.
    /// @param from The current owner of the NFT
    /// @param to The new owner
    /// @param tokenId The NFT to transfer
    /// @param data Additional data with no specified format, sent in call to `_to`
    /// 安全转账（如果接收方是合约地址，会要求实现ERC721Receiver接口）。参数为转出地址from，接收地址to和tokenId
    function safeTransferFrom(
        address from,
        address to,
        uint256 tokenId,
        bytes memory data
    ) external;

    /// @notice Transfers the ownership of an NFT from one address to another address
    /// @dev This works identically to the other function with an extra data parameter,
    ///  except this function just sets data to "".
    /// @param from The current owner of the NFT
    /// @param to The new owner
    /// @param tokenId The NFT to transfer
    function safeTransferFrom(
        address from,
        address to,
        uint256 tokenId
    ) external;

    /// @notice Transfer ownership of an NFT -- THE CALLER IS RESPONSIBLE
    ///  TO CONFIRM THAT `_to` IS CAPABLE OF RECEIVING NFTS OR ELSE
    ///  THEY MAY BE PERMANENTLY LOST
    /// @dev Throws unless `msg.sender` is the current owner, an authorized
    ///  operator, or the approved address for this NFT. Throws if `_from` is
    ///  not the current owner. Throws if `_to` is the zero address. Throws if
    ///  `_tokenId` is not a valid NFT.
    /// @param from The current owner of the NFT
    /// @param to The new owner
    /// @param tokenId The NFT to transfer
    function transferFrom(
        address from,
        address to,
        uint256 tokenId
    ) external;

    /// @notice Change or reaffirm the approved address for an NFT
    /// @dev The zero address indicates there is no approved address.
    ///  Throws unless `msg.sender` is the current NFT owner, or an authorized
    ///  operator of the current owner.
    /// @param to The new approved NFT controller
    /// @param tokenId The NFT to approve
    /// 授权另一个地址使用你的NFT。参数为被授权地址approve和tokenId
    function approve(address to, uint256 tokenId) external;

    /// @notice Enable or disable approval for a third party ("operator") to manage
    ///  all of `msg.sender`'s assets
    /// @dev Emits the ApprovalForAll event. The contract MUST allow
    ///  multiple operators per owner.
    /// @param operator Address to add to the set of authorized operators
    /// @param _approved True if the operator is approved, false to revoke approval
    /// 将自己持有的该系列NFT批量授权给某个地址operator
    function setApprovalForAll(address operator, bool _approved) external;

    /// @param tokenId The NFT to find the approved address for
    /// @return operator The approved address for this NFT, or the zero address if there is none
    /// 查询tokenId被批准给了哪个地址
    function getApproved(uint256 tokenId) external view returns (address operator);

    /// @notice Query if an address is an authorized operator for another address
    /// @param owner The address that owns the NFTs
    /// @param operator The address that acts on behalf of the owner
    /// @return True if `_operator` is an approved operator for `_owner`, false otherwise
    /// 查询某地址的NFT是否批量授权给了另一个operator地址
    function isApprovedForAll(address owner, address operator) external view returns (bool);
}
```

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;
/**
 * @title ERC721 token receiver interface
 * @dev Interface for any contract that wants to support safeTransfers
 * from ERC721 asset contracts.
 * 如果进行NFT转账时，接收方是一个合约地址，那么必须要实现IERC721Receiver接口，具有onERC721Received方法，否则NFT直接被打入黑洞
 */
interface IERC721Receiver {
    
  /**
   * @notice Handle the receipt of an NFT
   * @dev The ERC721 smart contract calls this function on the recipient
   * after a `safeTransfer`. This function MUST return the function selector,
   * otherwise the caller will revert the transaction. The selector to be
   * returned can be obtained as `this.onERC721Received.selector`. This
   * function MAY throw to revert and reject the transfer.
   * Note: the ERC721 contract address is always the message sender.
   * @param operator The address which called `safeTransferFrom` function
   * @param from The address which previously owned the token
   * @param tokenId The NFT identifier which is being transferred
   * @param data Additional data with no specified format
   * @return `bytes4(keccak256("onERC721Received(address,address,uint256,bytes)"))`
   */
    function onERC721Received(
        address operator,
        address from,
        uint tokenId,
        bytes calldata data
    ) external returns (bytes4);
}
```

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;
import "./ERC165.sol";
import "./IERC721.sol";
import "./IERC721Metadata.sol";
import "./IERC721Receiver.sol";
import "./Address.sol";

contract ERC721 is ERC165, IERC721 {

    using Address for address;

  // Equals to `bytes4(keccak256("onERC721Received(address,address,uint256,bytes)"))`
  // which can be also obtained as `IERC721Receiver(0).onERC721Received.selector`
  bytes4 private constant ERC721_RECEIVED = 0x150b7a02;

/*
   * 0x80ac58cd ===
   *   bytes4(keccak256('balanceOf(address)')) ^
   *   bytes4(keccak256('ownerOf(uint256)')) ^
   *   bytes4(keccak256('approve(address,uint256)')) ^
   *   bytes4(keccak256('getApproved(uint256)')) ^
   *   bytes4(keccak256('setApprovalForAll(address,bool)')) ^
   *   bytes4(keccak256('isApprovedForAll(address,address)')) ^
   *   bytes4(keccak256('transferFrom(address,address,uint256)')) ^
   *   bytes4(keccak256('safeTransferFrom(address,address,uint256)')) ^
   *   bytes4(keccak256('safeTransferFrom(address,address,uint256,bytes)'))
   */
  bytes4 private constant ERC721_InterfaceId = 0x80ac58cd;

    constructor(){
        registerInterface(ERC721_InterfaceId);
    }
    //地址和该地址的NFT数量的映射关系
    mapping (address => uint) balances;
    //tokenId和所属地址之间的映射关系
    mapping (uint => address) owners;

    //某tokenId和授权地址的映射关系(每个token在同一时间只可以授权给一个地址)
    mapping (uint => address) tokenApprovals;
    //将owner地址授权给operator的映射关系
    mapping (address => mapping (address => bool)) operatorApprovals;

    //返回某个地址拥有的NFT的数量
    function balanceOf(address _owner) external view override returns (uint256 balance){
        require(_owner != address(0), "black hole address");
        balance = balances[_owner];
    }

    //返回某个tokenId所属的地址
    function ownerOf(uint256 _tokenId) public view override returns (address owner){
        owner = owners[_tokenId];
        require(owner != address(0), "token is in the black hole");
    }
  /**
   * @dev Safely transfers the ownership of a given token ID to another address
   * If the target address is a contract, it must implement `onERC721Received`,
   * which is called upon a safe transfer, and return the magic value
   * `bytes4(keccak256("onERC721Received(address,address,uint256,bytes)"))`; otherwise,
   * the transfer is reverted.
   * Requires the msg sender to be the owner, approved, or operator
   * @param _from current owner of the token
   * @param _to address to receive the ownership of the given token ID
   * @param _tokenId uint256 ID of the token to be transferred
   * @param _data bytes data to send along with a safe transfer check
   * 安全的转账，为了保证接收地址如果是合约，如果没有实现onERC721Received会出错
   */
    function safeTransferFrom(address _from,address _to,uint256 _tokenId,bytes memory _data) public override{
        transferFrom(_from, _to, _tokenId);
        require(_checkERC721Received(_from, _to, _tokenId, _data));
    }

    //如果是合约，则必须实现该接口，否则NFT发送到该合约便消失了
    function _checkERC721Received(address _from,address _to,uint256 _tokenId,bytes memory _data)internal returns (bool){
        if(!_to.isContract()){
            return true;
        }
        bytes4 code = IERC721Receiver(_to).onERC721Received(msg.sender, _from, _tokenId, _data);
        return code == ERC721_RECEIVED;
    }

 
    function safeTransferFrom(address _from,address _to,uint256 _tokenId) external override{
        safeTransferFrom(_from, _to, _tokenId, "");
    }

  /**
   * @dev Transfers the ownership of a given token ID to another address
   * Usage of this method is discouraged, use `safeTransferFrom` whenever possible
   * Requires the msg sender to be the owner, approved, or operator
   * @param _from current owner of the token
   * @param _to address to receive the ownership of the given token ID
   * @param _tokenId uint256 ID of the token to be transferred
  */
    function transferFrom(address _from, address _to, uint256 _tokenId) public override{
        require(_isApprovedOrOwner(msg.sender, _tokenId));
        require(_to != address(0));
        //清除授权
        _clearApproval(_from, _tokenId);
        _removeTokenFrom(_from, _tokenId);
        _addTokenTo(_to, _tokenId);
        emit Transfer(_from, _to, _tokenId);
    }

    function _addTokenTo(address _to, uint _tokenId)internal {
        require(owners[_tokenId] == address(0));
        balances[_to] += 1;
        owners[_tokenId] = _to;
    }

    function _removeTokenFrom(address _from, uint _tokenId)internal {
        require(ownerOf(_tokenId) == _from);
        balances[_from] -= 1;
        owners[_tokenId] = address(0);
    }

    //清除授权信息
    function _clearApproval(address _owner, uint _tokenId) internal {
        require(ownerOf(_tokenId) == _owner);
        tokenApprovals[_tokenId] = address(0);
    }

    //是否是授权地址或者是拥有者
    function _isApprovedOrOwner(address _caller, uint _tokenId) internal view returns (bool){
        address owner = ownerOf(_tokenId);
        //三种情况：1.拥有者 2.当前tokenId授权给了该地址 3.将当前地址下的所有NFT全部授权给了该地址
        return (_caller == owner || getApproved(_tokenId) == _caller || isApprovedForAll(owner, _caller));
    }



    /**
   * @dev Approves another address to transfer the given token ID
   * The zero address indicates there is no approved address.
   * There can only be one approved address per token at a given time.
   * Can only be called by the token owner or an approved operator.
   * @param _to address to be approved for the given token ID
   * @param _tokenId uint256 ID of the token to be approved
   * 将tokenId授权给to地址；
   */
    function approve(address _to, uint256 _tokenId) external override{
        //获取当前tokenId的拥有者
        address owner = ownerOf(_tokenId);
        //不要自己给自己发送
        require(owner != _to);
        //仅当前tokenId拥有者或者授权的合约地址可以调用该方法;isApprovedForAll查询owner地址的NFT是否批量授权给msg.sender调用者
        require(msg.sender == owner || isApprovedForAll(owner, msg.sender));
        //将_tokenId授权给_to地址
        tokenApprovals[_tokenId] = _to;
        emit Approval(owner, _to, _tokenId);
    }

  /**
   * @dev Sets or unsets the approval of a given operator
   * An operator is allowed to transfer all tokens of the sender on their behalf
   * @param _operator operator address to set the approval
   * @param _approved representing the status of the approval to be set
   * 将全部代币授权给operator地址或者撤销授权
   */
    function setApprovalForAll(address _operator, bool _approved) external override{
        require(_operator != msg.sender);
        operatorApprovals[msg.sender][_operator] = _approved;
        emit ApprovalForAll(msg.sender, _operator, _approved);
    }

  /**
   * @dev Gets the approved address for a token ID, or zero if no address set
   * Reverts if the token ID does not exist.
   * @param _tokenId uint256 ID of the token to query the approval of
   * @return operator currently approved for the given token ID
   * 查询当前tokenId的授权地址
   */
    function getApproved(uint256 _tokenId) public override view returns (address operator){
        require(_exists(_tokenId));
        operator = tokenApprovals[_tokenId];
    }

    /**
   * @dev Returns whether the specified token exists
   * @param _tokenId uint256 ID of the token to query the existence of
   * @return whether the token exists
   */
  function _exists(uint256 _tokenId) internal view returns (bool) {
    address owner = owners[_tokenId];
    return owner != address(0);
  }

  /**
   * @dev Tells whether an operator is approved by a given owner
   * @param _owner owner address which you want to query the approval of
   * @param _operator operator address which you want to query the approval of
   * @return bool whether the given operator is approved by the given owner
   */
    function isApprovedForAll(address _owner, address _operator) public override view returns (bool){
        return operatorApprovals[_owner][_operator];
    }

    /**
   * @dev Internal function to mint a new token
   * Reverts if the given token ID already exists
   * @param _to The address that will own the minted token
   * @param _tokenId uint256 ID of the token to be minted by the msg.sender
   */
  function _mint(address _to, uint256 _tokenId) internal {
    require(_to != address(0));
    _addTokenTo(_to, _tokenId);
    emit Transfer(address(0), _to, _tokenId);
  }

  /**
   * @dev Internal function to burn a specific token
   * Reverts if the token does not exist
   * @param _tokenId uint256 ID of the token being burned by the msg.sender
   */
  function _burn(uint256 _tokenId) internal {
    address owner = ownerOf(_tokenId);
    require(msg.sender == owner, "you can not burn someone else's token");
    _clearApproval(owner, _tokenId);
    _removeTokenFrom(owner, _tokenId);
    emit Transfer(owner, address(0), _tokenId);
  }
}
```

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;
import "./IERC721.sol";
/**
 * @title ERC-721 Non-Fungible Token Standard, optional metadata extension
 * @dev See https://github.com/ethereum/EIPs/blob/master/EIPS/eip-721.md
 */
interface IERC721Metadata is IERC721 {
    
    //返回代币名称
    function name() external view returns (string memory);

    //返回代币代号
    function symbol() external view returns (string memory);

    //通过tokenId查询链接url
    function tokenURI(uint256 tokenId) external view returns (string memory);
}
```

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;
import "./ERC165.sol";
import "./ERC721.sol";
import "./IERC721Metadata.sol";
import "./Strings.sol";

contract ERC721Metadata is ERC165, ERC721, IERC721Metadata{

    string internal tokenName;

    string internal tokenSymbol;

    using Strings for uint256;

  /**
   * 0x5b5e139f ===
   *   bytes4(keccak256('name()')) ^
   *   bytes4(keccak256('symbol()')) ^
   *   bytes4(keccak256('tokenURI(uint256)'))
   */
    bytes4 private constant ERC721Metadata_InterfaceId = 0x5b5e139f;

    constructor(string memory _name, string memory _symbol) {
        tokenName = _name;
        tokenSymbol = _symbol;
        registerInterface(ERC721Metadata_InterfaceId);
    }

    function name() external override view returns (string memory){
        return tokenName;
    }

    function symbol() external override view returns (string memory){
        return tokenSymbol;
    }

    function tokenURI(uint256 tokenId) external override view returns (string memory){
        require(_exists(tokenId));
        string memory baseURI = _baseURI();
        return bytes(baseURI).length > 0 ? string(abi.encodePacked(baseURI, tokenId.toString())) : "";
    }

    //定义一个方法，发行ERC721代币，需要继承当前合约，并且实现该方法
    function _baseURI() internal view virtual returns (string memory){
        return "";
    }
}
```

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;
import "./ERC721Metadata.sol";
import "./SafeMath.sol";

contract Road2Web3 is ERC721Metadata{

    uint public MAX_SUPPLY = 1000;

    using SafeMath for uint256;

    uint private index = 1;

    constructor(string memory _name, string memory _symbol) ERC721Metadata(_name, _symbol) {
        
    }

    //设置ipfs
     function _baseURI() internal pure override returns (string memory){
        return "ipfs://QmeDEvsWpBk429UJj9JTrgtHZpNJksvPVK4GfQv439UpXW/";
    }

    function mint() external{
        require(index <= MAX_SUPPLY, "All items have been minted");
        _mint(msg.sender, index);
    }
}
```

除此之外还需要一些工具类，就不展示了。

## 荷兰拍卖

荷兰拍卖这种拍卖方式较为独特，采用降价的形式进行。指的是竞价标的物从高到低依次递减的方式，直至第一个竞价人应价时成交的一种拍卖。

因为荷兰拍卖持续的时间比较久，这样就可以避免`gas`的激增。

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;
import "../erc721/Road2Web3.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract DutchAuction is Ownable, Road2Web3("Road2Web3", "r2w3"){
    //NFT总数量
    uint256 public constant COLLECTION_SIZE = 1000;
    //竞拍开始价格
    uint256 public constant AUCTION_START_PRICE = 1 ether;
    //竞拍地板价格
    uint256 public constant AUCTION_FLOOR_PRICE = 0.1 ether;
    //竞拍持续时间
    uint256 public constant AUCTION_DURATION_TIME = 10 minutes;
    //无人竞拍时，多久衰减一次价格
    uint256 public constant AUCTION_DECLINE_INTERVAL = 1 minutes;
    //价格衰减率
    uint256 public constant AUCTION_DECLINE_RATE = (AUCTION_START_PRICE - AUCTION_FLOOR_PRICE) / (AUCTION_DURATION_TIME / AUCTION_DECLINE_INTERVAL);

    uint256 public auctionStartTime;

    uint256 private baseTokenURI;

    constructor(){
        auctionStartTime = block.timestamp;
    }

    //只有合约的部署者才可以mint No.1号NFT，后续要进行拍卖
    function mintGenesisNFT() external onlyOwner {
        uint mintIndex = totalSupply() + 1;
        _mint(msg.sender, mintIndex);
        _addTokenIndex(mintIndex);
    }

    //项目方开始拍卖前需要调用该方法
    function setAuctionStartTime(uint256 _timestamp) external onlyOwner {
        auctionStartTime = _timestamp;
    }

    /**
     * block.timestamp < auctionTime,还未开始拍卖-----开始价格
     * block.timestamp - auctionTime > duration_time, 拍卖已经结束-----地板价
     */
    function getAuctionPrice() public view returns (uint){
        if(block.timestamp < auctionStartTime){
            return AUCTION_START_PRICE;
        }else if((block.timestamp - auctionStartTime) >= AUCTION_DURATION_TIME){
            return AUCTION_FLOOR_PRICE;
        }else {
            //衰减了多少次
            uint numberOfDecline = (block.timestamp - auctionStartTime) / AUCTION_DECLINE_INTERVAL;
            return AUCTION_START_PRICE - (AUCTION_DECLINE_RATE * numberOfDecline);
        }
    }
    

    function auctionAndMint(uint number)external payable{
        //建立局部变量，减少gas费
        uint _startTime = auctionStartTime;
        require(_startTime != 0 && block.timestamp >= _startTime, "Sale have not been started");
        require(totalSupply() + number <= COLLECTION_SIZE, "All items have benn minted");
        uint totalCost = getAuctionPrice() * number;
        require(msg.value >= totalCost, "Not enough ETH to mint");
        payable(msg.sender).transfer(msg.value - totalCost);
        for (uint i = 0; i < number; i++) {
            uint mintIndex = totalSupply() + 1;
            _mint(msg.sender, mintIndex);
            _addTokenIndex(mintIndex);
        }

    }

    //提取所有的eth主币
    function withdraw() external onlyOwner{
        (bool success, ) = msg.sender.call{value: address(this).balance}("");
        require(success, "withdraw failed");
    }

    fallback() external payable{}

    receive() external payable{}
}
```

## 英式拍卖

英式拍卖就是我们平时经常见到的拍卖形式。在拍卖期间，最高价者竞拍得到标的物品。英式拍卖通常对某单个标的物品进行拍卖。比如NFT发售等场景，一般不会采取这种方式。但是比如`1`编号的某个NFT可能会进行拍卖。那么便可以使用这种方式。比如下面案例我们就实现了使用荷兰拍卖NFT藏品，但是针对特殊的No.1号藏品，我们采取的是项目方先mint之后，采取英式拍卖的方式来进行拍卖。整体业务逻辑较为复杂，需要涉及到多个账号。

![image-20221119120455203](README.assets/image-20221119120455203.png)

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;
import "../erc721/Road2Web3.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
//采用英式拍卖拍卖NFT中的第1号藏品
//设定拍卖时间为12h，由最高竞拍者获得
//竞拍时需要将主币转移到合约中，如果目前出价最高，则拍下不允许撤销拍卖
//如果有其他人出更高价格，那么当前竞拍者可以提取出代币，再次进行竞拍；或者直接放弃等等......
contract EnglishAuction is Ownable{
    
    //要拍卖的是哪个NFT
    Road2Web3 public nftAddress;
    //NFT中的几号藏品
    uint public tokenId;

    //出售方，可以是个人，也可以是项目方，也就是拥有上述tokenId的账户
    address public seller;

    uint public auctionStartTime;

    uint public constant AUCTION_DURATION_TIME = 5 minutes;

    //最高报价
    uint public highestBid;

    //最高报价的出价人
    address public highestBidder;

    //不是最高竞价的竞拍者和竞拍价格的映射
    mapping (address => uint) public withdraw;

    constructor(uint _tokenId, address _nftAddress){
        tokenId = _tokenId;
        nftAddress = Road2Web3(_nftAddress);
    }

    //卖家授权挂牌拍卖
    function listToSell() external{
        require(nftAddress.ownerOf(tokenId) == msg.sender, "you don't the item");
        seller = msg.sender;
        //(bool result, ) = address(nftAddress).delegatecall(abi.encodeWithSignature("approve(address,uint256)", address(this), tokenId));
        //nftAddress.approve(address(this), tokenId);
        //require(result, "approve failed");
    }

    function setAuctionStartTime(uint _timestamp) external onlyOwner{
        auctionStartTime = _timestamp;
    }

    function bid() external payable{
        require(block.timestamp >= auctionStartTime, "auction has not started");
        require((auctionStartTime + AUCTION_DURATION_TIME) > block.timestamp, "auction has ended");
        require(msg.value > highestBid, "you must bid higger");
        if(highestBid != 0){
            //如果有新的竞拍者出了更高的价格，那么之前的竞拍者就不是最高出价了，它可以选择退出拍卖，拿回现金；或者继续追加
            withdraw[highestBidder] += highestBid;
        }
        highestBid = msg.value;
        highestBidder = msg.sender;
    }
    //竞拍失败的竞拍者可以随时取出竞拍金
    function withdrawMoney() external {
        uint balance = withdraw[msg.sender];
        require(balance != 0, "you can not withdraw");
        withdraw[msg.sender] = 0;
        payable(msg.sender).transfer(balance);
    }
    //竞拍成功的竞拍者可以mint No.1 NFT
    function claim() external{
        require(msg.sender == highestBidder, "only highest bidder can mint");
        nftAddress.safeTransferFrom(seller, highestBidder, tokenId);
    }
}
```

## Merkle Tree

`Merkle Tree`是区块链底层大量使用的一种技术。`Merkle Tree`最主要的功能是可以对大量的数据进行有效、安全、高效的验证。比如可以在区块链中对于交易来进行验证。在我们开发过程中，我们可以利用`Merkle Tree`来进行白名单验证等。比如`Aptos`发放空投等，验证某个地址是否在空投白名单中。还比如最近因为`FTX`暴雷而导致的`CEX`信任危机。`CEX`提出的资产证明方案。同时附上<a href='https://vitalik.ca/index.html'>Vitalik Buterin's website</a>

![image-20221121231224697](README.assets/image-20221121231224697.png)

![img](README.assets/K1X0b1i.png)

上图是`Merkle Tree`的结构。相邻两个节点进行`hash`运算之后的`hash`值，再两两进行运算，最终会得到一个`ROOT hash`值。

`Merkle Tree`可以允许对大型数据结构的内容进行有效以及安全的验证(`Merkle Proof`)。比如图中绿色标注的`Charlie`，如果想验证该节点是否在`Merkle Tree`中，那么它的`Merkle Proof`为`hash(David)`、`hash(L,R) 70ETH`、`hash(L,R) 1236ETH`。为什么呢？因为对`Charlie`和`David`进行`hash`值运算之后，会得到`hash(L2,R2) 174ETH`，而它与`hash(L,R) 70ETH`再次进行运算会得到`hash(L,R) 244ETH`。该值和最后`hash(L,R) 1236ETH`运算之后便会得到`ROOT hash`值。通过比对给定的`ROOT hash`值和计算得到的`ROOT hash`值比对，这样便可以确认该节点是否在`Merkle Tree`中了。



![image-20221123224432534](README.assets/image-20221123224432534.png)

如上图所示，如果我希望验证蓝色标注的地址是否在白名单中，那么我们可以使用`Merkle Tree`来维护白名单。我们验证时`Proof`只需要绿色标注的部分即可。	

**如何生成Merkle Tree，可以参考如下两个地址：**

<a href='https://lab.miguelmota.com/merkletreejs/example/'>网页页面</a>

<a href='https://github.com/merkletreejs/merkletreejs'>Github-MerkleTreeJS</a>

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.7;
library MerkleTreeProof {
    
    /**
     * 利用leaf节点和给定的proof，推算出一个root值，如果和给定的root值相等，则说明leaf在Merkle Tree中
     */
    function verify(bytes32[] calldata proof, bytes32 root, bytes32 leaf) public pure returns(bool){
        return _proofProcedure(proof, leaf) == root;
    }

    function _proofProcedure(bytes32[] calldata _proof, bytes32 _leaf) internal pure returns(bytes32){
        bytes32 computedHash = _leaf;
        for (uint i = 0; i < _proof.length; i++) {
            computedHash = _combineHash(computedHash, _proof[i]);
        }
        return computedHash;
    }

    /**
     * 排序之后进行hash运算，保障无论顺序如何得到的hash值永远是相同的
     */
    function _combineHash(bytes32 _b1, bytes32 _b2) internal pure returns(bytes32){
        return _b1 < _b2 ? keccak256(abi.encodePacked(_b1, _b2)) : keccak256(abi.encodePacked(_b2, _b1));
    }
}
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.7;
import "../erc721/Road2Web3.sol";
import "./MerkleTreeProof.sol";
/**
 * 利用白名单来mint，如果用户很多，怎么验证某个用户是否在白名单中。挨个遍历太浪费gas。可以使用Merkle Proof
 */
contract WhiteList is Road2Web3("Road2Web3Dao", "r2w3"){
    //记录Merkle Tree Root
    bytes32 immutable public merkleRoot;
    //是否已经mint过的地址
    mapping (address => bool) public mintedAddress;

    uint public tokenId;

    constructor(bytes32 _merkleRoot){
        merkleRoot = _merkleRoot;
    }

    function mint(bytes32[] calldata  _proof) external{
        require(mintedAddress[msg.sender] == false, "current address already minted");
        require(_verify(_leafHash(msg.sender), _proof), "invalid merkle proof");
        tokenId ++;
        mint(tokenId);
    }

    //这种方法也可以将address类型转换成了bytes类型
    function _leafHash(address _account) internal pure returns (bytes32){
        return keccak256(abi.encodePacked(_account));
    }

    function _verify(bytes32 _leaf, bytes32[] calldata _proof) internal view returns (bool){
        return MerkleTreeProof.verify(_proof, merkleRoot, _leaf);
    }   
}
```

## 数字签名

数字签名是一种证明你拥有私钥却无需暴露私钥的方式。如图所示便是`metamask`钱包进行签名的截图。

<img src="README.assets/image-20221127210822034.png" alt="image-20221127210822034" style="zoom:50%;" />

以太坊和比特币所使用的数字签名算法叫做双椭圆曲线数字签名算法(`ECDSA`)。基于双椭圆曲线私钥-公钥对来对数据进行签名。主要有三个作用：

- 身份认证：证明签名方是私钥持有人
- 不可否认：发送方不能否认发送过该数据
- 完整性：消息在传输过程中无法被修改

`ECDSA`流程如下：

签名者利用**私钥**(隐私)来对**消息**(公开)创建**签名**(公开)

其他人使用消息和签名恢复签名者的公钥并验证签名。

### 创建签名

**1.打包消息**：在以太坊的`ECDSA`标准中，被签名的消息是一组数据的`keccak256`哈希值，所以其类型为`bytes32`类型 。我们可以把任意想要签名的数据利用`abi.encodePacked()`进行打包，然后使用`keccak256`计算哈希值，用来作为消息。

```solidity
/**
     * 在实际交互过程中，第二个参数地址可以直接获取，但是为了我们此处测试方便，直接进行赋值
     */
    function packedMessageHash(string memory _message, address _account) public pure returns (bytes32) {
        bytes memory data = abi.encodePacked(_message, _account);
        return keccak256(data);
    }
```

**2.计算以太坊签名消息**：消息可以是能被执行的交易，也可以是简单的验证身份等。为了避免用户误签名了恶意交易信息，`EIP191`提倡在消息前加上`\x19Ethereum Signed Message:\n32`字符信息，并且在做一次`keccak256`哈希运算。作为以太坊签名消息。该消息不可以用于执行交易。

```solidity
/**
     * https://ethereum.org/en/developers/docs/apis/json-rpc/#eth_sign
     * https://eips.ethereum.org/EIPS/eip-191
     */
    function ethSignedHashMessage(bytes32 _hash) public pure returns (bytes32){
        return keccak256(abi.encodePacked("\x19Ethereum Signed Message:\n32", _hash));
    }
```

**3.利用钱包来进行签名**：比如在上述案例中，我们就使用`metamask`在访问某个网站时进行签名验证。`metamask`的`personal_sign`方法会自动将消息转换成以太坊签名消息。所以我们使用`metamask`时无需主动调用该方法。

只需要提供上述的`_message`和`_account`数据即可。需要特别注意的的地址需要和当前钱包的地址相同。

首先先对需要签名的消息进行哈希运算。

```
_message:Welcome to Road2Web3Dao.This Request will not cause any gas fees.
_account:0xF8c3f049908D3E924845AB8b1CAEb10C96CE57fb
```

运算得到的哈希值为`0x48105d62aad2ebe1cc4deaf476195564b0f3d9ecdb460f8b787243e4bf526e70`

![image-20221128212244625](README.assets/image-20221128212244625.png)

然后在`Chrome开发者工具-Console`中依次输入如下指令(但是得保证`metamask`钱包处于连接状态)：

```
account = "0xF8c3f049908D3E924845AB8b1CAEb10C96CE57fb"
hash = "0x48105d62aad2ebe1cc4deaf476195564b0f3d9ecdb460f8b787243e4bf526e70"
ethereum.request({method: "personal_sign", params: [account, hash]})

0x2b61e7b5d090985a5522cd78b1925f5d8386ec2b5d6ab425dd068f60496f40cf2fa0a35030ecd1eae79af553fbaf70864adabb25def1b50d6596feeacb48c5361c
```

![image-20221128223259007](README.assets/image-20221128223259007.png)



下面这种方式也是ok的。上面的消息是`bytes32`，下面的消息是字符。

```
account = "0xF8c3f049908D3E924845AB8b1CAEb10C96CE57fb"
message = "Welcome to Road2Web3Dao.This Request will not cause any gas fees. wallet address is 0xF8c3f049908D3E924845AB8b1CAEb10C96CE57fb"
ethereum.request({method: "personal_sign", params: [account, message]})

0x75901ebbc431338941d59a88a09c8667a61ef7a2cdd2f1ad5133ebf2dfae0aec6c3315ebb6d657e4f0e02787b758385a4310b3fdc786dbcd21111759ba939e441c
```

![image-20221128213804566](README.assets/image-20221128213804566.png)

### 验证签名

**4.通过签名和消息来恢复公钥**：`签名`是由数学算法生成的。这里我们使用的是`rsv签名`，`签名`中包含`r, s, v`三个值的信息。而后，我们可以通过`r, s, v`及`以太坊签名消息`来求得`公钥`。下面的`recoverSigner()`函数实现了上述步骤，它利用`以太坊签名消息 _msgHash`和`签名 _signature`恢复`公钥`

**但是最终上述两种方式，第一种方式可以正常恢复公钥，第二种方式暂时无法恢复公钥。不清楚metamask在处理字符消息时做了何种处理。**

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.7;
contract ECDSASignature {
    
    /**
     * 在实际交互过程中，第二个参数地址可以直接获取，但是为了我们此处测试方便，直接进行赋值
     */
    function packedMessageHash(string memory _message, address _account) public pure returns (bytes32) {
        bytes memory data = abi.encodePacked(_message, _account);
        return keccak256(data);
    }


    /**
     * https://ethereum.org/en/developers/docs/apis/json-rpc/#eth_sign
     * https://eips.ethereum.org/EIPS/eip-191
     */
    function ethSignedHashMessage(bytes32 _hash) public pure returns (bytes32){
        return keccak256(abi.encodePacked("\x19Ethereum Signed Message:\n32", _hash));
    }


        // @dev 从_msgHash和签名_signature中恢复signer地址
    function recoverSigner(bytes32 _msgHash, bytes memory _signature) public pure returns (address){
        // 检查签名长度，65是标准r,s,v签名的长度
        require(_signature.length == 65, "invalid signature length");
        bytes32 r;
        bytes32 s;
        uint8 v;
        // 目前只能用assembly (内联汇编)来从签名中获得r,s,v的值
        assembly {
            /*
            前32 bytes存储签名的长度 (动态数组存储规则)
            add(sig, 32) = sig的指针 + 32
            等效为略过signature的前32 bytes
            mload(p) 载入从内存地址p起始的接下来32 bytes数据
            */
            // 读取长度数据后的32 bytes
            r := mload(add(_signature, 0x20))
            // 读取之后的32 bytes
            s := mload(add(_signature, 0x40))
            // 读取最后一个byte
            v := byte(0, mload(add(_signature, 0x60)))
        }
        // 使用ecrecover(全局函数)：利用 msgHash 和 r,s,v 恢复 signer 地址
        return ecrecover(_msgHash, v, r, s);
    }
}
```

针对上述问题，后续打算针对`packedMessageHash`函数做一些修改。如果此时传递的是单个参数，看能否签名验证通过。

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.7;
contract ECDSASignature2 {
    
    /**
     * 在实际交互过程中，第二个参数地址可以直接获取，但是为了我们此处测试方便，直接进行赋值
     */
    function packedMessageHash(string memory _message) public pure returns (bytes32) {
        bytes memory data = abi.encodePacked(_message);
        return keccak256(data);
    }


    /**
     * https://ethereum.org/en/developers/docs/apis/json-rpc/#eth_sign
     * https://eips.ethereum.org/EIPS/eip-191
     */
    function ethSignedHashMessage(bytes32 _hash) public pure returns (bytes32){
        return keccak256(abi.encodePacked("\x19Ethereum Signed Message:\n32", _hash));
    }


        // @dev 从_msgHash和签名_signature中恢复signer地址
    function recoverSigner(bytes32 _msgHash, bytes memory _signature) public pure returns (address){
        // 检查签名长度，65是标准r,s,v签名的长度
        require(_signature.length == 65, "invalid signature length");
        bytes32 r;
        bytes32 s;
        uint8 v;
        // 目前只能用assembly (内联汇编)来从签名中获得r,s,v的值
        assembly {
            /*
            前32 bytes存储签名的长度 (动态数组存储规则)
            add(sig, 32) = sig的指针 + 32
            等效为略过signature的前32 bytes
            mload(p) 载入从内存地址p起始的接下来32 bytes数据
            */
            // 读取长度数据后的32 bytes
            r := mload(add(_signature, 0x20))
            // 读取之后的32 bytes
            s := mload(add(_signature, 0x40))
            // 读取最后一个byte
            v := byte(0, mload(add(_signature, 0x60)))
        }
        // 使用ecrecover(全局函数)：利用 msgHash 和 r,s,v 恢复 signer 地址
        return ecrecover(_msgHash, v, r, s);
    }
}
```

操作流程：

1.调用`packedMessageHash`函数来对消息进行哈希运算。哈希运算之后的消息使用`metamask`来对其进行签名。

分别执行上述过程。

2.我们调用`packedMessageHash`函数处理过的哈希值再次进行`ethSignedHashMessage`函数运算。最终得到的以太坊哈希和签名之后的消息最终通过调用`recoverSigner`函数来恢复原始的签名账户。



## NFT交易所

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.7;
import "../erc721/IERC721Receiver.sol";
import "../erc721/IERC721.sol";
contract NFTSwap is IERC721Receiver{

    fallback() external payable{}
    //需要接收eth主币
    receive() external payable{}

    //NFTSwap需要中转接收用户挂单发送过来的NFT，所以需要实现IERC721Receiver，否则无法接收
    function onERC721Received(address operator, address from,uint tokenId,bytes calldata data) external override returns (bytes4){
        return IERC721Receiver.onERC721Received.selector;
    }

    constructor() {
        
    }

    event List(address indexed seller, address indexed nftAddr, uint256 indexed tokenId, uint256 price);

    event Buy(address indexed buyer, address indexed nftAddr, uint256 indexed tokenId, uint256 price);

    event Revoke(address indexed seller, address indexed nftAddr, uint256 indexed tokenId);

    event Update(address indexed seller, address indexed nftAddr, uint256 indexed tokenId, uint256 newPrice);

    struct Order{
        address owner;
        uint price;
    }

    //NFT合约地址以及对应的tokenId和订单的映射关系
    mapping (address => mapping (uint256 => Order)) nftList;

    /**
     * 卖家挂单上架nft,将指定的NFT发送到当前合约中来
     */
    function list(address _nftAddr, uint256 _tokenId, uint256  _price) public{
        IERC721 _nft = IERC721(_nftAddr);
        require(_nft.getApproved(_tokenId) == address(this), "approve contract first");
        require(_price > 0);
        Order storage _order = nftList[_nftAddr][_tokenId];
        _order.owner = msg.sender;
        _order.price = _price;
        _nft.safeTransferFrom(msg.sender, address(this), _tokenId);
        emit List(msg.sender, _nftAddr, _tokenId, _price);
    }

    /**
     * 撤销挂单
     */
    function revoke(address _nftAddr, uint256 _tokenId) public{
        Order storage _order = nftList[_nftAddr][_tokenId];
        require(msg.sender == _order.owner, "You are not the owner.You do not have permission");
        IERC721 _nft = IERC721(_nftAddr);
        require(_nft.ownerOf(_tokenId) == address(this), "wrong arguments");
        _nft.safeTransferFrom(address(this), msg.sender, _tokenId);
        delete nftList[_nftAddr][_tokenId];
        emit Revoke(msg.sender, _nftAddr, _tokenId);
    }

    function update(address _nftAddr, uint256 _tokenId, uint256 _newPrice) public{
        require(_newPrice > 0, "invalid price");
        Order storage _order = nftList[_nftAddr][_tokenId];
        require(_order.owner == msg.sender, "you do not have permission");
        IERC721 _nft = IERC721(_nftAddr);
        require(_nft.ownerOf(_tokenId) ==  address(this), "wrong parameter");
        _order.price = _newPrice;
        emit Update(msg.sender, _nftAddr, _tokenId, _newPrice);
    }

    function buy(address _nftAddr, uint256  _tokenId) public payable{
        Order storage _order = nftList[_nftAddr][_tokenId];
        require(_order.price > 0, "invalid price");
        require(msg.value >= _order.price, "insufficient price");
        IERC721 _nft = IERC721(_nftAddr);
        require(_nft.ownerOf(_tokenId) == address(this), "wrong parameter");
        _nft.safeTransferFrom(address(this), msg.sender, _tokenId);
        payable(msg.sender).transfer(msg.value - _order.price);
        payable(_order.owner).transfer(_order.price);
        delete nftList[_nftAddr][_tokenId];
        emit Buy(msg.sender, _nftAddr, _tokenId, _order.price);
    }
}
```

##  链上随机数

方式一是利用一些链上成员变量作为参数，利用`keccak256`函数来获取伪随机数。很多项目也的确使用该种方法来生成随机数。但是这些项目也无一例外的全部被攻击了。攻击者可以铸造他们想要的`NFT`，而不是随机获取。

```solidity
uint index = uint(keccak256(abi.encodePacked(nonce, msg.sender, block.difficulty, block.timestamp))) % totalSize;
```

https://forum.openzeppelin.com/t/understanding-the-meebits-exploit/8281/2

**方式二使用链下生成随机数**，然后通过预言机把随机数上传到链上。`ChainLink`提供`VRF`（可验证随机函数）服务。开发者可以通过支付`LINK`代币来获取随机数。

![Chainlnk VRF](README.assets/39-1-8ce51bfb6a4ef7e77b54af56022d402b.png)

<a href='https://docs.chain.link/vrf/v2/introduction/'>Introduction to Chainlink VRF</a>

> Randomness is very difficult to generate on blockchains. This is because every node on the blockchain must come to the same conclusion and form a consensus. Even though random numbers are versatile and useful in a variety of blockchain applications, they cannot be generated natively in smart contracts. The solution to this issue is [**Chainlink VRF**](https://docs.chain.link/vrf/v2/introduction/), also known as Chainlink Verifiable Random Function.

通过上述文档的介绍，可以有两种方式实现产生随机数。一种是订阅，预先充值；一种是每次请求的时候使用`LINK`代币来进行支付。文档主要介绍的是第二种方式。

![Vrf v2 Direct funding method end to end diagram](README.assets/v2-direct-funding-e2e.webp)

`ChainLink VRF`同时使用了链上组件以及链下组件：

> - [VRF v2 Wrapper (on-chain component)](https://github.com/smartcontractkit/chainlink/blob/develop/contracts/src/v0.8/VRFV2Wrapper.sol): A wrapper for the VRF Coordinator that provides an interface for consuming contracts.
> - [VRF v2 Coordinator (on-chain component)](https://github.com/smartcontractkit/chainlink/blob/develop/contracts/src/v0.8/VRFCoordinatorV2.sol): A contract designed to interact with the VRF service. It emits an event when a request for randomness is made, and then verifies the random number and proof of how it was generated by the VRF service.
> - VRF service (off-chain component): Listens for requests by subscribing to the VRF Coordinator event logs and calculates a random number based on the block hash and nonce. The VRF service then sends a transaction to the `VRFCoordinator` including the random number and a proof of how it was generated.

`VRF wrapper`通过调用`coordinator`来处理请求的过程如下：

1. **The consuming contract must inherit [VRFV2WrapperConsumerBase](https://github.com/smartcontractkit/chainlink/blob/develop/contracts/src/v0.8/VRFV2WrapperConsumerBase.sol) and implement the `fulfillRandomWords` function, which is the *callback VRF function*.** Submit your VRF request by calling the `requestRandomness` function in the [VRFV2WrapperConsumerBase](https://github.com/smartcontractkit/chainlink/blob/develop/contracts/src/v0.8/VRFV2WrapperConsumerBase.sol) contract. Include the following parameters in your request:
   - `requestConfirmations`: The number of block confirmations the VRF service will wait to respond. The minimum and maximum confirmations for your network can be found [here](https://docs.chain.link/vrf/v2/direct-funding/supported-networks/#configurations).
   - `callbackGasLimit`: The maximum amount of gas to pay for completing the callback VRF function.
   - `numWords`: The number of random numbers to request. You can find the maximum number of random values per request for your network in the [Supported networks](https://docs.chain.link/vrf/v2/direct-funding/supported-networks/#configurations) page.
2. The consuming contract calls the [VRFV2Wrapper](https://github.com/smartcontractkit/chainlink/blob/develop/contracts/src/v0.8/VRFV2Wrapper.sol) `calculateRequestPrice` function to estimate the total transaction cost to fulfill randomness. Then the consuming contract calls the [LinkToken](https://github.com/smartcontractkit/chainlink/blob/develop/contracts/src/v0.4/LinkToken.sol) `transferAndCall` function to pay the wrapper with the calculated request price. This method sends LINK tokens and executes the [VRFV2Wrapper](https://github.com/smartcontractkit/chainlink/blob/develop/contracts/src/v0.8/VRFV2Wrapper.sol) `onTokenTransfer` logic. This triggers the VRF [VRF Coordinator](https://github.com/smartcontractkit/chainlink/blob/develop/contracts/src/v0.8/VRFCoordinatorV2.sol) `requestRandomWords` function to request randomness. 
3. The VRF coordinator emits an event.
4. The event is picked up by the VRF service and waits for the specified number of block confirmations to respond back to the VRF coordinator with the random values and a proof (`requestConfirmations`).
5. The VRF coordinator verifies the proof on-chain. Then, it calls back the wrapper contract `fulfillRandomWords` function.
6. Finally, the VRF Wrapper calls back your consuming contract.

```solidity
// SPDX-License-Identifier: MIT
// An example of a consumer contract that directly pays for each request.
pragma solidity ^0.8.7;

import "./ConfirmedOwner.sol";
import "./VRFV2WrapperConsumerBase.sol";

/**
 * Request testnet LINK and ETH here: https://faucets.chain.link/
 * Find information on LINK Token Contracts and get the latest ETH and LINK faucets here: https://docs.chain.link/docs/link-token-contracts/
 */

/**
 * THIS IS AN EXAMPLE CONTRACT THAT USES HARDCODED VALUES FOR CLARITY.
 * THIS IS AN EXAMPLE CONTRACT THAT USES UN-AUDITED CODE.
 * DO NOT USE THIS CODE IN PRODUCTION.
 */

contract VRFv2DirectFundingConsumer is
    VRFV2WrapperConsumerBase,
    ConfirmedOwner
{
    event RequestSent(uint256 requestId, uint32 numWords);
    event RequestFulfilled(
        uint256 requestId,
        uint256[] randomWords,
        uint256 payment
    );

    struct RequestStatus {
        uint256 paid; // amount paid in link
        bool fulfilled; // whether the request has been successfully fulfilled
        uint256[] randomWords;
    }
    mapping(uint256 => RequestStatus)
        public s_requests; 

    // past requests Id.
    uint256[] public requestIds;
    uint256 public lastRequestId;

    // Depends on the number of requested values that you want sent to the
    // fulfillRandomWords() function. Test and adjust
    // this limit based on the network that you select, the size of the request,
    // and the processing of the callback request in the fulfillRandomWords()
    // function.
    uint32 callbackGasLimit = 100000;

    // The default is 3, but you can set this higher.
    uint16 requestConfirmations = 3;

    // For this example, retrieve 2 random values in one request.
    // Cannot exceed VRFV2Wrapper.getConfig().maxNumWords.
    uint32 numWords = 2;

    // Address LINK - hardcoded for Goerli
    address linkAddress = 0x326C977E6efc84E512bB9C30f76E30c160eD06FB;

    // address WRAPPER - hardcoded for Goerli
    address wrapperAddress = 0x708701a1DfF4f478de54383E49a627eD4852C816;

    constructor() ConfirmedOwner(msg.sender) VRFV2WrapperConsumerBase(linkAddress, wrapperAddress){}

    //Click the requestRandomWords() function to send the request for random values to Chainlink VRF. 
    //MetaMask opens and asks you to confirm the transaction.
    function requestRandomWords() external onlyOwner returns (uint256 requestId){
        requestId = requestRandomness(
            callbackGasLimit,
            requestConfirmations,
            numWords
        );
        s_requests[requestId] = RequestStatus({
            paid: VRF_V2_WRAPPER.calculateRequestPrice(callbackGasLimit),
            randomWords: new uint256[](0),
            fulfilled: false
        });
        requestIds.push(requestId);
        lastRequestId = requestId;
        emit RequestSent(requestId, numWords);
        return requestId;
    }

    //After you approve the transaction, Chainlink VRF processes your request. 
    //Chainlink VRF fulfills the request and returns the random values to your contract in a callback to the fulfillRandomWords() function.
    function fulfillRandomWords(uint256 _requestId,uint256[] memory _randomWords) internal override {
        require(s_requests[_requestId].paid > 0, "request not found");
        s_requests[_requestId].fulfilled = true;
        s_requests[_requestId].randomWords = _randomWords;
        emit RequestFulfilled(
            _requestId,
            _randomWords,
            s_requests[_requestId].paid
        );
    }

    function getRequestStatus(
        uint256 _requestId
    )
        external
        view
        returns (uint256 paid, bool fulfilled, uint256[] memory randomWords)
    {
        require(s_requests[_requestId].paid > 0, "request not found");
        RequestStatus memory request = s_requests[_requestId];
        return (request.paid, request.fulfilled, request.randomWords);
    }

    /**
     * Allow withdraw of Link tokens from the contract
     */
    function withdrawLink() public onlyOwner {
        LinkTokenInterface link = LinkTokenInterface(linkAddress);
        require(
            link.transfer(msg.sender, link.balanceOf(address(this))),
            "Unable to transfer"
        );
    }
}
```

将合约部署到Goerli测试网中，同时发布源码。

![image-20221220223110117](README.assets/image-20221220223110117.png)

VRF利用回调函数将随机数返回给当前合约。

利用链上随机数来获取随机`mint`的`tokenId`。

下面图示第一次抽取的是5，第二次再次抽取时，容量要变小。我们采取的策略是把最后一个9剔除出去，但是9还没被用过，所以用5号位置来存储9.即便下次再次抽取到的是5，那么此时取出来的是9.

![image-20221223171525272](README.assets/image-20221223171525272.png)

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.7;
contract RandomNumber {

    uint256 public totalSupply = 100; // 总供给

    uint256[100] public ids; // 用于计算可供mint的tokenId

    uint256 public mintCount; // 已mint数量

    /**
     * 利用链上产生的随机数来随机mint NFT，最大的难点在于随机tokenId的实现
     * 大体的思路：设定有totalSupply个杯子，每个杯子旁边有一个带有对应数字的小球
     * 那么，每次利用randomIndex先从杯子中获取，如果杯子中已经被取过了，则会取到的是其他后放入的球
     * 主要的逻辑在于三目运算符
     * 从当前杯子中取数字；如果ids[randomIndex] == 0表示这个杯子之前没有被抽取过，直接取出当前下标对应的数字；随后将最后一个杯子或者杯子旁边的数字赋值给刚刚该杯子
     * 因为下一次再次取时，容量会小一个；所以我们就将最后一个杯子剔除出去
     */
    function pickRandomUniqueId(uint256 random) public returns (uint256 tokenId) {
        uint256 len = totalSupply - mintCount; // 可mint数量
        require(len > 0, "mint close"); // 所有tokenId被mint完了
        uint256 randomIndex = random % len; // 获取链上随机数
        
        tokenId = ids[randomIndex] != 0 ? ids[randomIndex] : randomIndex; // 获取tokenId
        ids[randomIndex] = ids[len - 1] == 0 ? len - 1 : ids[len - 1]; // 更新ids 列表
        ids[len - 1] = 0; // 删除最后一个元素，能返还gas
        mintCount ++;
    }
}
```

##  ERC1155

无论是`ERC20`还是`ERC721`，均是对应的单个代币对应一个合约。但是在一个大型游戏系统中，游戏可能会存在很多类型的装备，每个装备可能都对应一个`NFT`。如果每一个装备设置一个合约，那么海量的合约也不便于管理。`ERC1155`标准允许一个合约中包含多个同质化以及非同质化代币。`ERC1155`主要用在Gamefi中，比如Sandbox、Decentraland等。下面列举了主要的代码逻辑。

```solidity
// SPDX-License-Identifier: MIT
// OpenZeppelin Contracts (last updated v4.7.0) (token/ERC1155/IERC1155.sol)

pragma solidity ^0.8.0;

import "../erc721/IERC165.sol";

/**
 * @dev Required interface of an ERC1155 compliant contract, as defined in the
 * https://eips.ethereum.org/EIPS/eip-1155[EIP].
 *
 * _Available since v3.1._
 */
interface IERC1155 is IERC165 {
    /**
     * @dev Emitted when `value` tokens of token type `id` are transferred from `from` to `to` by `operator`.
     */
    event TransferSingle(address indexed operator, address indexed from, address indexed to, uint256 id, uint256 value);

    /**
     * @dev Equivalent to multiple {TransferSingle} events, where `operator`, `from` and `to` are the same for all
     * transfers.
     */
    event TransferBatch(
        address indexed operator,
        address indexed from,
        address indexed to,
        uint256[] ids,
        uint256[] values
    );

    /**
     * @dev Emitted when `account` grants or revokes permission to `operator` to transfer their tokens, according to
     * `approved`.
     */
    event ApprovalForAll(address indexed account, address indexed operator, bool approved);

    /**
     * @dev Emitted when the URI for token type `id` changes to `value`, if it is a non-programmatic URI.
     *
     * If an {URI} event was emitted for `id`, the standard
     * https://eips.ethereum.org/EIPS/eip-1155#metadata-extensions[guarantees] that `value` will equal the value
     * returned by {IERC1155MetadataURI-uri}.
     */
    event URI(string value, uint256 indexed id);

    /**
     * @dev Returns the amount of tokens of token type `id` owned by `account`.
     *
     * Requirements:
     *
     * - `account` cannot be the zero address.
     */
    function balanceOf(address account, uint256 id) external view returns (uint256);

    /**
     * @dev xref:ROOT:erc1155.adoc#batch-operations[Batched] version of {balanceOf}.
     *
     * Requirements:
     *
     * - `accounts` and `ids` must have the same length.
     */
    function balanceOfBatch(address[] calldata accounts, uint256[] calldata ids)
        external
        view
        returns (uint256[] memory);

    /**
     * @dev Grants or revokes permission to `operator` to transfer the caller's tokens, according to `approved`,
     *
     * Emits an {ApprovalForAll} event.
     *
     * Requirements:
     *
     * - `operator` cannot be the caller.
     */
    function setApprovalForAll(address operator, bool approved) external;

    /**
     * @dev Returns true if `operator` is approved to transfer ``account``'s tokens.
     *
     * See {setApprovalForAll}.
     */
    function isApprovedForAll(address account, address operator) external view returns (bool);

    /**
     * @dev Transfers `amount` tokens of token type `id` from `from` to `to`.
     *
     * Emits a {TransferSingle} event.
     *
     * Requirements:
     *
     * - `to` cannot be the zero address.
     * - If the caller is not `from`, it must have been approved to spend ``from``'s tokens via {setApprovalForAll}.
     * - `from` must have a balance of tokens of type `id` of at least `amount`.
     * - If `to` refers to a smart contract, it must implement {IERC1155Receiver-onERC1155Received} and return the
     * acceptance magic value.
     */
    function safeTransferFrom(
        address from,
        address to,
        uint256 id,
        uint256 amount,
        bytes calldata data
    ) external;

    /**
     * @dev xref:ROOT:erc1155.adoc#batch-operations[Batched] version of {safeTransferFrom}.
     *
     * Emits a {TransferBatch} event.
     *
     * Requirements:
     *
     * - `ids` and `amounts` must have the same length.
     * - If `to` refers to a smart contract, it must implement {IERC1155Receiver-onERC1155BatchReceived} and return the
     * acceptance magic value.
     */
    function safeBatchTransferFrom(
        address from,
        address to,
        uint256[] calldata ids,
        uint256[] calldata amounts,
        bytes calldata data
    ) external;
}
```

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;
import "./IERC1155.sol";
import "./IERC1155Receiver.sol";
import "./extensions/IERC1155MetadataURI.sol";
import "../erc721/IERC165.sol";
import "../erc721/Address.sol";
import "../erc721/Strings.sol";

contract ERC1155 is IERC165, IERC1155, IERC1155MetadataURI{

    using Address for address;

    using Strings for uint256;

    //代币种类id到账户account以及余额之间的映射关系
    mapping (uint256 => mapping (address => uint)) balances;

    //address到授权地址的授权映射关系
    mapping (address => mapping (address => bool)) operatorApprovals;


    function supportsInterface(bytes4 interfaceId) external pure override returns (bool){
        return interfaceId == type(IERC1155).interfaceId || interfaceId == type(IERC1155MetadataURI).interfaceId || interfaceId == type(IERC165).interfaceId;
    }

    //实现Metadata接口
    //bytes和string之间的相互转换    bytes(string)    string(bytes)
    function uri(uint256 id) public view virtual override returns (string memory){
        string memory baseURI = _baseURI();
        return bytes(baseURI).length > 0 ? string(abi.encodePacked(baseURI, id.toString())) : "";
    }

    function _baseURI() internal view virtual returns (string memory){
        return "";
    }

    /**
     * 查询持仓量。但是和ERC721不同的是不仅需要输入查询的账号，还需要币种编号
     */
    function balanceOf(address _account, uint256 _id) public view override returns (uint256){
        require(_account != address(0), "ERC1155:address zero is not a valid address");
        return balances[_id][_account];
    }

    /**
     * @dev xref:ROOT:erc1155.adoc#batch-operations[Batched] version of {balanceOf}.
     *
     * Requirements:
     *
     * - `accounts` and `ids` must have the same length.
     * 批量查询持仓量
     */
    function balanceOfBatch(address[] memory _accounts, uint256[] memory _ids)public view override returns (uint256[] memory){
        require(_accounts.length == _ids.length, "ERC1155: accounts and ids length mismatch");
        uint256[] memory batchBalances = new uint256[](_accounts.length);
        for (uint i = 0; i < _accounts.length; i++) {
            batchBalances[i] = balanceOf(_accounts[i], _ids[i]);
        }
        return batchBalances;
    }

    /**
     * @dev Grants or revokes permission to `operator` to transfer the caller's tokens, according to `approved`,
     *
     * Emits an {ApprovalForAll} event.
     *
     * Requirements:
     *
     * - `operator` cannot be the caller.
     * 批量授权，触发ApprovalForAll事件
     */
    function setApprovalForAll(address _operator, bool _approved) override public{
        address owner = msg.sender;
        require(owner != _operator, "ERC1155: setting approval status for self");
        operatorApprovals[owner][_operator] = _approved;
        emit ApprovalForAll(owner, _operator, _approved);
    }

    /**
     * @dev Returns true if `operator` is approved to transfer ``account``'s tokens.
     *
     * See {setApprovalForAll}.
     * 查询批量授权信息
     */
    function isApprovedForAll(address _account, address _operator) public view override returns (bool){
        return operatorApprovals[_account][_operator];
    }

    /**
     * @dev Transfers `amount` tokens of token type `id` from `from` to `to`.
     *
     * Emits a {TransferSingle} event.
     *
     * Requirements:
     *
     * - `to` cannot be the zero address.
     * - If the caller is not `from`, it must have been approved to spend ``from``'s tokens via {setApprovalForAll}.
     * - `from` must have a balance of tokens of type `id` of at least `amount`.
     * - If `to` refers to a smart contract, it must implement {IERC1155Receiver-onERC1155Received} and return the
     * acceptance magic value.
     * 单币种安全转账，触发TransferSingle事件。
     * 注意事项：
     * 1.to不可以是0地址
     * 2._from要么是转出方、要么是转出方授权的地址
     * 3._from必须币种编号_id有至少有_amount代币
     * 4._to如果是一个合约地址，则必须实现IERC1155Receiver-onERC1155Received
     */
    function safeTransferFrom(address _from, address _to, uint256 _id, uint256 _amount, bytes memory _data) override public{
        //必须是当前转账方调用或者授权给某个合约来进行转账
        address operator = msg.sender;
        require(_from == operator || isApprovedForAll(_from, operator), "ERC1155: caller is not token owner or approved");
        require(_to != address(0), "ERC1155: can not transfer to the zero address");
        uint256 fromBalance = balances[_id][_from];
        require(fromBalance >= _amount, "ERC1155: insufficient balance for transfer");
        //安全检查
        _safeTransferCheck(operator, _from, _to, _id, _amount, _data);
        balances[_id][_from] = fromBalance - _amount;
        balances[_id][_to] += _amount;
        emit TransferSingle(operator, _from, _to, _id, _amount);
    }

    function _safeTransferCheck(address _operator, address _from, address _to, uint256 _id, uint256 _amount, bytes memory _data) private{
        if(_to.isContract()){
            try IERC1155Receiver(_to).onERC1155Received(_operator, _from, _id, _amount, _data) returns (bytes4 result) {
                if(result != IERC1155Receiver.onERC1155Received.selector){
                    revert("ERC1155:ERC1155Receiver rejected");
                }
            } catch  {
                revert("ERC1155:receiver do not implement ERC1155Receiver");
            }
        }
    }

    /**
     * @dev xref:ROOT:erc1155.adoc#batch-operations[Batched] version of {safeTransferFrom}.
     *
     * Emits a {TransferBatch} event.
     *
     * Requirements:
     *
     * - `ids` and `amounts` must have the same length.
     * - If `to` refers to a smart contract, it must implement {IERC1155Receiver-onERC1155BatchReceived} and return the
     * acceptance magic value.
     * 多币种批量安全转账。触发TransferBatch事件。
     */
    function safeBatchTransferFrom(address _from, address _to, uint256[] memory _ids, uint256[] memory _amounts, bytes memory _data) override public{
        address operator = msg.sender;
        require(_from == operator || isApprovedForAll(_from, operator), "ERC1155: caller is not token owner nor approved");
        require(_ids.length == _amounts.length, "ERC1155: ids and amounts length mismatch");
        require(_to != address(0), "ERC1155: can not transfer to the zero address");
        //安全检查
        _safeBatchTransferCheck(operator, _from, _to, _ids, _amounts, _data);
        for (uint i = 0; i < _ids.length; i++) {
            uint256 id = _ids[i];
            uint256 amount = _amounts[i];
            uint256 fromBalance = balances[id][_from];
            require(fromBalance >= amount, "ERC1155: insufficient balance for transfer");
            balances[id][_from] = fromBalance - amount;
            balances[id][_to] += amount;
        }
        emit TransferBatch(operator, _from, _to, _ids, _amounts);
    }

    function _safeBatchTransferCheck(address _operator, address _from, address _to, uint256[] memory _ids, uint256[] memory _amounts, bytes memory _data) private{
        if(_to.isContract()){
           try IERC1155Receiver(_to).onERC1155BatchReceived(_operator, _from, _ids, _amounts, _data) returns (bytes4 result) {
            if(result != IERC1155Receiver.onERC1155BatchReceived.selector){
                revert("ERC1155: ERC1155Receiver rejected tokens");
            }
           } catch {
                revert("ERC1155: transfer to non-ERC1155Receiver implementer contract");
           }
        }
    }

    function _mint(address _to, uint256 _id, uint256 _amount, bytes memory _data) internal virtual{
        require(_to != address(0), "ERC1155: can not mint to zero address");
        address operator = msg.sender;
        _safeTransferCheck(operator, address(0), _to, _id, _amount, _data);
        balances[_id][_to] += _amount;
        emit TransferSingle(operator, address(0), _to, _id, _amount);
    }

    function _mintBatch(address _to, uint256[] memory _ids, uint256[] memory _amounts, bytes memory _data) internal virtual{
        require(_to != address(0), "ERC1155: can not mint to zero address");
        require(_ids.length == _amounts.length, "ERC1155: ids and amounts length mismatch");
        address operator = msg.sender;
        _safeBatchTransferCheck(operator, address(0), _to, _ids, _amounts, _data);
        for (uint i = 0; i < _ids.length; i++) {
            balances[_ids[i]][_to] += _amounts[i];
        }
        emit TransferBatch(operator, address(0), _to, _ids, _amounts);
    }

    function _burn(address _from, uint256 _id, uint256 _amount) internal virtual{
        require(_from != address(0), "ERC1155:can not burn from zero address");
        address operator = msg.sender;
        uint256 fromBalance = balances[_id][_from];
        require(fromBalance >= _amount, "ERC1155:burn amount exceeds balance");
        balances[_id][_from] = fromBalance - _amount;
        emit TransferSingle(operator, _from, address(0), _id, _amount);
    }

    function _burnBatch(address _from, uint256[] memory _ids, uint256[] memory _amounts) internal virtual{
        require(_from != address(0), "ERC1155:can not burn from zero address");
        require(_ids.length == _amounts.length, "ERC1155: ids and amounts length mismatch");
        address operator = msg.sender;
        for (uint i = 0; i < _ids.length; i++) {
            uint256 id = _ids[i];
            uint256 amount = _amounts[i];
            uint256 fromBalance = balances[id][_from];
            require(fromBalance >= amount, "ERC1155:batch burn amount exceeds balance");
            balances[id][_from] = fromBalance - amount;
        }
        emit TransferBatch(operator, _from, address(0), _ids, _amounts);
    }

}
```

