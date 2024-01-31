# 1. Ownable

---

## 개념

Ownable 컨트랙트는 스마트 컨트랙트 개발에서 널리 사용되는 디자인 패턴 중 하나로 특정 권한을 컨트랙트의 소유자(Owner)에게만 부여하는 것을 목적으로 한다.

예를들어 컨트랙트 업그레이드, 수수료율, 토큰 발행 한도와 같은 중요한 변수 설정이 있을 수 있다.
이를 통해 컨트랙트는 중요한 변경사항을 소유자의 승인 하에만 진행할 수 있게 하여, 무분별한 변경을 방지하고 보안을 강화한다.

오픈제플린 오너블 컨트랙트는 하나의 소유자 계정만 존재한다. 이 소유자 계정은 공개적으로 확인할 수 있으며, 이는 컨트랙트의 운영에 대한 투명성과 책임을 제공한다.

## 주요 기능 및 특징

- 소유자 설정:
  배포 시 생성자 내부에서 컨트랙트를 생성한 주소를 디폴트 소유자로 설정한다.
  ```
  constructor() {
      _transferOwnership(_msgSender());
  }
  ```
- 소유권 이전 transferOwnership() :
  소유자는 다른 주소에 소유권을 이전할 수 있다.
  ```
   function transferOwnership(address newOwner) public virtual onlyOwner {
       require(newOwner != address(0), "Ownable: new owner is the zero address");
       _transferOwnership(newOwner);
   }
  ```
- 제한된 접근 onlyOwner :
  특정 함수는 오직 소유자에 의해서만 호출될 수 있다. onlyOwner modifier를 사용하여 구현되며, 이 modifier가 있는 함수는 소유자가 아닌 다른 주소에서 호출을 막는다.

  ```
  modifier onlyOwner() {
       _checkOwner();
       _;
  }

  function _checkOwner() internal view virtual {
      require(owner() == _msgSender(), "Ownable: caller is not the owner");
  }
  ```

- 소유권 포기 renounceOwnership() :
  오너는 renounceOwnership()을 통해 오너쉽을 영구히 포기 할 수 있다. 오너 주소를 address(0)으로 설정
  하기 때문에 포기된 오너쉽은 되살릴 수 없다.
  ```
  function renounceOwnership() public virtual onlyOwner {
      _transferOwnership(address(0));
  }
  ```
- 소유권 이전 이벤트 OwnershipTransferred:
  오너쉽이 바뀔때마다 OwnershipTransferred 이벤트가 emit된다.
  ```
  event OwnershipTransferred(address indexed previousOwner, address indexed newOwner);
  ```

## 주의사항

- Ownable 컨트랙트는 특정 주소에 광범위한 권한을 부여하기 때문에, 소유자의 의도나 행동에 따라 시스템이 크게 영향을 받을 수 있다.
- 소유자의 권한이 강력하므로, 소유자 변경이나 중요한 행동은 투명하게 관리되어야 한다.
- 실수로 소유권 이전을 진행하거나 원하지 않는 계정에 이전될 경우 되돌릴 수 없으므로 소유권 이전 작업에 각별한 주의가 필요하다.

## Ownable 테스트

```
npx hardhat test test/ownable.ts
```

# 2. Ownable 2step

---

## 개념

"Ownable 2 step" 또는 "Two-Step Ownable"은 표준 Ownable 컨트랙트의 확장된 형태로, 소유권 이전 과정에 추가적인 안전장치를 더하는 패턴이다. 기본적인 Ownable 디자인에서는 소유자가 다른 주소로 소유권을 바로 이전할 수 있었지만, Two-Step Ownable 패턴에서는 이 과정을 소유권 이전 요청과 수락 두 단계로 나누어 더욱 안전하게 소유권을 이전한다.

Ownable 2 step는 새 소유자가 명시적으로 **소유권을 수락해야만 이전이 완료**되기 때문에 실수로 잘못된 주소로 소유권을 이전하는 것을 방지한다. 또한 악의적인 공격자가 소유자의 계정을 해킹하여 소유권을 빼앗겨도 소유권 이전이 두 단계를 거쳐야 하기 때문에, 추가적인 보안 장벽이 생기는 장점이 있다.

Ownable 2 step은 대규모 자금을 관리하는 금융 서비스나 중요한 기능을 수행하는 시스템에서 사용하기에 적합하다.당연히 Ownable 2 step를 사용하면 관리 복잡성과 가스비가 이중으로 늘어난다.

## Two-Step Ownable 확장 기능

- 소유권 이전 요청:
  현재 소유자가 새로운 소유자로 지정하고자 하는 주소에 대해 '이전 요청'을 하면 바로 소유권이 이전되지 않고\_pendingOwner 변수에 새로운 소유자 주소가 담긴다.

  ```
  function transferOwnership(address newOwner) public virtual override onlyOwner {
      _pendingOwner = newOwner;
      emit OwnershipTransferStarted(owner(), newOwner);
  }
  ```

- 소유권 수락:
  새 소유자가 소유권 이전을 수락하면, 소유권이 이전된다.
  ```
  function acceptOwnership() external {
      address sender = _msgSender();
      require(pendingOwner() == sender, "Ownable2Step: caller is not the new owner");
      _transferOwnership(sender);
  }
  ```

