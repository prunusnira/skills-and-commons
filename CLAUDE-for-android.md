# CLAUDE.md - Android 프로젝트 개발 규칙

## 작업 디렉터리 원칙

* 별도의 Git worktree를 생성하거나 다른 worktree로 전환하지 않는다.
* 사용자가 현재 열어 둔 작업 디렉터리에서 직접 작업하여 로컬 변경사항을 바로 확인할 수 있게 한다.
* 현재 작업 디렉터리에 있는 기존 로컬 변경사항은 유지하며, 요청과 무관한 변경을 수정하거나 되돌리지 않는다.

## 1. 기본 응답 규칙

* 모든 응답과 설명은 한국어로 작성한다.
* 코드, 클래스명, 함수명, 라이브러리명 등 기술 식별자는 원문을 유지한다.
* 변경 사항을 설명할 때는 무엇을 변경했는지보다 변경 이유와 영향을 우선한다.
* 확실하지 않은 내용은 추측하지 말고 확인이 필요하다고 명시한다.
* 불필요한 요약, 인사말, 마무리 문장을 작성하지 않는다.
* 이모지와 장식용 특수문자를 사용하지 않는다.
* 마크다운 테이블은 비교 항목이 많아 표가 명확한 경우에만 사용한다.
* 사용자가 요청하지 않은 범위까지 코드를 수정하거나 구조를 재설계하지 않는다.

---

## 2. 작업 시작 전 확인

코드를 수정하기 전에 다음 항목을 확인한다.

1. 프로젝트의 기존 디렉터리 구조
2. `build.gradle.kts` 또는 `build.gradle`
3. `gradle/libs.versions.toml`
4. `settings.gradle.kts`
5. 사용 중인 Android Gradle Plugin과 Kotlin 버전
6. `minSdk`, `targetSdk`, `compileSdk`
7. Jetpack Compose 또는 XML View 사용 여부
8. 프로젝트의 아키텍처와 상태 관리 방식
9. DI, 네트워크, 데이터베이스 라이브러리
10. 테스트 프레임워크와 코드 스타일

기존 구현 방식이 이 문서의 일반적인 권장 방식과 다르면 기존 프로젝트의 일관성을 우선한다.

프로젝트 설정을 확인하지 않은 상태에서 라이브러리 버전이나 API 사용 가능 여부를 단정하지 않는다.

---

## 3. 기본 기술 방향

별도 지시가 없고 신규 프로젝트인 경우 다음 구성을 우선 고려한다.

* 언어: Kotlin
* UI: Jetpack Compose
* 비동기 처리: Kotlin Coroutines
* 상태 스트림: Flow, StateFlow
* 아키텍처: UI, Domain, Data 계층 분리
* 화면 상태 관리: ViewModel
* 의존성 주입: Hilt
* 네트워크: Retrofit 또는 Ktor
* JSON 직렬화: Kotlinx Serialization 또는 프로젝트 기존 방식
* 로컬 데이터베이스: Room
* 설정 저장: DataStore
* 백그라운드 작업: WorkManager
* 화면 이동: Navigation Compose 또는 프로젝트 기존 Navigation 방식

기존 프로젝트에서는 새로운 기술을 임의로 도입하지 않는다.

---

## 4. 코드 작성 원칙

### 기본 원칙

* 변경 범위를 요청된 기능에 한정한다.
* 미래 요구사항을 예상한 추상화를 추가하지 않는다.
* 중복 제거보다 코드의 명확성을 우선한다.
* 한 번만 사용되는 단순 로직을 무리하게 함수나 클래스로 분리하지 않는다.
* 의미 없는 인터페이스, 래퍼, 추상 클래스 계층을 만들지 않는다.
* 사용하지 않는 코드, import, 변수, 리소스를 남기지 않는다.
* 기존 코드 스타일과 네이밍 규칙을 따른다.
* 파일 전체를 재작성하기보다 필요한 부분만 수정한다.
* 코드 포맷 변경만 발생하는 불필요한 diff를 만들지 않는다.
* 기존 공개 API를 변경할 때는 호출부와 영향 범위를 함께 확인한다.

### 주석

* 코드만으로 의도가 명확하면 주석을 작성하지 않는다.
* 비정상적인 처리, 플랫폼 제약, 우회 구현 등 WHY가 명확하지 않은 경우에만 작성한다.
* 코드 내용을 그대로 설명하는 주석은 작성하지 않는다.
* 오래된 주석과 실제 구현이 충돌하면 주석을 수정하거나 제거한다.
* TODO를 추가할 때는 이유와 완료 조건이 명확해야 한다.

```kotlin
// 나쁜 예시: 사용자 정보를 가져온다.
val user = repository.getUser()

// 좋은 예시: 서버가 204를 반환해도 기존 사용자 정보를 유지해야 한다.
val user = repository.getUser()
```

---

## 5. Kotlin 작성 규칙

### Null 처리

* 불필요한 nullable 타입을 만들지 않는다.
* `!!` 사용을 피한다.
* nullable 값이 정상적인 상태인지 오류 상태인지 구분한다.
* 기본값으로 문제를 숨기지 않는다.
* `?.let` 중첩이 깊어지면 명시적인 조건문이나 함수 분리를 고려한다.

