# 스택

## 언어 & 툴체인

- **Solidity** 0.8.13 — 게임 컨트랙트가 쓰는 버전과 동일하게 핀. overflow/check 시맨틱이 완전히 같도록 보장하기 위한 선택
- **Foundry** — 빌드, 테스트, 가스 프로파일링 (`forge test`, `forge snapshot`)
- **Solmate** — 유틸 라이브러리 (`SafeCastLib` 로 panic 없이 narrowing cast)

## 수학

- **SignedWadMath** — 18자리 고정소수점 산술 (`wadMul`, `wadDiv`, `wadExp`, `wadLn`). 가격 수식이 연속 지수 감쇠를 사용하기 때문에 정수 산술로는 몇 턴만 지나도 발산하는 구조다.

## 코드 레이아웃 (정본)

```
src/
├── Monaco.sol             — 게임 컨트랙트 ("스펙")
├── utils/SignedWadMath.sol — 고정소수점 헬퍼
└── cars/
    ├── Car.sol            — 추상 베이스. takeYourTurn 인터페이스 정의
    └── ExampleCar.sol     — 레퍼런스 구현. 상대로 두기에도 활용 가능

script/
└── Deploy*.s.sol          — Foundry 배포 스크립트

test/
└── Monaco.t.sol           — Foundry 테스트. 간단한 시뮬레이터 역할 겸용
```

## Foundry 설정에서 핵심 부분

```toml
[profile.default]
bytecode_hash = "none"      # 결정적 bytecode (채점 환경에서 의미 있음)
optimizer_runs = 1000000    # 최대 최적화. 배포 가스보다 런타임 가스가 중요한 도메인
solc = "0.8.13"             # 핀

[profile.intense]
fuzz_runs = 10000           # 엣지 케이스용 property-based 테스트
```

## 이 스택이 이 문제 영역에서 자주 선택되는 이유

- **Foundry vs Hardhat:** 테스트 루프가 10~100배 빠름. 봇 한 번 고칠 때마다 시뮬레이션 게임을 여러 번 돌려야 하므로 반복 속도가 의사결정 변수가 된다.
- **Solmate vs OpenZeppelin:** Solmate 쪽 라이브러리가 더 작고 가스 효율적. 특히 fixed-point math primitive 는 OZ 에 직접적인 대응이 없다.
- **컨트랙트 repo 안에 프론트엔드/오프체인 오케스트레이션을 두지 않는 분리:** 시뮬레이터는 별도 프로젝트(Python 또는 TypeScript 권장)로 두고 Solidity 룰을 *미러링*한다. 온체인 스펙은 단순하게, 오프체인 최적화는 풍부하게 — 이 도메인에서 통념적으로 권장되는 분리다.

## 배포 노트

CTF 본 게임에서는 주최자가 게임 컨트랙트를 배포하고 각 플레이어 car 를 등록한다. 로컬 self-play 의 경우 동봉된 배포 스크립트가 하나의 Monaco 인스턴스와 세 대의 car 를 연결한다. 봇은 런타임에 오프체인 인프라가 필요하지 않은 순수 온체인 reactive 컴포넌트다.
