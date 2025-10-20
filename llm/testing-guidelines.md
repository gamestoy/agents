## COMPREHENSIVE TESTING GUIDELINES

### Imports

Imports should always be at the top of the file, not in the methods or functions.

### TDD Enforcement (MANDATORY)

```python
# ALWAYS follow RED-GREEN-REFACTOR cycle:

# 1. RED: Write failing test first
def test_clone_user_success(self, user_manager, db_session):
    """Test successful user cloning with basic properties."""
    # Test should FAIL initially - no implementation exists
    original_user = make_user(name="Original User")
    db_session.add(original_user)
    db_session.flush()
  
    # This should fail because clone_user method doesn't exist yet
    cloned_user = user_manager.clone_user(original_user, "Cloned User")
    assert cloned_user.name == "Cloned User"

# 2. GREEN: Write minimal implementation to pass test
def clone_user(self, original_user: User, new_name: str) -> User:
    # Minimal implementation to make test pass
    return User(name=new_name)

# 3. REFACTOR: Improve implementation while keeping tests green
def clone_user(self, original_user: User, new_name: str) -> User:
    # Enhanced implementation with proper cloning logic
    if not new_name.strip():
        raise ValueError("User name cannot be empty")
    # ... rest of implementation
```

### Test Organization Patterns

#### Class-Based Test Organization

```python
class TestUserCloning:
    """Test cases for user cloning functionality following TDD RED phase."""
  
    def test_clone_user_success(self, user_manager, db_session):
        """Test successful user cloning with basic properties."""
      
    def test_clone_user_with_preferences(self, user_manager, db_session):
        """Test that user preferences are properly cloned."""
      
    def test_clone_user_duplicate_name_raises_error(self, user_manager, db_session):
        """Test that cloning with duplicate name raises appropriate error."""

class TestItemAggregation:
    @pytest.fixture
    def items(self, item_specs):
        return [make_item(**spec) for spec in item_specs]
  
    def test_aggregation_by_category(self, db_session, item_manager, items):
        # Test implementation
```

#### Nested Class Organization for Complex Scenarios

```python
class TestIsActiveUser:
    @pytest.fixture
    def user(self, suspended, deleted, verified, active):
        return make_user(
            verified_at=datetime.now() if verified else None,
            suspended_at=datetime.now() if suspended else None,
            deleted_at=datetime.now() if deleted else None,
            is_active=active,
        )

    def test_orm_property(self, user, active):
        assert user.is_active is active

    def test_expression(self, db_session, user, active):
        db_session.add(user)
        db_session.flush()
        assert db_session.scalar(select(User.is_active)) is active
```

### Naming Conventions (MANDATORY)

#### Test Method Names

```python
# Pattern: test_{what}_{condition}_{expected_outcome}
def test_clone_user_success(self):
def test_clone_user_duplicate_name_raises_error(self):
def test_clone_user_empty_name_raises_error(self):
def test_clone_user_transaction_rollback_on_error(self):

# For parametrized tests, use descriptive IDs
@pytest.mark.parametrize(
    "user_role, permission_level, expected_access",
    [
        pytest.param(
            UserRole.admin,
            PermissionLevel.full,
            AccessType.granted,
            id="admin-full-access"
        ),
        pytest.param(
            UserRole.viewer, 
            PermissionLevel.read_only, 
            AccessType.limited,
            id="viewer-read-only"
        ),
    ],
)
```

#### Test Class Names

```python
class TestUserCloning:           # Feature-based grouping
class TestItemAggregation:       # Functionality grouping  
class TestProcessInitiation:     # Action-based grouping
class TestIsActiveUser:          # Property-based grouping
```

### AAA Pattern (Arrange-Act-Assert)

Skip the comments:

```python
def test_clone_user_success(self, user_manager, db_session):
    # Arrange - Set up test data and preconditions
    original_user = make_user(
        name="Original User",
        email="original@example.com",
        role=UserRole.standard,
        preferences={"theme": "dark", "language": "en"},
    )
    db_session.add(original_user)
    db_session.flush()

    # Act - Execute the functionality being tested
    cloned_user = user_manager.clone_user(original_user, "Cloned User")

    # Assert - Verify the expected outcomes
    assert cloned_user.name == "Cloned User"
    assert cloned_user.email != original_user.email
    assert cloned_user.id != original_user.id
    assert cloned_user.is_active is False
```

### Parametrized Testing Standards

#### Comprehensive Parameter Coverage