```kotlin
// 지양
val userName = response.user!!.name!!

// 권장
val user = response.user ?: return Result.failure(UserNotFoundException())
val userName = user.name
```

### 함수

* 함수는 하나의 명확한 책임을 갖도록 작성한다.
* 함수명이 구현 방식이 아닌 의도를 표현하도록 한다.
* Boolean 반환 함수는 `is`, `has`, `can`, `should` 등의 접두사를 사용한다.
* 기본 인자로 동작을 과도하게 숨기지 않는다.
* 인자가 많아지면 단순히 데이터 클래스로 감싸기보다 함수 책임이 과도한지 먼저 확인한다.

### 클래스와 데이터 모델

* 변경 불가능한 데이터는 `val`과 immutable collection을 우선한다.
* 값 전달 목적의 모델은 `data class`를 사용한다.
* 제한된 상태 집합은 `sealed interface`, `sealed class`, `enum class`를 상황에 맞게 사용한다.
* 도메인 모델과 네트워크 DTO를 무조건 분리하지 않는다.
* 서버 스키마와 화면 요구사항이 다르거나 변환 책임이 존재할 때 분리한다.
* Android Framework 타입을 Domain 계층에 노출하지 않는다.

### Scope Function

* `let`, `run`, `apply`, `also`, `with`를 단순히 코드를 줄이기 위해 사용하지 않는다.
* scope function 중첩을 피한다.
* 수신 객체와 반환 값의 의미가 불분명해지면 일반 변수와 명시적 코드를 사용한다.

### 확장 함수

* 특정 타입에 자연스럽게 속하는 범용 동작에만 사용한다.
* 프로젝트 흐름이나 외부 의존성을 숨기는 확장 함수는 만들지 않는다.
* 이름 충돌 가능성이 높은 범용 확장 함수는 피한다.

---

## 6. 아키텍처 규칙

### 계층 책임

#### UI 계층

* 화면 표시와 사용자 입력 처리를 담당한다.
* Android UI 관련 타입을 사용한다.
* 비즈니스 규칙을 직접 구현하지 않는다.
* Repository 구현체에 직접 접근하지 않는다.
* ViewModel이 제공하는 상태와 이벤트를 사용한다.

#### Domain 계층

* 애플리케이션의 비즈니스 규칙을 담당한다.
* Android Framework에 의존하지 않는다.
* 단순 CRUD 전달만 하는 UseCase는 만들지 않는다.
* 여러 Repository 조합, 정책 적용, 재사용되는 비즈니스 규칙이 있을 때 UseCase를 사용한다.

#### Data 계층

* 네트워크, 데이터베이스, 파일, 캐시 등 데이터 접근을 담당한다.
* DTO와 Entity 변환을 담당한다.
* 데이터 소스 선택과 캐시 정책을 관리한다.
* UI 상태를 직접 생성하지 않는다.

### 의존성 방향

```text
UI -> Domain -> Data abstraction
Data implementation -> Domain
```

* UI가 Retrofit API, Room DAO 등에 직접 의존하지 않도록 한다.
* Domain 계층이 Android Context, Activity, Fragment, Compose 타입에 의존하지 않도록 한다.
* 단순한 소규모 프로젝트에서는 불필요한 계층 분리를 강제하지 않는다.

---

## 7. ViewModel 규칙

* 화면 상태와 화면 단위 비즈니스 흐름을 관리한다.
* Activity, Fragment, View, NavController를 직접 참조하지 않는다.
* `Context`가 필요한 로직은 가능한 한 별도 의존성으로 분리한다.
* `AndroidViewModel`은 실제로 Application Context가 필요한 경우에만 사용한다.
* 외부에는 immutable 상태만 노출한다.
* `MutableStateFlow`와 `MutableLiveData`를 외부에 공개하지 않는다.
* ViewModel에서 일회성 UI 객체를 보관하지 않는다.
* 여러 화면에서 공통으로 사용하는 상태라고 해서 무조건 싱글턴으로 만들지 않는다.

```kotlin
class UserViewModel(
    private val userRepository: UserRepository,
) : ViewModel() {

    private val _uiState = MutableStateFlow(UserUiState())
    val uiState: StateFlow<UserUiState> = _uiState.asStateFlow()
}
```

---

## 8. UI 상태 설계

화면 상태는 서로 모순되는 여러 Boolean보다 명확한 상태 모델로 표현한다.

```kotlin
sealed interface UserUiState {
    data object Loading : UserUiState
    data class Success(val user: User) : UserUiState
    data class Error(val message: String) : UserUiState
}
```

다만 일부 데이터만 갱신되는 복합 화면에서는 하나의 `data class` 상태가 더 적합할 수 있다.

```kotlin
data class UserUiState(
    val isLoading: Boolean = false,
    val user: User? = null,
    val errorMessage: String? = null,
)
```

다음 기준으로 선택한다.

