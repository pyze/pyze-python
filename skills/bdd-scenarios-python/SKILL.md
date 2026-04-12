---
name: bdd-scenarios-python
description: "This skill should be used when writing BDD scenarios in pytest, structuring tests as Given/When/Then, using parametrize for spec-derived scenarios, or deciding between behavior testing and implementation testing in Python."
---

# BDD Scenarios in Python

Python companion to the bdd-scenarios skill (in pyze-workflow). This skill shows how to implement Given/When/Then scenarios using pytest.

## Given/When/Then in pytest

Structure each test with clearly marked phases. Use descriptive function names that read as behavior specifications.

```python
def test_authentication_with_valid_credentials_returns_token():
    # Given: a registered user exists
    user = create_test_user(email="alice@example.com", password="secret123")

    # When: authenticating with correct credentials
    token = authenticate(email="alice@example.com", password="secret123")

    # Then: returns a valid auth token for that user
    assert token is not None
    assert token.user_id == user.id
    assert token.is_expired is False

def test_authentication_with_wrong_password_raises_auth_error():
    # Given: a registered user exists
    create_test_user(email="alice@example.com", password="secret123")

    # When/Then: authenticating with wrong password raises AuthError
    with pytest.raises(AuthError, match="Invalid credentials"):
        authenticate(email="alice@example.com", password="wrong")
```

---

## Parametrized Scenarios

Use `@pytest.mark.parametrize` when a spec example has multiple valid inputs that should all produce the same behavior pattern.

```python
@pytest.mark.parametrize("invalid_email,expected_error", [
    ("", "Email is required"),
    ("notanemail", "Invalid email format"),
    ("missing@", "Invalid email format"),
    ("@nodomain.com", "Invalid email format"),
])
def test_registration_rejects_invalid_email(invalid_email, expected_error):
    # When/Then: invalid email raises ValidationError
    with pytest.raises(ValidationError, match=expected_error):
        register_user(email=invalid_email, name="Test", password="valid123")


@pytest.mark.parametrize("amount,expected_tier", [
    (0, "bronze"),
    (500, "bronze"),
    (1000, "silver"),
    (5000, "gold"),
    (10000, "platinum"),
])
def test_loyalty_tier_assignment(amount, expected_tier):
    # Given: a customer with total purchases
    customer = create_customer(total_purchases=amount)

    # When: determining loyalty tier
    tier = calculate_tier(customer)

    # Then: tier matches expected level
    assert tier == expected_tier
```

---

## Fixtures as Given State

Use `@pytest.fixture` to represent preconditions. Fixtures make the "Given" phase reusable across tests.

```python
@pytest.fixture
def registered_user(db_session):
    """Given: a registered user exists in the database."""
    return create_test_user(
        email="alice@example.com",
        password="secret123",
        db=db_session,
    )

@pytest.fixture
def authenticated_user(registered_user):
    """Given: a user is authenticated with a valid token."""
    token = authenticate(registered_user.email, "secret123")
    return registered_user, token

def test_authenticated_user_can_view_profile(authenticated_user):
    user, token = authenticated_user
    # When: requesting own profile
    profile = get_profile(user.id, auth_token=token)
    # Then: returns profile data
    assert profile["email"] == user.email

def test_authenticated_user_cannot_view_other_profile(authenticated_user):
    _, token = authenticated_user
    # When/Then: requesting someone else's profile raises Forbidden
    with pytest.raises(ForbiddenError):
        get_profile(user_id=9999, auth_token=token)
```

---

## Edge Cases from Spec Uncertainties

When a spec has a decision point ("Token expires in 24h or 7d?"), write test scenarios for both to catch regressions if the decision changes.

```python
from freezegun import freeze_time
from datetime import datetime, timedelta

def test_token_valid_within_expiry_window():
    # Given: a token created now
    token = create_auth_token(user_id=1)

    # When: checking validity 23 hours later
    with freeze_time(datetime.now() + timedelta(hours=23)):
        # Then: token is still valid
        assert is_token_valid(token) is True

def test_token_expired_after_24_hours():
    # Given: a token created now
    token = create_auth_token(user_id=1)

    # When: checking validity 25 hours later
    with freeze_time(datetime.now() + timedelta(hours=25)):
        # Then: token is expired
        assert is_token_valid(token) is False
```

---

## Behavior vs Implementation Testing

Test what a function **does**, not **how** it does it.

**WRONG — testing implementation details:**

```python
@patch("app.services.user_repo.query")
@patch("app.services.token_generator.create")
def test_authenticate(mock_create, mock_query):
    mock_query.return_value = User(id=1)
    mock_create.return_value = Token(value="abc")

    result = authenticate("alice@example.com", "secret")

    # Testing HOW it works — these break when refactoring
    mock_query.assert_called_once_with(email="alice@example.com")
    mock_create.assert_called_once_with(user_id=1)
```

**RIGHT — testing behavior:**

```python
def test_authenticate_returns_valid_token():
    # Given: user exists
    create_test_user(email="alice@example.com", password="secret")

    # When: authenticating
    token = authenticate("alice@example.com", "secret")

    # Then: token is valid and identifies the user
    assert token is not None
    assert token.user_id == 1
    assert not token.is_expired
```

The right version survives refactoring (swap database, change token format, add caching) because it tests the contract, not the wiring.

---

## pytest-bdd vs Plain pytest

**Use pytest-bdd** when:
- Stakeholders (product, QA) need to read and write test scenarios
- You have a formal Gherkin spec (.feature files) to implement
- Scenarios span multiple systems (integration/acceptance tests)

**Use plain pytest** when:
- Tests are developer-facing (unit and integration)
- Given/When/Then comments in test functions provide sufficient structure
- You want minimal framework overhead

Most Python projects get full value from plain pytest with descriptive test names and Given/When/Then comments. Add pytest-bdd only when non-developer stakeholders need to author or review scenarios.