```python
@pytest.mark.parametrize(
    "item_specs, expected_count",
    [
        ([], {}),
        ([{"sku": "product-a"}], {"product-a": 1}),
        ([{"sku": "product-a", "quantity": 2}], {"product-a": 2}),
        ([{"sku": "product-a"}, {"sku": "product-b"}], {"product-a": 1, "product-b": 1}),
    ],
)
class TestItemAggregation:
    @pytest.fixture
    def items(self, item_specs):
        return [make_item(**spec) for spec in item_specs]
```

#### Error Condition Testing with Parametrization

```python
@pytest.mark.parametrize(
    "tasks_spec, expectation",
    [
        pytest.param(
            [
                (TaskType.async_task, "task_a"),
                (TaskType.sync_task, "task_a"),
            ],
            pytest.raises(ProcessException, match="Can't process non-homogeneous"),
            id="different task types",
        ),
        pytest.param(
            [
                (TaskType.async_task, "task_a"),
                (TaskType.async_task, "task_b"),
            ],
            does_not_raise(),
            id="homogeneous tasks",
        ),
    ],
)
def test_process_non_homogeneous_tasks(self, task_manager, db_session, tasks_spec, expectation):
    with expectation:
        task_manager.process_tasks(tasks)
```

### Error Testing Standards

#### Exception Testing with Context Managers

```python
def test_clone_user_duplicate_name_raises_error(self, user_manager, db_session):
    """Test that cloning with duplicate name raises appropriate error."""
    original_user = make_user(name="Original User")
    existing_user = make_user(name="Existing User")
    db_session.add_all([original_user, existing_user])
    db_session.flush()

    with pytest.raises(ValueError, match="already exists"):
        user_manager.clone_user(original_user, "Existing User")
```

#### Transaction Rollback Testing

```python
def test_clone_user_transaction_rollback_on_error(self, user_manager, db_session):
    """Test that failed cloning operations don't leave partial data."""
    original_user = make_user(name="Original User")
    existing_user = make_user(name="Existing User")
    preference = UserPreference(user=original_user, key="theme", value="dark")
    original_user.preferences = [preference]
    db_session.add_all([original_user, existing_user])
    db_session.flush()

    initial_preference_count = db_session.scalar(select(func.count(UserPreference.id)))
    initial_user_count = db_session.scalar(select(func.count(User.id)))

    with pytest.raises(ValueError):
        user_manager.clone_user(original_user, "Existing User")

    final_preference_count = db_session.scalar(select(func.count(UserPreference.id)))
    final_user_count = db_session.scalar(select(func.count(User.id)))
    assert final_preference_count == initial_preference_count
    assert final_user_count == initial_user_count
```

### Database Testing Patterns

#### Using Factory Functions from helpers.py

```python
# Always use helper functions for test data creation
def test_something(self, db_session):
    user = make_user(name="Test User", email="test@example.com")
    order = make_order(items=[make_item()])
    product = make_product(sku="test-sku", category=Category.electronics)
  
    db_session.add_all([user, order])
    db_session.flush()  # Use flush() for partial commits in tests
```

#### Hybrid Property Testing (ORM + SQL Expression)

```python
class TestIsActiveUser:
    def test_orm_property(self, user, is_active):
        """Test the property works in Python/ORM context."""
        assert user.is_active is is_active

    def test_expression(self, db_session, user, is_active):
        """Test the property works in SQL context."""
        db_session.add(user)
        db_session.flush()
        assert db_session.scalar(select(User.is_active)) is is_active

    def test_filterable(self, db_session, user, is_active):
        """Test the property can be used in WHERE clauses."""
        db_session.add(user)
        db_session.flush()
        active_users_q = select(User).where(User.is_active)
        expected_users = [user] if is_active else []
        assert list(db_session.scalars(active_users_q)) == expected_users
```

### Fixture Usage Patterns

#### Manager Fixtures with Dependencies

```python
@pytest.fixture
def user_manager(db_session, tmp_path, email_service, task_queue):
    return UserManager(
        db=db_session,
        file_storage=DummyFileStorage(tmpdir=tmp_path),
        email_sender=lambda _to, _subject: DummyEmailSender(),
        email_service=email_service,
        task_queue=task_queue,
    )
```

#### Parameterized Fixtures

```python
@pytest.fixture
def user(self, suspended, deleted, verified, active):
    return make_user(
        verified_at=datetime.now() if verified else None,
        suspended_at=datetime.now() if suspended else None,
        deleted_at=datetime.now() if deleted else None,
        is_active=active,
    )
```

### Mocking and Patching Standards

#### Monkeypatch for Dependency Injection

