---
name: nunit-test
description: "Unity Test Framework(NUnit)로 EditMode/PlayMode 테스트를 작성/수정/리뷰할 때 따르는 기준. Given-When-Then 구조, 관찰 가능한 동작 검증, 외부 경계만 Mock 하는 원칙을 강제한다. /impl 종료 후 자동 테스트 작성 시 이 스킬을 참고한다."
---

# NUnit 테스트 작성 Skill

## 목적

이 문서는 AI 에이전트가 이 저장소에서 Unity Test Framework(NUnit) 기반 테스트를 작성, 수정, 리뷰할 때 따라야 하는 기준을 정의한다.

이 프로젝트는 다음 환경을 사용한다.

- Unity 6 (6000.3.11f1)
- C#
- Unity Test Framework (NUnit 3)
- EditMode / PlayMode 테스트

목표는 테스트 개수를 늘리는 것이 아니다.
목표는 **읽기 쉽고, 안정적이며, 유지보수 가능한 테스트**를 작성하는 것이다.

---

## 핵심 원칙

### 1. Given-When-Then 구조를 반드시 유지한다

모든 테스트는 기본적으로 다음 구조를 따른다.

```csharp
[Test]
public void 기대하는동작_설명()
{
    // Given
    // 초기 상태, fixture, 테스트 대상 설정

    // When
    // 테스트 대상 메서드를 호출하거나 상태 변화를 발생시킨다

    // Then
    // 관찰 가능한 결과를 검증한다
}
```

테스트가 매우 짧더라도 Given-When-Then 흐름이 드러나야 한다. 구역이 명확히 구분되도록 빈 줄로 분리한다.

좋은 예:

```csharp
[Test]
public void PlayOption_GaugeType이브로_설정하면_게이지가브로_적용된다()
{
    // Given
    var option = new PlayOption();

    // When
    option.GaugeType = GaugeType.Bro;

    // Then
    Assert.That(option.GaugeType, Is.EqualTo(GaugeType.Bro));
}
```

나쁜 예:

```csharp
[Test]
public void SetGauge_Works()
{
    var option = new PlayOption();
    option.SetGauge(GaugeType.Bro);
    Assert.IsTrue(option.IsBro);
}
```

테스트 이름은 내부 구현이 아니라 **사용자 관점의 동작**을 설명해야 한다.

NUnit에서는 `MethodName_Scenario_ExpectedBehavior` 형태도 허용하지만, 한국어 동작 설명이 더 읽기 쉬우면 그것을 우선한다. `it(...)`/`test(...)`의 대응물이 `[Test] public void ...` 임을 잊지 않는다.

---

## 테스트 철학

테스트는 구현 세부사항이 아니라 **관찰 가능한 동작**을 검증해야 한다.

우선적으로 검증해야 하는 것:

- public 메서드의 반환값
- public 프로퍼티 / 필드의 변화
- 이벤트 구독자에게 전달되는 인자
- 인스펙터 노출 필드(`[SerializeField]`)의 직렬화 결과가 필요한 경우
- 씬 전환 요청, RPC 호출 방향(ServerRpc/ClientRpc)
- 로딩/성공/실패 상태 플래그
- 예외 발생
- 비즈니스 의미가 있는 상태 변화

되도록 피해야 하는 것:

- private 필드 / private 메서드
- 내부 구현의 분기 경로 자체
- 내부 리스트/딕셔너리의 정확한 순서(의미가 없다면)
- 유니티 내부 컴포넌트의 정확한 렌더링 결과
- 과도한 GameObject 계층 구조 검증
- 큰 객체 전체의 deep equality (필드 하나만 검증 가능하면 그것만)
- 구현 편의를 위한 과도한 mock

---

## NUnit + Unity Test Framework 기본

### 어셈블리 분리

- EditMode 테스트는 `Assets/Tests/EditMode/` 아래에, PlayMode 테스트는 `Assets/Tests/PlayMode/` 아래에 둔다.
- 각 폴더에는 `*.asmdef`가 있어야 한다.
  - EditMode: `nunit.framework`, `UnityEngine.TestFramework`, `UnityEditor.TestFramework` 참조, Editor 플랫폼 포함
  - PlayMode: `nunit.framework`, `UnityEngine.TestFramework` 참조, 필요 플랫폼 포함
- 테스트 대상 어셈블리(BMSPlayer 등)를 참조한다.

### TestFixture / Test

