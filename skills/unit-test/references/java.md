# Java Unit Test Reference (JUnit 5 + Mockito)

## 목차
1. [의존성 설정](#의존성-설정)
2. [기본 테스트 구조](#기본-테스트-구조)
3. [Assertion 패턴](#assertion-패턴)
4. [Mockito 사용법](#mockito-사용법)
5. [Spring Boot 테스트](#spring-boot-테스트)
6. [파라미터화 테스트](#파라미터화-테스트)
7. [예외 테스트](#예외-테스트)
8. [테스트 라이프사이클](#테스트-라이프사이클)

---

## 의존성 설정

**Maven (pom.xml)**
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
<!-- spring-boot-starter-test에 JUnit 5, Mockito, AssertJ 포함 -->
```

**Gradle (build.gradle)**
```groovy
testImplementation 'org.springframework.boot:spring-boot-starter-test'
```

Spring Boot 없는 순수 Java:
```xml
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter</artifactId>
    <version>5.10.0</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-core</artifactId>
    <version>5.5.0</version>
    <scope>test</scope>
</dependency>
```

---

## 기본 테스트 구조

```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.DisplayName;
import static org.assertj.core.api.Assertions.*;

class OrderServiceTest {

    @Test
    @DisplayName("주문 금액이 0보다 크면 주문이 생성된다")
    void createOrder_withPositiveAmount_success() {
        // given
        OrderService orderService = new OrderService();
        int amount = 1000;

        // when
        Order order = orderService.createOrder(amount);

        // then
        assertThat(order).isNotNull();
        assertThat(order.getAmount()).isEqualTo(1000);
    }
}
```

**테스트 메서드 명명 규칙:**
- `메서드명_상황_기대결과` 형식 권장
- 예: `createOrder_withZeroAmount_throwsException`
- `@DisplayName`으로 한국어 설명 추가 가능

---

## Assertion 패턴

AssertJ (권장) 사용:

```java
// 기본값 검증
assertThat(result).isEqualTo(expected);
assertThat(result).isNotNull();
assertThat(result).isNull();
assertThat(flag).isTrue();
assertThat(flag).isFalse();

// 숫자
assertThat(price).isGreaterThan(0);
assertThat(price).isBetween(100, 1000);

// 문자열
assertThat(name).startsWith("홍");
assertThat(name).contains("길동");
assertThat(name).hasSize(3);

// 컬렉션
assertThat(list).hasSize(3);
assertThat(list).contains("A", "B");
assertThat(list).containsExactly("A", "B", "C");
assertThat(list).isEmpty();

// 객체 필드 검증
assertThat(order)
    .extracting("id", "amount", "status")
    .containsExactly(1L, 1000, OrderStatus.PENDING);

// 예외
assertThatThrownBy(() -> service.process(null))
    .isInstanceOf(IllegalArgumentException.class)
    .hasMessage("입력값이 null입니다");
```

---

## Mockito 사용법

### 기본 Mock 생성

```java
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import static org.mockito.Mockito.*;
import static org.mockito.BDDMockito.*;

@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock
    private OrderRepository orderRepository;

    @Mock
    private PaymentClient paymentClient;

    @InjectMocks
    private OrderService orderService; // Mock들이 자동 주입됨
}
```

### Stubbing (동작 정의)

```java
// BDD 스타일 (권장)
given(orderRepository.findById(1L)).willReturn(Optional.of(order));
given(paymentClient.pay(any())).willReturn(PaymentResult.success());

// 전통 스타일
when(orderRepository.findById(1L)).thenReturn(Optional.of(order));

// void 메서드
willDoNothing().given(orderRepository).delete(any());
doNothing().when(orderRepository).delete(any());

// 예외 던지기
given(paymentClient.pay(any())).willThrow(new PaymentException("결제 실패"));

// 순차적 응답
given(repository.findAll())
    .willReturn(List.of(order1))
    .willReturn(List.of(order1, order2));
```

### Argument Matchers

```java
given(repo.findById(anyLong())).willReturn(Optional.of(order));
given(service.process(any(Order.class))).willReturn(result);
given(service.search(eq("keyword"))).willReturn(results);
given(repo.findByName(anyString())).willReturn(Optional.empty());
```

### Verify (호출 검증)

```java
// 호출 횟수 검증
then(orderRepository).should().save(any(Order.class));
then(orderRepository).should(times(2)).save(any());
then(orderRepository).should(never()).delete(any());

// 인자 캡처
ArgumentCaptor<Order> captor = ArgumentCaptor.forClass(Order.class);
then(orderRepository).should().save(captor.capture());
Order savedOrder = captor.getValue();
assertThat(savedOrder.getStatus()).isEqualTo(OrderStatus.CONFIRMED);
```

---

## Spring Boot 테스트

### Service 계층 단위 테스트 (슬라이스 없이)

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    private UserRepository userRepository;

    @InjectMocks
    private UserService userService;

    @Test
    void findUser_existingId_returnsUser() {
        // given
        User user = User.builder().id(1L).name("홍길동").build();
        given(userRepository.findById(1L)).willReturn(Optional.of(user));

        // when
        UserDto result = userService.findUser(1L);

        // then
        assertThat(result.getName()).isEqualTo("홍길동");
    }
}
```

### Controller 슬라이스 테스트

```java
@WebMvcTest(UserController.class)
class UserControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private UserService userService;

    @Autowired
    private ObjectMapper objectMapper;

    @Test
    void getUser_existingId_returns200() throws Exception {
        // given
        UserDto dto = new UserDto(1L, "홍길동");
        given(userService.findUser(1L)).willReturn(dto);

        // when & then
        mockMvc.perform(get("/users/1")
                .contentType(MediaType.APPLICATION_JSON))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.name").value("홍길동"));
    }

    @Test
    void createUser_validRequest_returns201() throws Exception {
        CreateUserRequest request = new CreateUserRequest("홍길동", "hong@test.com");
        given(userService.createUser(any())).willReturn(new UserDto(1L, "홍길동"));

        mockMvc.perform(post("/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.id").value(1));
    }
}
```

### Repository 슬라이스 테스트

```java
@DataJpaTest
class UserRepositoryTest {

    @Autowired
    private UserRepository userRepository;

    @Test
    void findByEmail_existingEmail_returnsUser() {
        // given
        User user = User.builder().name("홍길동").email("hong@test.com").build();
        userRepository.save(user);

        // when
        Optional<User> found = userRepository.findByEmail("hong@test.com");

        // then
        assertThat(found).isPresent();
        assertThat(found.get().getName()).isEqualTo("홍길동");
    }
}
```

---

## 파라미터화 테스트

```java
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.*;

@ParameterizedTest
@ValueSource(ints = {-1, 0, -100})
@DisplayName("0 이하의 금액은 예외가 발생한다")
void createOrder_nonPositiveAmount_throwsException(int amount) {
    assertThatThrownBy(() -> orderService.createOrder(amount))
        .isInstanceOf(IllegalArgumentException.class);
}

@ParameterizedTest
@CsvSource({
    "1000, SILVER",
    "5000, GOLD",
    "10000, PLATINUM"
})
void getGrade_byAmount_returnsCorrectGrade(int amount, String expectedGrade) {
    assertThat(gradeService.getGrade(amount)).isEqualTo(Grade.valueOf(expectedGrade));
}

@ParameterizedTest
@MethodSource("provideOrders")
void processOrder_variousOrders(Order order, boolean expectedResult) {
    assertThat(orderService.process(order)).isEqualTo(expectedResult);
}

static Stream<Arguments> provideOrders() {
    return Stream.of(
        Arguments.of(Order.of(1000), true),
        Arguments.of(Order.of(0), false)
    );
}
```

---

## 예외 테스트

```java
// 방법 1: assertThatThrownBy (권장)
assertThatThrownBy(() -> service.findUser(-1L))
    .isInstanceOf(IllegalArgumentException.class)
    .hasMessage("ID는 양수여야 합니다")
    .hasMessageContaining("양수");

// 방법 2: assertThatExceptionOfType
assertThatExceptionOfType(UserNotFoundException.class)
    .isThrownBy(() -> service.findUser(999L))
    .withMessage("사용자를 찾을 수 없습니다: 999");

// 방법 3: @Test expected (JUnit 4 스타일, 비권장)
// assertThrows도 가능하지만 AssertJ 방식이 더 표현력 좋음
```

---

## 테스트 라이프사이클

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @BeforeEach
    void setUp() {
        // 각 테스트 전 실행 — 공통 fixture 초기화
    }

    @AfterEach
    void tearDown() {
        // 각 테스트 후 실행 — 리소스 정리
    }

    @BeforeAll
    static void setUpAll() {
        // 클래스 내 모든 테스트 전 1회 실행
    }

    @AfterAll
    static void tearDownAll() {
        // 클래스 내 모든 테스트 후 1회 실행
    }

    @Nested
    @DisplayName("주문 생성")
    class CreateOrder {
        @Test
        void success() { ... }

        @Test
        void fail_whenAmountZero() { ... }
    }
}
```

**`@Nested`**: 관련 테스트를 그룹화할 때 사용. 컨텍스트별로 묶으면 가독성 향상.
