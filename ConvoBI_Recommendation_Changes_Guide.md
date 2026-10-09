# ConvoBI lightweight recommendation changes

## Goal

Generate a single batch of 10 question/SQL pairs in one LLM call, then validate candidates sequentially using ConvoBI's existing schema discovery and SQL schema validator. Stop validating as soon as the API's requested number of answerable questions has been found. Do **not** call `invoke_convobi_agent()` for recommendation validation, and do not regenerate the batch when candidates fail.

The validator treats a candidate as answerable for recommendation purposes only if its SQL passes the read-only/single-query guard, passes `validate_sql_against_schema()`, executes successfully, and returns at least one row. This is a runtime check, not a guarantee that the question's business interpretation is perfect or that data/permissions will remain unchanged later.

## Files in this package

- `recommendation_validator.py` — new module to place at `convobi/recommendation_validator.py`.
- This guide — exact manual changes for `main.py` and `routes.py`.

No production `main.py` or `routes.py` is included or overwritten by this package.

## 1. `main.py` — remove only recommendation-specific code

Do not reformat or otherwise modify `main.py`. Remove only the recommendation-only block currently beginning at `RECOMMENDATION_CACHE_DIR = Path(...)` and ending at the end of `generate_recommended_questions(...)`, immediately before the next unrelated section (`# ---------- ...` or the next non-recommendation declaration).

That block contains recommendation-only constants and helpers such as:

- `RECOMMENDATION_CACHE_DIR` and its `.mkdir(...)` call (only if no other code uses this cache directory)
- `MAX_RECOMMEND_CONTEXT_CHARS`
- `MAX_RECOMMENDATION_COUNT`
- `RECOMMENDATION_CANDIDATE_MULTIPLIER`
- recommendation-only validation time/row constants
- `ValidatedRecommendation`
- `build_recommendation_context()`
- `build_recommendation_cache_key()`
- `_normalize_recommendation_question()`
- `_recommendation_is_duplicate()`
- `_recommendation_score()`
- `_generate_recommendation_candidates()`
- `generate_recommended_questions()`

Before deleting the cache setup, search `main.py` for every use of `RECOMMENDATION_CACHE_DIR`. If any non-recommendation code uses it, keep the shared dependency and remove only recommendation-specific uses. Do not delete or change `get_schema_info()`, `validate_sql_against_schema()`, `_get_convobi_llm_async()`, connection management, SQL safety tools, agent construction, or `invoke_convobi_agent()`.

## 2. `routes.py` — change only the recommendation import

In the `from .main import (...)` import list, remove only this entry:

```python
generate_recommended_questions,
```

Add a separate import near the other local imports:

```python
from .recommendation_validator import generate_recommended_questions
```

Do not change the `/recommended-questions` endpoint contract or its call. It already calls `generate_recommended_questions(user_id=..., db=..., question_count=..., force_refresh=...)`, which matches the new module.

## 3. Why the schema is sent to the LLM

The validator sends the schema discovered by ConvoBI, including table names, column names/types, row counts, primary/foreign keys, relationships, and any meanings/examples available in the schema metadata. It does **not** send every row of database data: doing so would increase token usage and could expose sensitive records. The database itself is queried only to verify each candidate's SQL.

## 4. Token and query behavior

- One LLM call generates 10 candidates per request.
- No LLM call is made per candidate during validation.
- Candidates are validated in order; once `question_count` valid questions have been collected, remaining candidates are not executed.
- If fewer than the requested count pass among the 10 candidates, the endpoint returns only the validated subset. It does not make another generation call.
- The request model's existing maximum of 8 remains unchanged.
- A candidate's SQL is kept alongside its question in `validated_recommendations`; if you later want click-time execution to reuse exactly the validated SQL, the route/API response contract must be deliberately extended. This package does not alter the response contract.

## 5. Validation before deployment

1. Run `python -m py_compile convobi/recommendation_validator.py convobi/main.py convobi/routes.py`.
2. Confirm ordinary `/query` requests still use the existing full agent and are unchanged.
3. Test recommendation requests for counts 1, 2, and 8 against a staging database.
4. Confirm invalid SQL, SQL execution errors, and zero-row results are rejected.
5. Confirm a successful candidate stops further database validation when the requested count has been reached.
6. Use a database account with read-only permissions where supported. Application-level SQL checks are not a substitute for database permissions.

## Important limitation

A query that executes and returns rows proves that it returned data at validation time. It does not mathematically prove semantic correctness or guarantee that a later execution will succeed if the database changes. The best practical design is to validate the exact SQL paired with the question and, if click-time behavior must be identical, execute that same approved SQL rather than asking the LLM to regenerate it.