```csharp
using NUnit.Framework;
using UnityEngine;

[TestFixture]
public class PlayOptionTests
{
    private PlayOption option;

    [SetUp]
    public void SetUp()
    {
        option = ScriptableObject.CreateInstance<PlayOption>();
    }

    [TearDown]
    public void TearDown()
    {
        Object.DestroyImmediate(option);
    }

    [Test]
    public void 기본_게이지타입은_Assist_이다()
    {
        // Given
        // (SetUp에서 초기화)

        // When
        var actual = option.GaugeType;

        // Then
        Assert.That(actual, Is.EqualTo(GaugeType.Assist));
    }
}
```

### 어서션

- `Assert.That(actual, Is.EqualTo(expected))` 제약 모델을 기본으로 사용한다.
- 컬렉션은 `Is.EquivalentTo`, `Contains`, `Has.Count` 등을 사용.
- 예외는 `Assert.Throws<TException>(() => ...)` 로 검증.
- 실패 메시지가 필요한 경우 `Assert.That(actual, Is.EqualTo(expected), "이유")` 형태로 이유를 남긴다.
- `Assert.Pass()`, `Assert.Fail()` 은 의미가 명확할 때만.

---

## Unity 특화 규칙

### EditMode vs PlayMode

- 로직(순수 C# 클래스, 정적 메서드, BMS 파싱, 패턴 매칭)은 EditMode로 검증한다. 빠르고 안정적이다.
- `MonoBehaviour` 생명주기, `Update`, 코루틴, `OnEnable`/`OnDisable` 구독, 씬 전환, NetworkBehaviour 스폰 등은 PlayMode로 검증한다.
- PlayMode 테스트는 느리고 불안정하므로 꼭 필요한 경우에만 작성한다.

### GameObject / ScriptableObject 정리

- 테스트에서 생성한 `UnityEngine.Object`는 반드시 `TearDown`에서 `Object.DestroyImmediate`로 해제한다. EditMode에서는 `Destroy`가 즉시 반영되지 않는다.
- PlayMode에서는 `[UnityTearDown]`에서 `Object.Destroy`를 사용한다.
- 씬에 잔여 GameObject가 남으면 테스트 간 간섭이 발생한다.

```csharp
[TearDown]
public void TearDown()
{
    if (go != null)
    {
        Object.DestroyImmediate(go);
    }
}
```

### 씬 의존성

- 테스트가 씬 로드에 의존하면 안 된다. 의존한다면 `EditorSceneManager.LoadScene`을 셋업에서 명시적으로 호출하고 티어다운에서 언로드한다.
- 씬 전환 로직 자체를 테스트할 때는 전환 **요청**(함수 호출, `Const.NextScene` 값 등)만 검증하고 실제 빌드된 씬 로드는 검증하지 않는다.

### Random / 시간 의존

- `Random.value`, `DateTime.Now`, `Time.realtimeSinceStartup` 에 의존하는 로직은 테스트 전에 시드를 고정하거나 의존성을 주입 가능하게 리팩터링한다.
- 단, 프로덕션 코드 수정은 사용자가 명시적으로 요청한 경우에만 수행한다.

### Input / 네트워크

- Unity Input System 직접 호출은 PlayMode에서만 의미가 있다.
- NetworkBehaviour는 실제 NGO 런타임 없이 검증하기 어렵다. 대신 순수 로직 부분(ServerRpc의 인자 검증, ClientRpc의 타겟 계산, 상태 동기화 알고리즘 등)을 분리해서 EditMode로 검증한다.

---

## Mock 규칙

Unity 프로젝트에서 Mock은 제한적으로 사용한다. 일반적으로 인터페이스 기반보다 concrete class + `Moq`/`NSubstitute` 보다 **작은 테스트 전용 stub/fake**를 직접 작성하는 경우가 많다.

Mock해도 되는 대상:

- 파일 시스템 접근(`db.sqlite`, BMS 파일 로드)
- 네트워크 요청 / NGO 통신
- FMOD / AVProVideo 등 외부 엔진
- Unity Editor API(테스트에서만)
- 날짜와 시간
- PlayerPrefs / 정적 상태

되도록 mock하지 말아야 하는 대상:

- 테스트 대상 컴포넌트 자체
- 단순한 자식 컴포넌트
- 내부 유틸 함수
- 어서션을 쉽게 만들기 위해 프로덕션 타입을 통째로 mock

mock이 너무 많다면 실제 동작이 아니라 mock 설정을 테스트하고 있을 가능성이 높다. 이 경우 테스트의 가치를 다시 평가한다.

외부 API를 mock할 때는 테스트 파일 안에 작은 stub 클래스를 만들어 사용하는 것을 우선하고, 별도 파일로 분리하는 것은 두 개 이상의 테스트 클래스에서 재사용될 때로 한정한다.

---

## Fixture 규칙

fixture는 작고 명확하게 작성한다. 테스트를 읽는 사람이 어떤 데이터가 중요한지 바로 이해할 수 있어야 한다.

좋은 예:

```csharp
private static BMSObject CreateSimpleNote(int channel, double time)
{
    return new BMSObject
    {
        channel = channel,
        time = time,
        type = NoteType.Normal
    };
}
```

피해야 하는 예:

- 테스트 파일 외부의 거대한 fixture 파일에서 데이터를 가져오기
- 한 테스트에서 사용하지 않을 필드까지 채운 fixture
- 매직 넘버가 설명 없이 들어간 fixture

`[TestCase]`, `[TestCaseSource]` 로 데이터를 주입할 때도 각 케이스가 무엇을 검증하는지 이름이나 주석으로 의도가 드러나야 한다.

```csharp
[TestCase(0,    "빈 입력은 빈 리스트를 반환한다")]
[TestCase(1,    "단일 노트는 길이 1의 결과를 반환한다")]
[TestCase(1000, "대량 노트도 정상 파싱된다")]
public void ParseNotes_시나리오별_동작(int count, string description)
{
    // Given
    var lines = BuildLines(count);

    // When
    var result = BMSParser.ParseNotes(lines);

    // Then
    Assert.That(result.Count, Is.EqualTo(count), description);
}
```

---

## 데이터 기반 테스트

- 단순 값 조합은 `[TestCase]`.
- 프로그래밍 생성 데이터는 `[TestCaseSource]`.
- `[Values]`, `[ValueSource]`는 파라미터 조합이 의미 있을 때만.
- `[Random]`, `[Range]`는 판정 윈도우, 게이지 임계값 등 수치 범위 테스트에 적합. 단, 재현성을 위해 시드를 고정할 수 있어야 한다.
- `[Combinatorial]`/`[Pairwise]`는 조합이 폭발하지 않는 선에서만.

---

## 비동기 / 코루틴 테스트

- `IEnumerator` 기반 코루틴은 PlayMode에서 `[UnityTest]`로 검증한다.

```csharp
[UnityTest]
public IEnumerator 로딩완료후_씬전환이_요청된다()
{
    // Given
    var controller = CreateController();

    // When
    yield return controller.LoadCoroutine();

    // Then
    Assert.That(Const.NextScene, Is.EqualTo(Scenes.PlayScreenSingle));
}
```

- 임의의 `yield return new WaitForSeconds(...)` 로 기다리지 않는다. 코루틴이 끝났는지 직접 기다리거나 상태 플래그를 폴링한다.
- `async`/`await`가 필요한 로직은 UniTask 등을 쓰지 않는 한 프로젝트에서는 드물다. Unity Test Framework는 기본적으로 `IEnumerator` 패러다임을 따른다.

---

## 에러 케이스 규칙

중요한 실패 케이스를 반드시 고려한다.

예시:

- null 입력 / 빈 입력
- 잘못된 포맷(BMS 파싱, JSON 직렬화)
- 인덱스 범위 초과
- 파일이 존재하지 않는 경우(MusicFileChecker)
- 선택적 필드 누락
- 경계값(판정 윈도우 양끝, 게이지 0% / 100%)
- 동시 호출 / 중복 호출
- 네트워크 단절 상황(ServerRpc 수신 불가 등)

happy path만 테스트하지 않는다. 다만 "발생 가능하지 않은" 예외까지 방어 코드와 테스트를 늘리지는 않는다.

---

## 테스트 격리 규칙

각 테스트는 서로 독립적이어야 한다.

테스트 사이에 새면 안 되는 상태:

- static 필드 (`NetworkStartControl.Instance`, `MusicListConst.SelectedItem` 등)
- PlayerPrefs
- ScriptableObject 인스턴스
- GameObject 씬 잔여물
- 파일 시스템(db.sqlite 쓰기 등)
- mock 호출 횟수 / 구현

필요하면 `[SetUp]`/`[TearDown]`에서 정리하고, static 상태는 `[OneTimeSetUp]`/`[OneTimeTearDown]` 또는 별도 리셋 함수로 원복한다.

```csharp
[TearDown]
public void TearDown()
{
    typeof(NetworkStartControl)
        .GetField("Instance", BindingFlags.Static | BindingFlags.NonPublic)
        ?.SetValue(null, null);
}
```

단, 프로덕션 코드에 테스트 전용 리셋 함수를 추가하는 것은 최소화한다.

---

## 테스트 파일 구조

- 테스트 대상 클래스와 1:1 대응되는 테스트 클래스를 작성한다.
  - `BMSAnalyzer.cs` → `BMSAnalyzerTests.cs`
  - `PlayOption.cs` → `PlayOptionTests.cs`
- partial class인 경우 한 테스트 파일로 묶거나, 분량이 크면 대상 partial마다 테스트 파일을 분리한다. `GamePlayController.Single.cs` → `GamePlayControllerSingleTests.cs`.
- 도메인 유틸 함수는 같은 폴더 구조를 `Tests/EditMode/{도메인}/` 아래에 재현한다.

```
Assets/Tests/
  EditMode/
    PlayOptionTests.cs
    BMSAnalyzerTests.cs
    MusicFileCheckerTests.cs
  PlayMode/
    NetworkStartControlTests.cs
```

---

## Snapshot / 직렬화 결과 검증

큰 객체 전체를 문자열로 비교하는 것은 피한다. 작고 의미 있는 필드만 검증한다.

나쁜 예:

```csharp
Assert.That(JsonUtility.ToJson(record), Is.EqualTo(expectedJson));
```

좋은 예:

```csharp
Assert.That(record.Title, Is.EqualTo("테스트 곡"));
Assert.That(record.ClearStatus, Is.EqualTo(ClearStatus.HardClear));
Assert.That(record.Rating, Is.GreaterThan(0));
```

---

## AI 에이전트가 테스트 작성 전에 확인해야 할 것

테스트를 작성하거나 수정하기 전에 반드시 다음을 확인한다.

1. 테스트 대상 클래스/함수와 public input/output
2. 사용자/시스템에게 관찰 가능한 동작
3. 외부 의존성(DB, 파일, 네트워크, FMOD)
4. 주변 테스트 파일의 스타일(있다면 따른다)
5. 프로젝트의 asmdef 설정(EditMode/PlayMode 어셈블리)
6. 공용 fixture / 헬퍼가 이미 있는지
7. 판정 타이밍, 게이지 계산 등 밸런스에 영향을 주는 값인지

기존 테스트 유틸이나 fixture가 있으면 새로 만들지 말고 재사용한다.

---

## AI 에이전트가 하면 안 되는 것

다음 행동은 하지 않는다.

- 테스트를 쉽게 만들기 위해 프로덕션 코드의 캡슐화를 크게 깨기
- 단순히 인스턴스 생성만 확인하는 의미 없는 테스트 추가
- 과도하게 mock/stub 만들기
- private 필드를 리플렉션으로 강제로 열어 테스트
- GameObject 계층이나 컴포넌트 구조에 강하게 의존하는 어서션
- 임의의 `yield return new WaitForSeconds(2f)` 로 대기
- 큰 객체 전체를 deep equality 검증
- `Assert.Ignore`, `[Ignore]` 남기기
- `[Explicit]` 남발하기
- 빌드/컴파일 에러가 있는 테스트를 커밋 대상에 두기

---

## 테스트 작성이 어려운 경우

테스트 작성이 어렵다면 무리하게 mock을 늘리지 않는다. 먼저 다음을 사용자에게 설명한다.

1. 테스트가 어려운 이유
2. 어떤 의존성 때문에 문제가 생기는지
3. EditMode / PlayMode 각각에서 검증 가능한 범위
4. 프로덕션 코드를 개선한다면 어떤 구조가 좋을지(분리 가능한 순수 로직 식별)

단, 프로덕션 코드 수정은 사용자가 명시적으로 요청한 경우에만 수행한다.

---

## 완료 전 체크리스트

작업을 마치기 전에 다음을 확인한다.

- [ ] Given-When-Then 구조를 따른다
- [ ] 테스트 이름이 동작을 설명한다
- [ ] 관찰 가능한 결과만 검증한다
- [ ] mock은 외부 경계에만 제한적으로 사용한다
- [ ] 중요한 에러 케이스를 포함한다
- [ ] `[SetUp]`/`[TearDown]`에서 상태가 격리된다
- [ ] 생성한 Unity Object를 모두 해제한다
- [ ] static 상태를 오염시키지 않거나 원복한다
- [ ] `[Ignore]`, `[Explicit]`, 임의 대기가 없다
- [ ] 컴파일 에러가 없다
- [ ] 테스트를 읽는 사람이 과도한 셋업 없이 이해할 수 있다
- [ ] 실제 동작이 깨지면 테스트도 실패한다

---

## AI 에이전트의 최종 응답 형식

테스트를 작성하거나 수정한 뒤에는 다음 내용을 요약한다.

1. 어떤 동작을 테스트했는지
2. 어떤 fixture/stub을 추가했고 왜 필요한지
3. 어떤 edge case를 포함했는지
4. EditMode로 검증하기 어려워 PlayMode(또는 수동/E2E)가 필요한 영역이 있는지