* 상태가 완전히 배타적이면 sealed 타입을 고려한다.
* 여러 상태가 동시에 존재할 수 있으면 data class를 고려한다.
* 화면의 모든 값을 무조건 하나의 거대한 상태 객체에 넣지 않는다.
* 상태 객체에 Activity, Fragment, View, Context, Bitmap 등 무거운 객체를 저장하지 않는다.

---

## 9. 이벤트 처리

* 사용자 입력은 명확한 함수나 이벤트 타입으로 ViewModel에 전달한다.
* 이벤트 종류가 적으면 개별 함수를 사용한다.
* 이벤트가 많고 일관된 reducer 구조를 사용하는 프로젝트라면 sealed event 타입을 사용한다.
* 단순한 클릭 하나 때문에 이벤트 추상화를 추가하지 않는다.

```kotlin
fun onRetryClick() {
    loadUser()
}
```

또는:

```kotlin
sealed interface UserEvent {
    data object RetryClicked : UserEvent
    data class NameChanged(val value: String) : UserEvent
}
```

### 일회성 이벤트

Snackbar, Toast, 화면 이동과 같은 일회성 이벤트는 일반 화면 상태와 구분한다.

* 이벤트가 재수집되어 중복 실행되지 않도록 한다.
* `Channel`, `SharedFlow`, 상태 기반 소비 방식 중 프로젝트의 기존 방식을 따른다.
* 이벤트 래퍼 클래스를 무조건 추가하지 않는다.
* 화면 이동은 가능하면 UI 계층에서 수행한다.

---

## 10. Coroutines와 Flow

### Coroutine Scope

* ViewModel에서는 `viewModelScope`를 사용한다.
* LifecycleOwner에서는 `lifecycleScope`를 사용한다.
* UI에서 Flow를 수집할 때 Lifecycle을 고려한다.
* 임의로 `CoroutineScope`를 생성하지 않는다.
* `GlobalScope`를 사용하지 않는다.

### Dispatcher

* 네트워크 라이브러리나 Room처럼 자체적으로 스레드를 관리하는 API에 불필요하게 `Dispatchers.IO`를 중복 적용하지 않는다.
* CPU 연산은 `Dispatchers.Default`를 고려한다.
* 블로킹 I/O는 `Dispatchers.IO`에서 실행한다.
* Dispatcher가 테스트에 영향을 주면 주입 가능한 구조를 고려한다.
* 모든 suspend 함수에 기계적으로 `withContext`를 추가하지 않는다.

### 예외 처리

* `CancellationException`을 일반 오류로 처리하지 않는다.
* 무조건적인 `catch (e: Exception)`으로 오류를 삼키지 않는다.
* 복구 가능한 오류와 치명적인 개발 오류를 구분한다.
* 오류를 로그만 남기고 성공처럼 처리하지 않는다.
* Repository가 예외를 던질지 `Result`를 반환할지는 기존 프로젝트 규칙을 따른다.
* 계층마다 같은 예외를 반복해서 감싸지 않는다.

```kotlin
try {
    repository.refresh()
} catch (e: CancellationException) {
    throw e
} catch (e: IOException) {
    _uiState.update { it.copy(errorMessage = "네트워크 연결을 확인해 주세요.") }
}
```

### Flow

* 한 번만 가져오는 값에 불필요하게 Flow를 사용하지 않는다.
* 지속적으로 변경되는 데이터나 관찰이 필요한 데이터에 Flow를 사용한다.
* UI에서는 `collectAsStateWithLifecycle` 또는 프로젝트의 lifecycle-aware 방식을 사용한다.
* `stateIn`, `shareIn`은 공유나 상태 보존이 실제로 필요한 경우에만 사용한다.
* `combine` 체인이 과도하게 복잡해지면 상태 책임을 재검토한다.

---

## 11. Jetpack Compose 규칙

### Composable 설계

* Composable은 가능한 한 상태를 외부에서 전달받는 stateless 구조를 우선한다.
* 화면 단위 Composable은 ViewModel과 연결할 수 있다.
* 재사용 UI 컴포넌트는 ViewModel이나 Repository에 직접 접근하지 않는다.
* Composable 내부에서 네트워크 요청이나 데이터베이스 접근을 직접 수행하지 않는다.
* UI 상태와 이벤트 콜백을 명시적인 파라미터로 전달한다.

```kotlin
@Composable
fun UserScreen(
    state: UserUiState,
    onRetryClick: () -> Unit,
    modifier: Modifier = Modifier,
)
```

### Modifier

* 재사용 가능한 Composable은 가능하면 `modifier: Modifier = Modifier`를 제공한다.
* `modifier`는 일반적으로 필수 UI 요소 중 가장 바깥쪽에 적용한다.
* 호출자가 전달한 modifier 뒤에 임의의 크기나 padding을 강제하여 확장성을 막지 않는다.
* Modifier 순서에 따라 동작이 달라질 수 있음을 고려한다.
* 모든 Composable에 기계적으로 modifier를 추가하지 않는다.

### 상태

* 화면 회전이나 프로세스 복원 후 유지되어야 하는 UI 상태는 `rememberSaveable`을 고려한다.
* 단순 recomposition 동안만 필요한 값은 `remember`를 사용한다.
* 계산 비용이 낮은 값까지 불필요하게 `remember`로 감싸지 않는다.
* 파생 상태는 필요할 때 `derivedStateOf`를 사용한다.
* ViewModel 상태를 Composable 내부 상태로 다시 복사하지 않는다.

