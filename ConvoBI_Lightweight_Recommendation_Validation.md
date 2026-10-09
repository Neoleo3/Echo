# ConvoBI Lightweight Recommendation Validation Architecture

You are a Principal Engineer and Enterprise Architect reviewing the ConvoBI recommendation engine.

## Current Problem

The recommendation engine currently validates questions by calling:

```python
invoke_convobi_agent()
```

This executes the entire ConvoBI workflow:

```text
Question
    ↓
LangGraph Agent
    ↓
SQL Generation
    ↓
SQL Validation
    ↓
SQL Execution
    ↓
LLM Summary
    ↓
Visualization Generation
    ↓
Response Parsing
```

This is expensive and consumes unnecessary tokens.

---

## Business Requirement

The recommendation endpoint does NOT need:

- Final answer generation
- Summaries
- Visualizations
- Analytics reports
- Natural language responses

The recommendation endpoint only needs to prove:

```text
The question works.
```

Meaning:

```text
Question
    ↓
SQL generated successfully
    ↓
SQL validated successfully
    ↓
SQL executed successfully
    ↓
Returned rows > 0
```

If all of the above are true, the recommendation is valid.

---

## Desired Behavior

### If question_count = 1

Generate candidate questions.

Example:

```text
Candidate 1
Candidate 2
Candidate 3
Candidate 4
Candidate 5
```

Validation flow:

```text
Candidate 1
    ↓
Generate SQL
    ↓
Validate SQL
    ↓
Execute SQL
    ↓
Rows returned?
```

If YES:

```text
STOP
Return Candidate 1
```

Do NOT process remaining candidates.

The first validated question should be returned.

---

### If question_count > 1

Example:

```json
{
  "question_count": 5
}
```

Flow:

```text
Generate candidate questions
        ↓
Validate Candidate 1
        ↓
Valid?
        ↓
Add to results

Validate Candidate 2
        ↓
Valid?
        ↓
Add to results
```

Continue until:

```python
len(valid_questions) == question_count
```

Then stop.

---

## Important Architecture Constraint

Do NOT create a second SQL generation implementation.

ConvoBI already has:

- SQL generation
- SQL validation
- Schema understanding

Reuse existing components.

There must be only ONE source of truth for SQL generation.

If ConvoBI SQL generation changes, recommendation validation should automatically use the same logic.

---

## Architecture Decision Required

Review the existing codebase and determine:

### Option A

Keep implementation inside:

```text
apps/convobi/main.py
```

Create lightweight helper functions such as:

```python
generate_sql_for_question()
validate_recommendation_question()
execute_validation_query()
```

---

### Option B

Create a new file:

```text
apps/convobi/recommendation_validator.py
```

that reuses existing ConvoBI SQL-generation components.

---

## Evaluation Criteria

Provide a recommendation based on:

- Maintainability
- Code duplication
- Coupling
- Performance
- Production supportability
- Testing complexity
- Future change management

---

## Deliverables

Provide:

1. Recommended architecture.
2. Whether a new file should be created.
3. Exact module structure.
4. Exact function structure.
5. End-to-end request flow.
6. Token cost comparison:
   - Current implementation
   - Proposed implementation
7. Production risks.
8. Detailed code changes.
9. Migration plan.
10. Final recommendation.

Goal:

Build the lightest-weight recommendation validation mechanism possible while ensuring that every returned recommendation:

```text
Generates SQL
Executes successfully
Returns real data
```

and avoid running the full ConvoBI workflow whenever possible.