## Ownable 2 step 테스트

```
npx hardhat test test/ownable2Step.ts
```

# 3. Access controll

---

## 개념

AccessControl 컨트랙트는 스마트 컨트랙트에서 세분화된 권한 관리를 가능하게 하는 중요한 디자인 패턴이다. 복잡한 권한 구조를 필요로 하는 응용 프로그램이나, 다양한 역할과 권한이 명확히 구분되어야 하는 시스템에서 특히 유용하다.

Ownable 컨트랙트가 단일 관리자(오너)에 초점을 맞춘 반면, AccessControl은 역할(Role)-기반의 접근 제어를 통해 다양한 역할에 따른 권한을 세분화하고 관리할 수 있도록 설계되었다. 이를 통해 컨트랙트의 각 기능은 특정 역할을 가진 주소에 의해서만 실행될 수 있게 된다.

예)

- 토큰 관리: 특정 토큰의 발행과 소각 등의 작업을 역할별로 구분하여 관리
- 다중 사용자 플랫폼: 사용자, 관리자, 슈퍼 관리자 등 다양한 역할을 구분하여 각각의 권한을 설정
- 금융 시스템: 자금 이체, 승인, 감사 등 금융 관련 작업을 역할별로 관리
- 게임 또는 NFT 시스템: 게임 내 아이템의 생성, 소멸, 이전 등을 역할별로 제어

> AccessControl은 Ownable, Multi-Signature 와 함께 사용할 수 있다.

## 주요 기능 및 특징

- 역할 기반 권한 할당:
  AccessControl은 다양한 역할에 대해 권한을 할당하고 관리할 수 있게 해준다. 이를 통해 컨트랙트 내에서 세분화된 권한 관리가 가능하다.

- 동적 권한 변경:
  시스템의 운영 중에도 역할과 권한을 동적으로 변경할 수 있다. 이는 컨트랙트의 유연성과 확장성을 높여준다.

- 멀티시그(Multi-Signature) 지원:
  특정 역할에 대한 권한 부여가 여러 주소의 승인을 필요로 할 수 있다. 이는 보안을 강화하는 중요한 요소다.

- 이벤트 로깅:
  권한 부여 및 취소, 역할 변경 등의 작업은 이벤트로 로깅되어 추적 가능성과 투명성을 제공한다.

## 사용 예시

AccessControl을 상속받은 컨트랙트 생성자에서 디폴트 어드민 롤을 부여한다.

```
_grantRole(DEFAULT_ADMIN_ROLE, _msgSender());
```

grantRole() 함수와 revokeRole() 함수로 특정 계정 주소에 롤을 부여하고 취소한다.

```
    /**
     * @dev Grants `role` to `account`.
     *
     * If `account` had not been already granted `role`, emits a {RoleGranted}
     * event.
     *
     * Requirements:
     *
     * - the caller must have ``role``'s admin role.
     *
     * May emit a {RoleGranted} event.
     */
    function grantRole(bytes32 role, address account) public virtual override onlyRole(getRoleAdmin(role)) {
        _grantRole(role, account);
    }

    /**
     * @dev Revokes `role` from `account`.
     *
     * If `account` had been granted `role`, emits a {RoleRevoked} event.
     *
     * Requirements:
     *
     * - the caller must have ``role``'s admin role.
     *
     * May emit a {RoleRevoked} event.
     */
    function revokeRole(bytes32 role, address account) public virtual override onlyRole(getRoleAdmin(role)) {
        _revokeRole(role, account);
    }
```

> 위 두 함수는 onlyRole 모디파이어에 의해 DEFAULT_ADMIN_ROLE 역할을 가진 주소만이 해당 함수를 호출할 수 있도록 제한된다.

롤은 특정 롤을 의미하는 문자를 해시하여 바이트32 타입으로 만든다.

```
bytes32 public constant MY_ROLE = keccak256("MY_ROLE");
```

ethers 에서는 다음과 같이 id()를 통해 만들 수 있다.

```
 const NEW_ROLE = ethers.utils.id("NEW_ROLE");
```

롤은 \_roles 맵핑에 의해 관리된다.

```
struct RoleData {
    mapping(address => bool) members;
    bytes32 adminRole;
}
mapping(bytes32 => RoleData) private _roles;
```

\_roles[역할키값]으로 RoleData 구조체에 접근할 수 있다.

```
_roles[role].members[account]
_roles[role].adminRole;
```

특정 역할을 관리하는 관리자를 지정할때는 \_setRoleAdmin()을 이용한다.

```
    /**
     * @dev Sets `adminRole` as ``role``'s admin role.
     *
     * Emits a {RoleAdminChanged} event.
     */
    function _setRoleAdmin(bytes32 role, bytes32 adminRole) internal virtual {
        bytes32 previousAdminRole = getRoleAdmin(role);
        _roles[role].adminRole = adminRole;
        emit RoleAdminChanged(role, previousAdminRole, adminRole);
    }
```

## AccessControll 테스트

```
npx hardhat test test/accessControlTest.ts
```