### Side Effect

* `LaunchedEffect`의 key를 정확히 지정한다.
* 매 recomposition마다 다시 실행되는 잘못된 key 사용을 피한다.
* 상태 변경을 감지하기 위해 무분별하게 `LaunchedEffect`를 사용하지 않는다.
* 콜백 최신 값이 필요하면 `rememberUpdatedState`를 고려한다.
* 정리 작업이 필요하면 `DisposableEffect`를 사용한다.
* Composable 본문에서 직접 side effect를 실행하지 않는다.

### Recomposition

* 성능 문제가 확인되지 않은 상태에서 과도한 최적화를 하지 않는다.
* 불안정한 객체와 람다 생성이 실제 recomposition 문제를 만드는지 확인한다.
* `remember`를 모든 값에 적용하지 않는다.
* Lazy 목록에는 안정적인 key를 제공한다.
* 대규모 목록에서 아이템 전체가 불필요하게 갱신되지 않도록 상태 범위를 조정한다.

```kotlin
LazyColumn {
    items(
        items = users,
        key = { user -> user.id },
    ) { user ->
        UserItem(user = user)
    }
}
```

### Preview

* 재사용 UI 컴포넌트에는 가능한 범위에서 Preview를 제공한다.
* Preview가 실제 네트워크, 데이터베이스, DI 컨테이너에 의존하지 않도록 한다.
* 화면 상태별 Preview가 유용하면 Loading, Success, Error를 분리한다.
* Preview 전용 샘플 데이터를 production 로직에 섞지 않는다.

---

## 12. XML View와 Fragment 규칙

기존 프로젝트가 XML View 기반이면 Compose 도입을 강제하지 않는다.

### Fragment

* Fragment는 화면 표시와 이벤트 연결에 집중한다.
* 비즈니스 로직을 Fragment에 직접 구현하지 않는다.
* ViewBinding 참조는 View Lifecycle에 맞게 해제한다.
* Fragment Lifecycle과 View Lifecycle을 구분한다.
* Flow나 LiveData는 가능한 한 `viewLifecycleOwner` 기준으로 수집한다.
* Fragment에서 Activity를 무조건 구체 타입으로 캐스팅하지 않는다.

```kotlin
private var _binding: FragmentUserBinding? = null
private val binding get() = requireNotNull(_binding)

override fun onDestroyView() {
    super.onDestroyView()
    _binding = null
}
```

### RecyclerView

* `ListAdapter`와 `DiffUtil` 사용 여부는 기존 프로젝트 방식을 따른다.
* `notifyDataSetChanged()` 사용을 피한다.
* ViewHolder에서 position을 장기간 저장하지 않는다.
* 클릭 시점에는 `bindingAdapterPosition`의 유효성을 확인한다.
* Adapter가 화면 비즈니스 로직을 담당하지 않도록 한다.

### Custom View

* 단순 UI 조합을 위해 불필요한 Custom View를 만들지 않는다.
* Lifecycle이나 Context를 장기간 보관하지 않는다.
* XML Attribute가 필요한 경우 기본값과 리소스 해제를 고려한다.

---

## 13. Lifecycle 규칙

* Activity, Fragment, View, Compose의 Lifecycle 차이를 고려한다.
* Fragment View보다 오래 살아가는 객체가 ViewBinding이나 View 참조를 보관하지 않도록 한다.
* Context가 필요한 경우 수명에 맞는 Context를 사용한다.
* Application Context로 UI 작업을 수행하지 않는다.
* Activity Context를 싱글턴에 저장하지 않는다.
* Listener, Callback, Observer 등록 시 해제 시점을 확인한다.
* 화면이 보이지 않을 때 계속 실행될 필요가 없는 작업은 Lifecycle에 맞춰 중지한다.

---

## 14. Navigation 규칙

* 화면 간에는 필요한 최소 데이터만 전달한다.
* 큰 객체 전체를 Bundle, Intent, SavedStateHandle에 넣지 않는다.
* ID를 전달하고 목적지에서 데이터를 조회하는 방식을 우선 고려한다.
* Navigation argument 타입과 nullable 여부를 명확히 한다.
* 중복 화면 이동을 방지한다.
* 딥링크 입력값은 신뢰하지 않고 검증한다.
* ViewModel에서 NavController를 직접 사용하지 않는다.
* 화면 이동 이벤트는 UI 계층에서 처리한다.

---

## 15. 의존성 주입

* 프로젝트에서 Hilt, Koin, 수동 DI 중 이미 사용 중인 방식을 따른다.
* 단위 테스트가 필요한 의존성은 교체 가능한 구조를 고려한다.
* 모든 클래스를 인터페이스로 만들지 않는다.
* 구현체가 하나뿐이고 교체 가능성이나 테스트 이점이 없다면 불필요한 인터페이스를 만들지 않는다.
* Context qualifier를 명확하게 구분한다.
* Activity 수명의 객체를 Singleton 범위에 주입하지 않는다.
* DI 모듈에서 생성 책임과 수명 범위를 명확히 한다.

