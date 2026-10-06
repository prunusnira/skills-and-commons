# CMake·CTest 통합

## 기존 구성을 먼저 따른다

- 프로젝트의 최소 CMake 버전, preset, 테스트 옵션, 대상 이름, 의존성 공급 방식과 등록 관례를 확인한다. 제품 라이브러리 대상이 있으면 테스트에 링크한다. 실행 파일만 있다면 CLI 계약을 테스트하거나, 단위 테스트가 필요할 때 최소한의 라이브러리 분리를 검토한다. 소스 파일을 별도 복제해 다른 컴파일 조건으로 빌드하지 않는다.
- 새 등록이 필요하다면 최상위 `CMakeLists.txt`에서 `include(CTest)`를 사용할 수 있다. 이는 `BUILD_TESTING` 옵션을 만들고 기본값이 켜져 있을 때 `enable_testing()`을 호출한다. 기존에 `enable_testing()`이나 자체 옵션이 있으면 중복 구성을 만들지 않는다.

```cmake
include(CTest)

if(BUILD_TESTING)
  add_subdirectory(tests)
endif()
```

- 자체 테스트 실행기라면 `add_test(NAME core_tests COMMAND core_tests)`로 CTest에 등록한다. 이 예시에서 `core_tests`는 실행 파일 대상이며, `add_test`에는 외부 명령도 사용할 수 있다. CTest는 대상을 빌드하지 않으므로 실행 전에 빌드한다.
- GoogleTest가 이미 준비된 프로젝트에서는 테스트 실행 파일을 필요한 제품 라이브러리 대상과 프로젝트가 제공하는 GoogleTest main 대상에 링크하고 `include(GoogleTest)` 뒤 `gtest_discover_tests(core_tests)`로 개별 테스트를 등록할 수 있다. `GTest::gtest_main`은 사용할 수 있는 경우의 대상 이름이다.
- Catch2 v3라면 `Catch2::Catch2WithMain` 또는 자체 main을 쓰는 `Catch2::Catch2`에 링크한다. `include(Catch)` 뒤 `catch_discover_tests(core_tests)`를 사용할 수 있으며, FetchContent로 공급하면 먼저 Catch2의 `extras` 디렉터리를 `CMAKE_MODULE_PATH`에 추가해야 한다. 같은 테스트를 `add_test`와 discovery로 중복 등록하지 않는다.
- discovery는 테스트 실행 파일을 실행해 목록을 얻는다. 크로스 컴파일 환경에서는 실행기나 대상 환경 설정을 확인하고, 실행할 수 없다면 검증 한계를 보고한다.

## 실행과 확인

프로젝트의 preset 또는 빌드 디렉터리를 우선 사용한다. 빌드 디렉터리가 없거나 테스트 설정이 바뀌었다면 프로젝트 방식대로 먼저 configure하고, `BUILD_TESTING`이 켜졌는지 확인한다. CMake 3.20 이상이라면 다음 형태로 실행할 수 있다.

```sh
cmake --build build
ctest --test-dir build --output-on-failure
```

- 다중 구성 생성기에서는 빌드에 `--config Debug`, CTest에 `-C Debug`처럼 같은 구성을 지정한다. 특정 테스트만 실행하려면 해당 실행 파일 대상을 빌드하고, 등록된 이름을 확인한 뒤 `ctest --test-dir build -R '<이름 정규식>' --output-on-failure`로 선택한다.
- CMake 3.20 이전에는 빌드 디렉터리에서 `ctest --output-on-failure`를 실행한다.
- CTest가 테스트를 0개 실행하면 기본적으로 성공 종료할 수 있다. CTest 3.17 이상에서는 `--no-tests=error`를 추가하고, 이전 버전에서는 `ctest -N`으로 등록된 테스트 수를 확인한다. 정규식으로 골랐을 때도 선택된 테스트가 있는지 확인한다.
- configure, build, test를 구분해 확인한다. 테스트 대상 변경 후에는 다시 빌드하고, 필요한 범위의 CTest를 실제 실행해 결과와 실패 수를 읽는다.

참고: [CMake CTest 모듈](https://cmake.org/cmake/help/latest/module/CTest.html), [add_test](https://cmake.org/cmake/help/latest/command/add_test.html), [GoogleTest 모듈](https://cmake.org/cmake/help/latest/module/GoogleTest.html), [FindGTest 대상](https://cmake.org/cmake/help/latest/module/FindGTest.html), [Catch2 CMake 통합](https://github.com/catchorg/Catch2/blob/devel/docs/cmake-integration.md), [CTest 명령](https://cmake.org/cmake/help/latest/manual/ctest.1.html), [CMake 3.17 릴리스 노트](https://cmake.org/cmake/help/v3.29/release/3.17.html).
