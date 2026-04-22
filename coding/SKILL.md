---
name: coding
description: Behavioral guidelines to reduce common LLM coding mistakes. Use when writing, reviewing, or refactoring code to avoid overcomplication, make surgical changes, surface assumptions, and define verifiable success criteria.
---

# Coding Guidelines

Behavioral guidelines to reduce common LLM coding mistakes.

**Tradeoff:** These guidelines bias toward caution over speed. For trivial tasks, use judgment.

## 1. Think Before Coding

**Don't assume. Don't hide confusion. Surface tradeoffs.**

Before implementing:
- State your assumptions explicitly. If uncertain, ask.
- If multiple interpretations exist, present them — don't pick silently.
- If a simpler approach exists, say so. Push back when warranted.
- If something is unclear, stop. Name what's confusing. Ask.

## 2. Simplicity First

**Minimum code that solves the problem. Nothing speculative.**

- No features beyond what was asked.
- No abstractions for single-use code.
- No "flexibility" or "configurability" that wasn't requested.
- No error handling for impossible scenarios.
- If you write 200 lines and it could be 50, rewrite it.

Ask yourself: "Would a senior engineer say this is overcomplicated?" If yes, simplify.

## 3. Surgical Changes

**Touch only what you must. Clean up only your own mess.**

When editing existing code:
- Don't "improve" adjacent code, comments, or formatting.
- Don't refactor things that aren't broken.
- Match existing style, even if you'd do it differently.
- If you notice unrelated dead code, mention it — don't delete it.

When your changes create orphans:
- Remove imports/variables/functions that YOUR changes made unused.
- Don't remove pre-existing dead code unless asked.

The test: every changed line should trace directly to the user's request.

## 4. Goal-Driven Execution

**Define success criteria. Loop until verified.**

Transform tasks into verifiable goals:
- "Add validation" → "Write tests for invalid inputs, then make them pass"
- "Fix the bug" → "Write a test that reproduces it, then make it pass"
- "Refactor X" → "Ensure tests pass before and after"

For multi-step tasks, state a brief plan:
1. [Step] → verify: [check]
2. [Step] → verify: [check]
3. [Step] → verify: [check]

Strong success criteria let you loop independently. Weak criteria ("make it work") require constant clarification.

---

# Examples

Real-world code examples demonstrating the four principles.

---

## 1. Think Before Coding

### Example 1: Hidden Assumptions

**User Request:** "Add a feature to export user data"

**What LLMs do wrong — silent assumptions:**

```python
def export_users(format='json'):
    users = User.query.all()
    if format == 'json':
        with open('users.json', 'w') as f:
            json.dump([u.to_dict() for u in users], f)
    elif format == 'csv':
        with open('users.csv', 'w') as f:
            writer = csv.DictWriter(f, fieldnames=['id', 'email', 'name'])
            writer.writeheader()
            writer.writerows([u.to_dict() for u in users])
    return f"Exported {len(users)} users"
```

Problems: assumed export scope, file location, which fields, CSV fieldnames — none of it asked for.

**What should happen — surface assumptions:**

```
Before implementing, I need to clarify:

1. Scope: Export all users or filtered subset? (privacy implications)
2. Format: Download file in browser? Background job? API endpoint?
3. Fields: Which user fields? (some might be sensitive)
4. Volume: How many users typically? (affects approach)

Simplest approach: API endpoint returning paginated JSON.
What's your preference?
```

### Example 2: Multiple Interpretations

**User Request:** "Make the search faster"

**What LLMs do wrong — pick silently:**

```python
@lru_cache(maxsize=1000)
async def search(query: str) -> List[Result]:
    # 200 lines of optimization nobody asked to specify
```

**What should happen — present options:**

```
"Make the search faster" could mean:

1. Faster response time (< 100ms instead of 500ms)
   - Add database indexes, cache frequent queries
   - Estimated effort: 2 hours

2. Handle more concurrent searches (throughput)
   - Add async/await, connection pooling
   - Estimated effort: 4 hours

3. Faster perceived speed (UX)
   - Show partial results immediately, progressive loading
   - Estimated effort: 3 hours

Current search takes ~500ms for typical queries.
Which aspect matters most for your use case?
```

---

## 2. Simplicity First

### Example 1: Over-abstraction

**User Request:** "Add a function to calculate discount"