---

## 16. 네트워크

* API 응답을 성공으로 가정하지 않는다.
* HTTP 상태 코드, 응답 body 부재, 역직렬화 실패를 고려한다.
* 서버 오류 메시지를 그대로 사용자에게 노출하지 않는다.
* 인증 토큰을 로그로 출력하지 않는다.
* 네트워크 DTO에 UI 표시 문자열을 포함하지 않는다.
* API 모델 변경에 취약한 무분별한 nullable 처리를 피한다.
* 서버가 실제로 null을 반환할 수 있는 필드만 nullable로 선언한다.
* 타임아웃과 재시도는 요청 성격에 따라 결정한다.
* POST, 결제, 주문처럼 중복 실행 위험이 있는 요청을 자동 재시도하지 않는다.
* API 인터페이스와 실제 base URL 환경을 확인한 뒤 코드를 작성한다.

---

## 17. 로컬 저장소

### Room

* 메인 스레드에서 데이터베이스 작업을 수행하지 않는다.
* Entity와 Domain 모델의 분리는 실제 차이가 있을 때 적용한다.
* Migration 없이 데이터베이스 버전을 올리지 않는다.
* destructive migration은 데이터 유실이 허용되는 경우에만 사용한다.
* 쿼리 결과가 변경될 수 있으면 Flow 사용을 고려한다.
* 트랜잭션이 필요한 여러 작업은 `@Transaction` 또는 명시적 트랜잭션으로 처리한다.
* 인덱스는 조회 패턴을 근거로 추가한다.

### DataStore

* 단순 key-value 설정은 Preferences DataStore를 고려한다.
* 구조화된 데이터와 스키마 안정성이 필요하면 Proto DataStore를 고려한다.
* 민감한 값을 평문으로 저장하지 않는다.
* 대용량 데이터나 관계형 데이터를 DataStore에 저장하지 않는다.

### 파일과 캐시

* 앱 내부 저장소와 외부 저장소의 차이를 고려한다.
* 캐시 데이터는 언제든 삭제될 수 있다고 가정한다.
* 파일 접근 시 예외와 저장 공간 부족을 고려한다.
* Scoped Storage 정책을 우회하지 않는다.

---

## 18. 권한 처리

* 실제 기능에 필요한 최소 권한만 요청한다.
* 사용 시점에 권한을 요청한다.
* 권한 요청 전에 필요한 이유를 사용자에게 설명할 수 있도록 한다.
* 거부와 영구 거부 상태를 구분한다.
* 권한이 없을 때 앱이 비정상 종료되지 않도록 한다.
* Android 버전에 따라 권한 동작이 달라지는지 확인한다.
* Manifest에 선언만 하고 사용하지 않는 권한을 제거한다.
* 사용자가 권한을 허용한다고 가정하지 않는다.

---

## 19. 백그라운드 작업

* 즉시 실행할 단기 작업에 WorkManager를 남용하지 않는다.
* 지연 가능하고 실행 보장이 필요한 작업에 WorkManager를 사용한다.
* 정확한 시간 실행이 필요하면 AlarmManager 정책과 권한을 확인한다.
* 장시간 사용자 인지가 필요한 작업은 Foreground Service를 검토한다.
* Android 백그라운드 실행 제한을 우회하지 않는다.
* Worker는 재실행될 수 있다고 가정하고 멱등성을 고려한다.
* 실패와 재시도 조건을 구분한다.
* 무한 재시도를 설정하지 않는다.

---

## 20. 리소스 관리

* 사용자에게 표시되는 문자열은 가능한 한 `strings.xml`에 정의한다.
* 문자열 연결보다 format resource를 사용한다.
* 색상과 크기를 코드에 반복해서 하드코딩하지 않는다.
* 다크 테마와 시스템 테마를 고려한다.
* 화면 방향과 화면 크기가 달라도 레이아웃이 깨지지 않도록 한다.
* 특정 기기 해상도만 기준으로 구현하지 않는다.
* 사용하지 않는 drawable, string, color 리소스를 제거한다.
* 이미지 리소스 형식과 크기를 확인한다.
* 의미가 같은 값을 여러 리소스에 중복 정의하지 않는다.

---

## 21. 접근성

* 클릭 가능한 UI에 적절한 접근성 설명을 제공한다.
* 장식용 이미지는 불필요하게 읽히지 않도록 한다.
* 텍스트 크기 확대 시 잘리거나 겹치지 않도록 한다.
* 색상만으로 상태를 구분하지 않는다.
* 터치 영역은 충분한 크기를 확보한다.
* Compose에서는 `semantics`, `contentDescription`, role을 상황에 맞게 사용한다.
* XML에서는 `contentDescription`, label 연결, focus 순서를 확인한다.
* 테스트 태그를 접근성 설명 대신 사용하지 않는다.

---

## 22. 보안