```python
@pytest.fixture()
def email_sender(self, monkeypatch, user_manager):
    sender = Mock()
    monkeypatch.setattr(user_manager, "get_email_sender", lambda _to: sender)
    return sender
```

#### Context Manager for Method Assertions

```python
def test_email_sent(self, db_session, user_manager, user, email_sender):
    notification_data = user_manager.get_notification_data(user.id)
    email_content = render_email_template(user.email, notification_data)
    with patch(
        "myapp.managers.user.render_email_template",
        return_value=email_content,
    ):
        user_manager.send_notification(user.email, notification_data)
    assert email_sender.mock_calls == [call.send_email(email_content)]
```

### Route Testing Patterns

#### Permission Testing

```python
@pytest.mark.parametrize(
    "patch_principals, location, expected_status",
    [
        (["developer"], Location.headquarters, HTTPStatus.OK),
        (["location:headquarters:admin"], Location.headquarters, HTTPStatus.OK),
        (["location:branch:admin"], Location.headquarters, HTTPStatus.FORBIDDEN),
        (["location:headquarters:viewer"], Location.headquarters, HTTPStatus.FORBIDDEN),
    ],
    indirect=["patch_principals"],
)
def test_permissions(self, db_session, client, patch_principals, location, expected_status):
    # Test implementation
```

#### API Response Validation

```python
def test_start_task_by_location(self, db_session, client, user_location, req_location, status_code):
    task = make_task(user=make_user(location=user_location))
    user = make_user(is_active=True, assigned_tasks=[task], location=user_location)
    db_session.add(user)
    db_session.flush()

    resp = client.post(
        f"/api/locations/{req_location}/users/{user.id}/start-task",
        json={"task_id": str(task.id)},
    )
    assert resp.status_code == status_code
```

### Time-Based Testing

#### Using freezegun for Deterministic Time

```python
@pytest.mark.parametrize(
    "timestamp",
    [
        pytest.param(datetime(2025, 3, 14, 0, 30), id="before midnight"),
        pytest.param(datetime(2025, 3, 14, 0, 30), id="after midnight"),
    ],
)
def test_report_contents(self, timestamp, report_manager, data):
    with freeze_time(timestamp):
        deadline = datetime.now(utc)
        summary = report_manager.get_summary(data.id)
        text = extract_text(BytesIO(render_report(deadline.today(), summary)))
        assert f"Report Date: {deadline:%d %b %Y}" in text
```

### Performance and Edge Case Testing

#### Boundary Value Testing

```python
@pytest.mark.parametrize("num_items", [0, 5])
@pytest.mark.parametrize("has_pending_tasks", [True, False])
def test_process_batch(db_session, batch_manager, num_items, has_pending_tasks):
    # Test with different boundary conditions
```

#### Empty Collection Handling

```python
def test_clone_user_with_no_preferences(self, user_manager, db_session):
    """Test cloning user with no preferences works correctly."""
    original_user = make_user(name="Original User", email="test@example.com")
    original_user.preferences = []
    db_session.add(original_user)
    db_session.flush()

    result = user_manager.clone_user(original_user, "Cloned User")

    assert len(result.preferences) == 0
    assert result.name == "Cloned User"
```

### Test Documentation Standards

#### Comprehensive Docstrings

```python
def test_clone_user_allows_name_reuse_when_existing_user_is_deleted(
    self, user_manager, db_session
):
    """Test that cloning allows reusing names from deleted users.
  
    This test verifies that the uniqueness constraint for user names
    only applies to active (non-deleted) users, allowing name reuse
    when the existing user with that name has been soft-deleted.
    """
```

#### Class-Level Documentation

```python
class TestUserCloning:
    """Test cases for user cloning functionality following TDD RED phase.
  
    These tests cover all aspects of user cloning including:
    - Basic property copying
    - Preference duplication
    - Validation and error handling
    - Transaction integrity
    - Business rule enforcement
    """
```

### Async Testing (When Required)

```python
class TestAsyncManager:
    async def test_async_operation_success(self, async_db_session):
        # Arrange
        manager = AsyncManager(async_db_session)
      
        # Act
        result = await manager.async_method(data)
      
        # Assert
        assert result.status == "success"
      
    async def test_async_operation_failure(self, async_db_session):
        # Test async error conditions
        with pytest.raises(AsyncException):
            await manager.failing_async_method()
```

### Integration Testing Guidelines

```python
class TestUserManagerIntegration:
    """Integration tests that verify manager works with real dependencies."""
  
    def test_full_workflow_integration(self, user_manager, db_session):
        """Test complete workflow from start to finish."""
        # Test multiple operations together
        # Verify side effects and state changes
        # Test real database transactions
```