**What LLMs do wrong — overengineered:**

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass

class DiscountStrategy(ABC):
    @abstractmethod
    def calculate(self, amount: float) -> float: pass

class PercentageDiscount(DiscountStrategy):
    def __init__(self, percentage: float):
        self.percentage = percentage
    def calculate(self, amount: float) -> float:
        return amount * (self.percentage / 100)

@dataclass
class DiscountConfig:
    strategy: DiscountStrategy
    min_purchase: float = 0.0
    max_discount: float = float('inf')

class DiscountCalculator:
    def __init__(self, config: DiscountConfig):
        self.config = config
    def apply_discount(self, amount: float) -> float:
        if amount < self.config.min_purchase:
            return 0
        discount = self.config.strategy.calculate(amount)
        return min(discount, self.config.max_discount)
# Usage requires 30+ lines of setup for a simple calculation
```

**What should happen — simple:**

```python
def calculate_discount(amount: float, percent: float) -> float:
    return amount * (percent / 100)

discount = calculate_discount(100.0, 10.0)  # $10 off
```

Add complexity only when you actually need multiple discount types.

### Example 2: Speculative Features

**User Request:** "Save user preferences to database"

**What LLMs do wrong — unnecessary features:**

```python
class PreferenceManager:
    def __init__(self, db, cache=None, validator=None):
        self.db = db
        self.cache = cache or InMemoryCache()
        self.validator = validator or DefaultValidator()

    def save(self, user_id, prefs, merge=True, validate=True, notify=False):
        if validate:
            errors = self.validator.validate(prefs)
            if errors:
                raise ValidationError(errors)
        existing = self.get(user_id) if merge else {}
        if merge:
            existing.update(prefs)
            prefs = existing
        self.db.execute("UPDATE users SET preferences = ? WHERE id = ?",
                        (json.dumps(prefs), user_id))
        if self.cache:
            self.cache.set(f'prefs:{user_id}', prefs)
        if notify:
            self.notify_preference_change(user_id, prefs)  # 30 more lines
        return True
```

**What should happen — just what was asked:**

```python
def save_preferences(db, user_id, preferences):
    db.execute('UPDATE users SET preferences = ? WHERE id = ?',
               (json.dumps(preferences), user_id))
```

Add caching, validation, merging only when the requirement actually emerges.

---

## 3. Surgical Changes

### Example 1: Drive-by Refactoring

**User Request:** "Fix the bug where empty emails crash the validator"

**What LLMs do wrong — too much:**

```diff
  def validate_user(user_data):
-     # Check email format
-     if not user_data.get('email'):
+     """Validate user data."""
+     email = user_data.get('email', '').strip()
+     if not email:
          raise ValueError("Email required")
-     if '@' not in user_data['email']:
+     if '@' not in email or '.' not in email.split('@')[1]:
          raise ValueError("Invalid email")
-     if not user_data.get('username'):
+     username = user_data.get('username', '').strip()
+     if not username:
          raise ValueError("Username required")
+     if len(username) < 3:
+         raise ValueError("Username too short")
      return True
```

Problems: improved email validation beyond the bug, added username validation nobody asked for, changed comments, added docstring.

**What should happen — surgical:**

```diff
  def validate_user(user_data):
      # Check email format
-     if not user_data.get('email'):
+     email = user_data.get('email', '')
+     if not email or not email.strip():
          raise ValueError("Email required")
-     if '@' not in user_data['email']:
+     if '@' not in email:
          raise ValueError("Invalid email")
      # Check username
      if not user_data.get('username'):
          raise ValueError("Username required")
      return True
```

Only changed the specific lines that fix empty email handling.

### Example 2: Style Drift

**User Request:** "Add logging to the upload function"

**What LLMs do wrong — reformat everything:**

```diff
- def upload_file(file_path, destination):
+ def upload_file(file_path: str, destination: str) -> bool:
+     """Upload file to destination with logging."""
+     logger.info(f"Uploading {file_path} to {destination}")
      try:
-         with open(file_path, 'rb') as f:
+         with open(file_path, "rb") as f:
              data = f.read()
-         response = requests.post(destination, files={'file': data})
-         if response.status_code == 200:
-             return True
-         else:
-             return False
+         response = requests.post(destination, files={"file": data})
+         success = response.status_code == 200
+         logger.info(f"Upload {'successful' if success else 'failed'}: {file_path}")
+         return success
      except Exception as e:
-         print(f"Error: {e}")
+         logger.exception(f"Upload error for {file_path}: {e}")
          return False
```

Problems: changed quote style, added type hints, added docstring, reformatted whitespace.

**What should happen — match existing style:**

```diff
+ import logging
+ logger = logging.getLogger(__name__)
+
  def upload_file(file_path, destination):
+     logger.info(f'Starting upload: {file_path}')
      try:
          with open(file_path, 'rb') as f:
              data = f.read()
          response = requests.post(destination, files={'file': data})
          if response.status_code == 200:
+             logger.info(f'Upload successful: {file_path}')
              return True
          else:
+             logger.error(f'Upload failed: {file_path}, status={response.status_code}')
              return False
      except Exception as e:
-         print(f"Error: {e}")
+         logger.exception(f'Upload error: {file_path}')
          return False
```

Matched: single quotes, no type hints, existing boolean pattern, spacing style.

---

## 4. Goal-Driven Execution

### Example 1: Vague vs. Verifiable

**User Request:** "Fix the authentication system"

**What LLMs do wrong — vague approach:**

```
I'll fix the authentication system by reviewing the code,
identifying issues, making improvements, and testing the changes.
[Proceeds to make changes without clear success criteria]
```

**What should happen — verifiable goals:**

```
To "fix authentication" I need to define success criteria.
What specific issue are we solving?

If the issue is "users stay logged in after password change":

Plan:
1. Write test: Change password → verify old session invalidated
   Verify: Test fails (reproduces bug)

2. Implement: Invalidate sessions on password change
   Verify: Test passes

3. Check edge cases: Multiple active sessions, concurrent changes
   Verify: Additional tests pass

4. Verify no regression: Existing auth tests still pass
   Verify: Full test suite green

What's the specific auth issue you're seeing?
```

### Example 2: Multi-Step with Verification

**User Request:** "Add rate limiting to the API"

**What LLMs do wrong — all at once:**

```python
# Implements full rate limiting with Redis, multiple strategies,
# configuration system, and monitoring in one 300-line commit
# No clear verification steps
```

**What should happen — incremental with verification:**

```
Plan for rate limiting:

1. Add basic in-memory rate limiting (single endpoint)
   Verify: 100 requests → first 10 succeed, rest get 429

2. Extract to middleware (apply to all endpoints)
   Verify: Rate limits apply to /users and /posts; existing tests pass

3. Add Redis backend (for multi-server)
   Verify: Rate limit persists across app restarts; two instances share counter

4. Add configuration (rates per endpoint)
   Verify: /search allows 10/min, /users allows 100/min

Each step is independently verifiable and deployable.
Start with step 1?
```

### Example 3: Test-First Verification

**User Request:** "The sorting breaks when there are duplicate scores"

**What LLMs do wrong — fix without reproducing:**

```python
def sort_scores(scores):
    return sorted(scores, key=lambda x: (-x['score'], x['name']))
# Immediately changes sort logic without confirming the bug
```

**What should happen — reproduce first:**

```python
# 1. Write a test that reproduces the issue
def test_sort_with_duplicate_scores():
    scores = [
        {'name': 'Alice', 'score': 100},
        {'name': 'Bob',   'score': 100},
        {'name': 'Charlie', 'score': 90},
    ]
    result = sort_scores(scores)
    assert result[0]['score'] == 100
    assert result[1]['score'] == 100
    assert result[2]['score'] == 90
# Verify: test fails with inconsistent ordering

# 2. Now fix with stable sort
def sort_scores(scores):
    return sorted(scores, key=lambda x: (-x['score'], x['name']))
# Verify: test passes consistently
```

---

## Anti-Patterns Summary

| Principle | Anti-Pattern | Fix |
|-----------|-------------|-----|
| Think Before Coding | Silently assumes file format, fields, scope | List assumptions explicitly, ask for clarification |
| Simplicity First | Strategy pattern for single discount calculation | One function until complexity is actually needed |
| Surgical Changes | Reformats quotes, adds type hints while fixing bug | Only change lines that fix the reported issue |
| Goal-Driven | "I'll review and improve the code" | "Write test for bug X → make it pass → verify no regressions" |

**Good code is code that solves today's problem simply, not tomorrow's problem prematurely.**