* API Key, 비밀번호, 토큰, 인증서 비밀값을 소스 코드에 직접 작성하지 않는다.
* 비밀값을 Git 저장소에 커밋하지 않는다.
* `BuildConfig`에 포함된 값은 앱 바이너리에서 추출될 수 있다고 가정한다.
* 민감한 사용자 정보를 로그로 출력하지 않는다.
* HTTPS 인증서 검증을 임의로 비활성화하지 않는다.
* WebView에서 불필요한 JavaScript interface를 노출하지 않는다.
* 외부 Intent와 딥링크 입력을 검증한다.
* exported Component를 최소화한다.
* 민감 데이터 저장 시 Android Keystore 사용 여부를 검토한다.
* 화면 캡처 방지는 실제 보안 요구사항이 있을 때만 적용한다.
* 난독화를 보안 수단 자체로 간주하지 않는다.

---

## 23. WebView

* WebView가 필요하지 않은 기능에 WebView를 사용하지 않는다.
* JavaScript는 필요한 경우에만 활성화한다.
* `addJavascriptInterface`로 노출되는 API를 최소화한다.
* 외부 URL 이동 정책을 명확히 한다.
* 파일 접근과 mixed content 허용을 기본적으로 비활성화한다.
* SSL 오류를 무시하고 계속 진행하지 않는다.
* 쿠키 저장소와 네이티브 네트워크 쿠키가 자동으로 동일하다고 가정하지 않는다.
* Activity나 Fragment 수명보다 오래 WebView를 보관하지 않는다.
* WebView 정리 시 callback과 참조 해제를 고려한다.
* 사용자 입력이나 외부 URL을 그대로 `loadUrl`에 전달하지 않는다.

---

## 24. 성능과 메모리

* 실제 성능 문제가 확인되지 않은 상태에서 복잡한 최적화를 추가하지 않는다.
* Activity, Fragment, View, Context의 메모리 누수를 확인한다.
* 긴 수명의 객체가 짧은 수명의 UI 객체를 참조하지 않도록 한다.
* 대용량 Bitmap을 원본 크기로 무조건 로드하지 않는다.
* 이미지 로딩은 프로젝트의 기존 이미지 라이브러리를 사용한다.
* 목록에서 불필요한 객체 생성과 전체 갱신을 피한다.
* 메인 스레드에서 파일, 네트워크, 대규모 JSON 처리 등을 수행하지 않는다.
* 시작 성능에 영향을 주는 초기화는 실제 필요 시점과 비용을 검토한다.
* Baseline Profile이나 Macrobenchmark는 측정 목적과 환경이 갖춰진 경우 사용한다.
* 추측으로 성능 문제를 단정하지 않고 Profiler, Trace, Benchmark 등 측정 결과를 근거로 판단한다.

---

## 25. 로그와 오류 처리

* 개발 로그는 프로젝트 로깅 방식을 따른다.
* release 빌드에 불필요한 상세 로그를 남기지 않는다.
* 개인정보, 토큰, 결제 정보, 전체 API 응답을 로그로 출력하지 않는다.
* 오류 로그에는 원인 파악에 필요한 문맥을 포함한다.
* 같은 오류를 여러 계층에서 반복 기록하지 않는다.
* 사용자가 해결할 수 없는 내부 오류 내용을 그대로 노출하지 않는다.
* 예외를 빈 catch 블록으로 무시하지 않는다.
* Crash reporting 도구에 전송되는 데이터가 민감 정보를 포함하지 않는지 확인한다.

---

## 26. 테스트 규칙

### 기본 원칙

* 구현 세부사항보다 외부에서 관찰 가능한 동작을 검증한다.
* 테스트 이름은 조건과 기대 결과를 알 수 있도록 작성한다.
* Given-When-Then 구조를 유지한다.
* 테스트 간 상태를 공유하지 않는다.
* 네트워크, 시간, 랜덤 값 등 비결정적 요소를 통제한다.
* 실제로 발생할 수 없는 상황을 과도하게 테스트하지 않는다.
* 단순 getter, data class 생성, 프레임워크 동작 자체를 검증하지 않는다.
* 테스트를 통과시키기 위해 production 코드를 부자연스럽게 노출하지 않는다.

### 테스트 우선순위

1. 비즈니스 규칙
2. 상태 전환
3. 오류와 경계값
4. 데이터 변환
5. 사용자 상호작용
6. 회귀 가능성이 높은 버그
7. 복잡한 비동기 흐름

### 단위 테스트

* ViewModel, UseCase, Repository 정책, Mapper 등 Android Framework 의존성이 적은 로직을 우선한다.
* Coroutine 테스트에서는 `runTest`와 테스트 Dispatcher를 사용한다.
* Flow는 실제 방출 순서와 상태를 검증한다.
* Mock 사용이 과도해지면 Fake 구현을 고려한다.
* 구현 내부의 함수 호출 횟수보다 최종 상태와 결과를 우선 검증한다.

```kotlin
@Test
fun `사용자 조회에 성공하면 성공 상태를 노출한다`() = runTest {
    // Given
    val repository = FakeUserRepository(
        user = User(id = "1", name = "Nira"),
    )
    val viewModel = UserViewModel(repository)

    // When
    viewModel.loadUser()

    // Then
    assertEquals(
        UserUiState.Success(User(id = "1", name = "Nira")),
        viewModel.uiState.value,
    )
}
```

