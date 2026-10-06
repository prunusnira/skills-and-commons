# 테스트 설계

## 무엇을 검증할지

- 대상의 공개 함수·타입·프로세스 경계에서 입력, 반환값, 상태 변화, 오류 코드, 예외 또는 외부에 전달된 결과를 확인한다. private 멤버나 호출 순서 자체는 계약일 때만 검증한다.
- 테스트 하나가 실패할 때 어떤 동작이 깨졌는지 이름과 assertion으로 알 수 있게 한다. 복잡한 셋업보다 작은 입력과 명시적인 기대값을 택한다.
- 빈 입력, 경계값, 잘못된 형식, 자원 부족처럼 실제 계약에 중요한 실패 사례를 고른다. 도달할 수 없는 가상의 경우까지 늘리지 않는다.
- 동일 동작의 여러 입력은 프로젝트 프레임워크가 제공하는 매개변수 테스트로 묶을 수 있다. 각 사례의 의도가 결과에서 구별되어야 한다.

## Assertion과 테스트 형태

- 프로젝트의 프레임워크를 따른다. GoogleTest의 `EXPECT_*`와 Catch2의 `CHECK`는 실패 후 검증을 이어가고, GoogleTest의 `ASSERT_*`와 Catch2의 `REQUIRE`는 그 지점에서 중단한다. 뒤의 코드가 선행 결과에 의존할 때만 중단형을 쓴다.
- 부동소수점 결과에는 도메인에 맞는 허용 오차를 정한다. 예외는 타입과 계약에 포함된 정보만 검증한다. 예외가 비활성화된 빌드에서는 해당 프로젝트의 오류 반환 계약을 따른다.
- 순수 로직은 좁은 단위 테스트로 검증한다. 파일 시스템, 프로세스, 네트워크와의 연동 자체가 계약이면 필요한 통합 테스트를 작성한다.
- 제품 소스 파일을 테스트에 직접 `#include`하거나 private 접근을 우회하지 않는다. 테스트 편의만을 위해 공개 API를 넓히지 않는다.

GoogleTest를 쓰는 프로젝트에서의 형태 예시다. `parse_port`와 반환형의 계약은 실제 프로젝트에서 확인해 바꾼다.

```cpp
#include "port_parser.h"
#include <gtest/gtest.h>
#include <string_view>

TEST(PortParserTest, OutOfRangeInputIsRejected) {
  // Given
  constexpr std::string_view input = "65536";

  // When
  const auto result = parse_port(input);

  // Then
  EXPECT_FALSE(result.has_value());
}
```

## 의존성과 상태

- 파일 시스템, 시계, 난수, 네트워크 같은 외부 경계는 결정적인 입력으로 제어한다. 작은 fake나 임시 파일이 충분하면 큰 mock 계층을 만들지 않는다.
- fixture는 테스트에 필요한 값만 준비한다. 생성한 파일·스레드·핸들·전역 설정은 테스트 종료 시 복구한다. 공유 mutable 상태나 실행 순서에 의존하지 않는다.
- 비동기 테스트는 관찰 가능한 완료 조건과 합리적인 제한 시간을 사용한다. 임의의 `sleep`으로 성공을 추정하지 않는다.
- 실제 장애가 재현되는 테스트라면 수정 전 실패와 수정 후 성공을 확인한다. 테스트만 추가할 때도 기대 동작을 위반하면 assertion이 실패할지 점검한다.

## 피할 것

- 실행만 하고 결과를 검증하지 않는 테스트, 과도한 mock, private 구현에 결합된 assertion, 설명 없는 큰 fixture.
- 비활성화된 테스트나 집중 실행 표시를 남기는 일. GoogleTest의 `DISABLED_`, Catch2의 숨김 태그 등은 의도와 기존 정책을 확인한다.
- 테스트 통과를 위해 제품 동작을 요청 범위 밖에서 바꾸는 일.

참고: [GoogleTest Primer](https://google.github.io/googletest/primer.html), [GoogleTest Assertions](https://google.github.io/googletest/reference/assertions.html), [Catch2 Assertions](https://github.com/catchorg/Catch2/blob/devel/docs/assertions.md).
