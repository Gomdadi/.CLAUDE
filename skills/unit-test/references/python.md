# Python Unit Test Reference (pytest)

## 목차
1. [설치 및 설정](#설치-및-설정)
2. [기본 테스트 구조](#기본-테스트-구조)
3. [Assertion 패턴](#assertion-패턴)
4. [Fixture 패턴](#fixture-패턴)
5. [Mock 패턴](#mock-패턴)
6. [예외 테스트](#예외-테스트)
7. [파라미터화 테스트](#파라미터화-테스트)
8. [비동기 테스트](#비동기-테스트)
9. [FastAPI / Django 테스트](#fastapi--django-테스트)

---

## 설치 및 설정

```bash
pip install pytest pytest-mock pytest-asyncio
# FastAPI 테스트용
pip install httpx
```

`pytest.ini` 또는 `pyproject.toml`:
```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
asyncio_mode = "auto"  # pytest-asyncio 사용 시
```

**파일 구조:**
```
tests/
├── conftest.py          # 공유 fixture 정의
├── test_order_service.py
└── test_user_service.py
```

---

## 기본 테스트 구조

```python
# test_order_service.py
import pytest
from app.services.order_service import OrderService


class TestOrderService:
    def test_create_order_with_positive_amount_success(self):
        # given
        service = OrderService()
        amount = 1000

        # when
        order = service.create_order(amount)

        # then
        assert order is not None
        assert order.amount == 1000

    def test_create_order_with_zero_amount_raises(self):
        service = OrderService()
        with pytest.raises(ValueError, match="금액은 양수여야 합니다"):
            service.create_order(0)
```

**명명 규칙:**
- 파일: `test_*.py` 또는 `*_test.py`
- 함수/메서드: `test_*`로 시작
- 클래스: `Test*`로 시작 (`__init__` 없이)
- 메서드명: `test_메서드_상황_기대결과` 권장

---

## Assertion 패턴

pytest는 표준 `assert` 문을 사용하며, 실패 시 상세한 diff를 출력한다:

```python
# 기본값
assert result == expected
assert result is not None
assert result is None
assert flag is True
assert flag is False

# 숫자
assert price > 0
assert 100 <= price <= 1000
assert abs(0.1 + 0.2 - 0.3) < 1e-9  # 부동소수점

# 문자열
assert "길동" in name
assert name.startswith("홍")
assert len(name) == 3

# 컬렉션
assert len(items) == 3
assert "apple" in items
assert items == ["a", "b", "c"]
assert set(items) == {"a", "b", "c"}

# 딕셔너리
assert result["status"] == "ok"
assert "error" not in result
```

---

## Fixture 패턴

```python
# conftest.py — 여러 테스트 파일에서 공유
import pytest
from app.models import User, Order


@pytest.fixture
def user():
    return User(id=1, name="홍길동", email="hong@test.com")


@pytest.fixture
def order(user):
    return Order(id=1, user=user, amount=1000)


@pytest.fixture
def order_service(mocker):
    repo = mocker.MagicMock()
    return OrderService(repository=repo)
```

```python
# 테스트에서 fixture 사용
class TestOrderService:
    def test_get_order(self, order_service, order):
        order_service.repository.find_by_id.return_value = order
        result = order_service.get_order(1)
        assert result.amount == 1000
```

**Fixture 스코프:**
```python
@pytest.fixture(scope="function")  # 기본값 — 각 테스트마다
@pytest.fixture(scope="class")     # 클래스당 1회
@pytest.fixture(scope="module")    # 모듈당 1회
@pytest.fixture(scope="session")   # 전체 세션 1회
```

---

## Mock 패턴

### pytest-mock (권장)

```python
def test_create_order_saves_to_repository(mocker):
    # given
    mock_repo = mocker.MagicMock()
    service = OrderService(repository=mock_repo)

    # when
    service.create_order(amount=1000)

    # then
    mock_repo.save.assert_called_once()
    call_args = mock_repo.save.call_args[0][0]
    assert call_args.amount == 1000
```

### unittest.mock

```python
from unittest.mock import MagicMock, patch, call

# 직접 mock 생성
mock_repo = MagicMock()
mock_repo.find_by_id.return_value = Order(id=1, amount=1000)
mock_repo.find_by_id.side_effect = [order1, order2]  # 순차 반환
mock_repo.save.side_effect = Exception("DB 오류")    # 예외 발생

# 호출 검증
mock_repo.save.assert_called_once_with(order)
mock_repo.find_by_id.assert_called_with(1)
assert mock_repo.save.call_count == 2
mock_repo.delete.assert_not_called()
```

### patch 데코레이터 / 컨텍스트 매니저

```python
from unittest.mock import patch

# 데코레이터 방식
@patch('app.services.order_service.PaymentClient')
def test_payment_called(mock_payment_class):
    mock_client = mock_payment_class.return_value
    mock_client.charge.return_value = {"status": "ok"}

    service = OrderService()
    service.process_payment(1000)

    mock_client.charge.assert_called_once_with(1000)


# 컨텍스트 매니저 방식
def test_external_api():
    with patch('app.clients.external_api.fetch') as mock_fetch:
        mock_fetch.return_value = {"data": [1, 2, 3]}
        result = service.get_data()
        assert len(result) == 3
```

### mocker.spy — 실제 구현 유지 + 호출 추적

```python
def test_log_called_on_error(mocker):
    spy = mocker.spy(logger, 'error')
    service.process_invalid_input(None)
    spy.assert_called_once()
```

---

## 예외 테스트

```python
import pytest

# 예외 타입만 검증
def test_raises_value_error():
    with pytest.raises(ValueError):
        service.create_order(-1)

# 메시지까지 검증
def test_raises_with_message():
    with pytest.raises(ValueError, match="금액은 양수여야 합니다"):
        service.create_order(-1)

# 예외 정보 접근
def test_raises_and_inspect():
    with pytest.raises(OrderNotFoundException) as exc_info:
        service.get_order(999)
    assert exc_info.value.order_id == 999
    assert "찾을 수 없습니다" in str(exc_info.value)
```

---

## 파라미터화 테스트

```python
import pytest

@pytest.mark.parametrize("amount", [-1, 0, -100])
def test_create_order_non_positive_raises(amount):
    with pytest.raises(ValueError):
        service.create_order(amount)


@pytest.mark.parametrize("amount,expected_grade", [
    (1000, "SILVER"),
    (5000, "GOLD"),
    (10000, "PLATINUM"),
])
def test_get_grade(amount, expected_grade):
    assert service.get_grade(amount) == expected_grade


# ID 지정으로 가독성 향상
@pytest.mark.parametrize("value,expected", [
    pytest.param(None, False, id="null_input"),
    pytest.param("", False, id="empty_string"),
    pytest.param("valid", True, id="valid_input"),
])
def test_validate(value, expected):
    assert service.validate(value) == expected
```

---

## 비동기 테스트

```python
import pytest
import pytest_asyncio

# pytest.ini에 asyncio_mode = "auto" 설정 시 마커 불필요
@pytest.mark.asyncio
async def test_async_get_user():
    # given
    user = User(id=1, name="홍길동")
    mock_repo.find_by_id = AsyncMock(return_value=user)

    # when
    result = await user_service.get_user(1)

    # then
    assert result.name == "홍길동"


# AsyncMock 사용
from unittest.mock import AsyncMock

mock_client = AsyncMock()
mock_client.fetch.return_value = {"status": "ok"}
```

---

## FastAPI / Django 테스트

### FastAPI

```python
import pytest
from fastapi.testclient import TestClient
from httpx import AsyncClient
from app.main import app


# 동기 클라이언트
@pytest.fixture
def client():
    return TestClient(app)


def test_get_user(client, mocker):
    mocker.patch('app.routers.users.user_service.get_user',
                 return_value={"id": 1, "name": "홍길동"})

    response = client.get("/users/1")
    assert response.status_code == 200
    assert response.json()["name"] == "홍길동"


def test_create_user_invalid_body(client):
    response = client.post("/users", json={"name": ""})
    assert response.status_code == 422  # Validation error


# 비동기 클라이언트
@pytest.mark.asyncio
async def test_get_user_async():
    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.get("/users/1")
    assert response.status_code == 200
```

### Django

```python
import pytest
from django.test import TestCase, Client
from django.urls import reverse


class UserViewTest(TestCase):
    def setUp(self):
        self.client = Client()
        self.user = User.objects.create(name="홍길동")

    def test_get_user(self):
        url = reverse("user-detail", kwargs={"pk": self.user.pk})
        response = self.client.get(url)
        self.assertEqual(response.status_code, 200)
        self.assertEqual(response.json()["name"], "홍길동")


# pytest-django 사용 시
@pytest.mark.django_db
def test_create_user():
    user = User.objects.create(name="홍길동")
    assert User.objects.count() == 1
    assert user.name == "홍길동"
```