### UI 테스트

* 사용자가 실제로 확인하고 조작할 수 있는 동작을 검증한다.
* 내부 구현 구조나 특정 Composable 계층에 과도하게 의존하지 않는다.
* 접근성 label, text, role 등 사용자 관점의 selector를 우선한다.
* 테스트 편의를 위한 tag는 필요한 경우에만 사용한다.
* 애니메이션, 시간, 비동기 작업으로 인한 flaky test를 방지한다.
* 화면이 렌더링된다는 사실만 검증하는 테스트를 반복해서 만들지 않는다.

### 스크린샷 테스트

* 디자인 회귀 위험이 높은 공통 컴포넌트와 핵심 화면에 사용한다.
* 모든 화면 상태를 무분별하게 스냅샷으로 만들지 않는다.
* 폰트, 시스템 UI, 기기 설정 등 실행 환경을 고정한다.
* 변경된 스냅샷을 이유 확인 없이 갱신하지 않는다.

---

## 27. Gradle과 의존성

* 프로젝트가 Version Catalog를 사용하면 `libs.versions.toml`을 우선 사용한다.
* 기존 의존성 선언 방식을 유지한다.
* 동적 버전인 `+`를 사용하지 않는다.
* alpha, beta, rc 버전은 필요성과 위험을 확인한 뒤 사용한다.
* 새 라이브러리를 추가하기 전에 Android 또는 Kotlin 표준 기능으로 해결 가능한지 확인한다.
* 비슷한 역할의 라이브러리를 중복 추가하지 않는다.
* 의존성 추가 시 적용 모듈을 최소화한다.
* `api`와 `implementation`의 노출 범위를 구분한다.
* annotation processor가 필요하면 KSP 지원 여부와 기존 프로젝트 설정을 확인한다.
* Gradle 설정 변경 시 로컬 빌드뿐 아니라 CI 영향도 확인한다.
* 버전 호환성을 확인하지 않고 AGP, Gradle, Kotlin을 개별적으로 올리지 않는다.

---

## 28. 멀티 모듈

* 모듈 분리는 실제 빌드 경계, 책임 분리, 재사용 필요성이 있을 때 적용한다.
* 파일 수가 많다는 이유만으로 모듈을 분리하지 않는다.
* 순환 의존성을 만들지 않는다.
* 공통 모듈이 모든 기능에 의존하는 구조를 피한다.
* `core` 모듈에 관련 없는 기능을 계속 추가하지 않는다.
* feature 모듈의 공개 API를 최소화한다.
* 구현 세부사항은 가능한 한 모듈 내부에 숨긴다.
* 모듈 간 데이터 모델 공유가 결합도를 높이는지 검토한다.
* 공통 UI 모듈이 특정 화면의 ViewModel이나 비즈니스 로직에 의존하지 않도록 한다.

---

## 29. 빌드 타입과 환경 설정

* debug, release, staging 등 기존 빌드 구성을 확인한다.
* 환경별 URL이나 기능 플래그를 코드 조건문으로 흩어 놓지 않는다.
* release 설정을 debug 설정과 동일하게 가정하지 않는다.
* ProGuard/R8 적용 시 reflection, serialization, DI 관련 규칙을 확인한다.
* release에서만 발생할 수 있는 난독화 문제를 고려한다.
* 서명 정보와 비밀값을 저장소에 포함하지 않는다.
* `BuildConfig.DEBUG`를 비즈니스 정책 분기에 사용하지 않는다.
* 환경별 설정이 앱 동작을 바꾼다면 테스트 가능한 형태로 분리한다.

---

## 30. Manifest와 Android Component

* Activity, Service, Receiver, Provider의 `exported` 값을 명시적으로 확인한다.
* 사용하지 않는 Component와 intent-filter를 제거한다.
* 앱 시작 시 자동 실행되는 ContentProvider와 초기화 코드를 점검한다.
* BroadcastReceiver의 등록 방식이 Android 버전 정책에 맞는지 확인한다.
* Service 시작 제한과 Foreground Service 유형을 확인한다.
* PendingIntent의 mutable, immutable 플래그를 목적에 맞게 설정한다.
* 외부 앱이 전달하는 Intent extra를 신뢰하지 않는다.

---

## 31. 날짜와 시간

* 저장과 통신에는 가능한 한 명확한 timezone 기준을 사용한다.
* 서버 시간, 기기 시간, 표시 시간을 구분한다.
* 날짜 문자열을 직접 잘라서 처리하지 않는다.
* 프로젝트의 `minSdk`와 desugaring 설정에 따라 `java.time` 사용 가능 여부를 확인한다.
* 날짜 포맷 문자열과 Locale을 명시한다.
* 날짜 비교 시 일광 절약 시간제와 timezone 영향을 고려한다.
* 테스트에서 현재 시간을 직접 참조하지 않고 주입 가능한 Clock을 고려한다.

---

## 32. 국제화

* 사용자 표시 문자열을 코드에 직접 작성하지 않는다.
* 복수형은 plurals resource 사용을 고려한다.
* 문자열 순서가 언어마다 같다고 가정하지 않는다.
* RTL 레이아웃을 고려해 `left`, `right`보다 `start`, `end`를 우선한다.
* 숫자, 통화, 날짜 포맷에 Locale을 고려한다.
* 영어 문자열 길이만 기준으로 레이아웃을 설계하지 않는다.
* 번역 키에 화면 위치보다 의미가 드러나는 이름을 사용한다.

---

## 33. 코드 변경 절차

코드를 수정할 때 다음 순서로 진행한다.

1. 관련 파일과 호출 흐름을 확인한다.
2. 기존 구현 방식과 규칙을 파악한다.
3. 문제의 직접 원인을 확인한다.
4. 가장 작은 변경으로 문제를 해결한다.
5. 영향받는 호출부와 상태 흐름을 확인한다.
6. 테스트를 추가하거나 기존 테스트를 수정한다.
7. 컴파일 오류와 정적 분석 오류를 확인한다.
8. 가능한 경우 관련 테스트와 빌드를 실행한다.
9. 임시 코드, 디버그 로그, 사용하지 않는 import를 제거한다.

오류 원인을 확인하지 못한 상태에서 우회 코드부터 추가하지 않는다.

---

## 34. 검증 명령어

프로젝트에 실제로 존재하는 Gradle task를 확인한 후 실행한다.

일반적인 예시는 다음과 같다.

```bash
./gradlew assembleDebug
./gradlew testDebugUnitTest
./gradlew connectedDebugAndroidTest
./gradlew lintDebug
./gradlew ktlintCheck
./gradlew detekt
```

다음 사항을 지킨다.

* 존재 여부를 확인하지 않은 task를 프로젝트 표준 명령어처럼 단정하지 않는다.
* 전체 빌드 비용이 큰 경우 변경된 모듈의 task를 우선 실행한다.
* 테스트나 빌드를 실행하지 못했으면 실행하지 않았다고 명시한다.
* 실패한 테스트를 삭제하거나 무시해서 통과시키지 않는다.
* 테스트 실패가 기존 문제인지 변경으로 인한 문제인지 구분한다.

---

## 35. 코드 리뷰 기준

코드 변경 후 다음 항목을 확인한다.

### 기능

* 요청한 기능이 정상적으로 동작하는가
* 정상, 오류, 빈 데이터, 로딩 상태를 고려했는가
* 화면 회전, 백그라운드 복귀, 프로세스 복원 영향을 고려했는가
* 중복 클릭과 중복 요청 가능성이 있는가

### 구조

* 책임이 적절한 계층에 위치하는가
* 기존 구조와 일관성이 있는가
* 불필요한 추상화가 추가되지 않았는가
* 공개 범위가 필요 이상으로 넓지 않은가

### 비동기

* Lifecycle에 맞는 Scope를 사용하는가
* 취소가 정상적으로 전파되는가
* race condition이나 중복 실행 가능성이 있는가
* 메인 스레드를 차단하는 작업이 있는가

### UI

* 로딩과 오류 상태가 표시되는가
* 재구성 또는 재바인딩 시 중복 실행되지 않는가
* 다크 테마와 글자 크기 확대에 문제가 없는가
* 접근성 정보가 제공되는가

### 보안

* 민감 정보가 로그나 코드에 포함되지 않았는가
* 외부 입력을 검증하는가
* 불필요한 권한이나 exported Component가 추가되지 않았는가

### 테스트

* 변경된 핵심 동작이 검증되는가
* 구현 세부사항에 과도하게 의존하지 않는가
* flaky 가능성이 있는가
* 실패 원인을 알 수 있는 테스트 이름인가

---

## 36. 금지 사항

다음 작업은 명시적인 요청이나 충분한 근거 없이 수행하지 않는다.

* 프로젝트 전체 아키텍처 변경
* XML 프로젝트의 전면 Compose 전환
* Compose 프로젝트의 XML 전환
* 의존성 주입 프레임워크 교체
* 네트워크 라이브러리 교체
* 모든 DTO와 Domain 모델의 일괄 분리
* 모든 클래스의 인터페이스화
* 단순 로직을 위한 UseCase 대량 생성
* 검증되지 않은 성능 최적화
* 전역 상태와 싱글턴 추가
* `GlobalScope` 사용
* `!!`을 통한 nullable 문제 회피
* 빈 catch 블록
* SSL 검증 무시
* 민감 정보 하드코딩
* `notifyDataSetChanged()`를 기본 갱신 방식으로 사용
* 사용자 요청과 무관한 파일의 포맷 변경
* 테스트 실패를 해결하기 위한 테스트 삭제
* deprecated API를 별도 이유 없이 신규 코드에 사용
* 출처가 불분명한 코드를 그대로 복사
* 빌드되지 않은 코드를 완료된 것으로 보고

---

## 37. 작업 결과 보고

작업 완료 후 다음 내용만 필요한 범위에서 전달한다.

* 변경한 핵심 내용
* 변경 이유
* 영향을 받는 영역
* 실행한 테스트 또는 빌드
* 실행하지 못한 검증 항목
* 남아 있는 위험이나 확인이 필요한 부분

확인하지 않은 결과를 성공했다고 표현하지 않는다.
