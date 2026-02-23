---
toc: True
layout: post
data: tools
title: Sloane Skill B 
categories: ['DevOps']
permalink: /skill_b_review
breadcrumb: True
author: Sloane
---

# AP CSP Create Performance Task — Skill B Written Response
### Performance Ratings & Reflection Web Application

---

## Row 3: Program Purpose and Function

**Program Purpose:**  
This program enables students to anonymously self-rate their preparedness (1–5) after completing English skill-building activities — specifically media bias analysis, thesis writing, and citation comprehension. The broader goal is to surface class-wide trends in student confidence so both teachers and students can identify gaps and act on them.

**Program Function:**  
When a user selects a 1–5 rating and submits the form, the frontend sends a POST request to the Flask REST API. The backend authenticates the user via JWT token, saves the rating to a database, computes the live class average, and returns personalized feedback comparing the user's rating to their peers.

**Input:** A numeric rating (1–5) selected by the user and submitted via an HTML form.

**Output:** A JSON object containing `your_rating`, `average_rating`, `status` (underprepared / average / overprepared), a personalized `message`, and the `performance_id` of the new record.

```json
{
  "your_rating": 2,
  "average_rating": 3.7,
  "status": "underprepared",
  "message": "The majority felt 3.7/5 prepared. You rated 2/5 - there's room to grow!",
  "performance_id": 42,
  "username": "john_doe"
}
```

---

## Row 4: Data Abstraction — Using a List/Collection

The program stores every student's submission as a row in a SQL database, retrieved and represented as a **list of dictionaries** in Python. Each dictionary holds the fields: `id`, `rating`, `user_id`, `username`, and `timestamp`.

```python
def getPerformances():
    items = Performance.list_all(limit=1000)
    return [p.read() for p in items]  # list of dicts
```

Each element looks like:

```python
{
    "id": 12,
    "rating": 4,
    "user_id": 7,
    "username": "jane_smith",
    "timestamp": "2025-05-01T10:32:00"
}
```

This list is the **central data structure** of the program. Every statistic — average, distribution, most common rating — is derived by querying this collection. The frontend admin dashboard fetches this list and dynamically renders each row into an HTML table with edit/delete controls.

---

## Row 5: Managing Complexity with the List

Without the list abstraction, the program would need a separate named variable for every student submission — completely unworkable at scale. Instead, the entire collection is passed through a single aggregation query:

```python
def getAverageRating():
    from sqlalchemy import func
    avg = db.session.query(func.avg(Performance.rating)).scalar()
    return round(float(avg), 1) if avg else 3.0
```

The list also powers the full statistics endpoint, which summarizes all submissions in one call:

```python
class _ReadStats(Resource):
    def get(self):
        return {
            'count': countPerformances(),      # total submissions
            'average': getAverageRating(),     # mean across all ratings
            'distribution': getRatingDistribution(),  # {1:4, 2:7, 3:12, ...}
            'most_common': getMostCommonRating()  # mode rating
        }
```

The `getRatingDistribution()` function iterates over the collection and tallies how many students gave each rating:

```python
def getRatingDistribution():
    distribution = {1: 0, 2: 0, 3: 0, 4: 0, 5: 0}
    rows = db.session.query(
        Performance.rating, func.count(Performance.rating)
    ).group_by(Performance.rating).all()

    for rating, count in rows:
        if rating in distribution:
            distribution[int(rating)] = int(count)
    return distribution
```

Without the collection, computing any of these statistics would require writing separate hardcoded logic for every student — the list reduces that to a single loop or SQL aggregate, directly **managing complexity**.

---

## Row 6: Procedural Abstraction

The `addPerformance` function is a **named procedure with parameters** that abstracts away all database creation logic from the API layer. It accepts `rating` and `user_id` as inputs, validates them, creates the model instance, commits to the database, and returns a formatted dictionary.

```python
def addPerformance(rating, user_id):
    if user_id is None:
        raise ValueError("user_id is required")
    
    try:
        rating = int(rating)
    except Exception:
        raise ValueError("rating must be an integer")
    
    if rating not in [1, 2, 3, 4, 5]:
        raise ValueError("rating must be between 1 and 5")

    perf = Performance(rating=rating, user_id=user_id, timestamp=datetime.utcnow())
    perf.create()
    return perf.read()
```

**Called inside the API endpoint:**

```python
new_performance = addPerformance(
    rating=rating,
    user_id=current_user.id
)
```

This procedure **contributes to overall program functionality** because the API endpoint only needs to pass two values — it never needs to know about SQLAlchemy sessions, model constructors, or commit logic. If the database implementation changes, only `addPerformance` needs to be updated, not every endpoint that creates a performance record.

A second named procedure, `getAverageRating()`, is called immediately after to compute the live class average used in the status comparison. Both procedures work together in sequence to produce the final response.

---

## Row 7: Algorithm Implementation (Sequencing, Selection, Iteration)

The `post()` method of the `_Submit` resource contains all three required algorithmic elements.

### Sequencing
Steps execute in a specific, meaningful order — changing the order would break the program:

```python
# Step 1: Authenticate and extract user
current_user = g.current_user

# Step 2: Parse and validate input
data = request.get_json()
rating = int(data.get('rating'))

# Step 3: Save to database
new_performance = addPerformance(rating=rating, user_id=current_user.id)

# Step 4: Calculate live class average AFTER saving (includes this submission)
avg_rating = getAverageRating()

# Step 5: Determine status and build response
if rating < avg_rating:
    status = "underprepared"
    ...
```

If step 4 ran before step 3, this student's own rating wouldn't be included in the average — sequencing matters here.

### Selection
Three mutually exclusive branches handle every possible outcome:

```python
if rating < avg_rating:
    status = "underprepared"
    message = f"The majority felt {avg_rating}/5 prepared. You rated {rating}/5 - there's room to grow!"
elif rating > avg_rating:
    status = "overprepared"
    message = f"Great! You rated {rating}/5 while most felt {avg_rating}/5. You're well-prepared!"
else:
    status = "average"
    message = f"You're right on track! Most people also felt {avg_rating}/5 prepared."
```

There is also input validation using nested selection before any data is stored:

```python
if rating is None:
    return {'error': 'Rating field is required'}, 400

if rating not in [1, 2, 3, 4, 5]:
    return {'error': 'Invalid rating. Must be 1-5.'}, 400
```

### Iteration
The `getRatingDistribution()` function iterates over every unique rating value returned by the database to build the frequency map:

```python
for rating, count in rows:
    if rating in distribution:
        distribution[int(rating)] = int(count)
```

The frontend also uses iteration to dynamically render every performance record returned by the API into the admin table:

```javascript
performances.forEach(perf => {
    const stars = '★'.repeat(perf.rating) + '☆'.repeat(5 - perf.rating);
    const row = `<tr>
        <td>${perf.id}</td>
        <td>${perf.rating}/5 ${stars}</td>
        <td>${perf.username}</td>
    </tr>`;
    tbody.innerHTML += row;
});
```

---

## Row 8: Testing — Two Different Calls

The `_Submit.post()` procedure was tested with two distinct inputs to verify the selection logic produces correct, different outputs.

---

### Test Call 1 — Rating BELOW the class average

**Condition:** `rating = 1`, class average = `3.5`  
**Selection branch hit:** `if rating < avg_rating` → `True`  
**Expected output:**

```json
{
  "your_rating": 1,
  "average_rating": 3.5,
  "status": "underprepared",
  "message": "The majority felt 3.5/5 prepared. You rated 1/5 - there's room to grow!"
}
```

**Result:** Correct — the `underprepared` branch fires, the message encourages improvement, and the correct ratings appear in the response.

---

### Test Call 2 — Rating ABOVE the class average

**Condition:** `rating = 5`, class average = `3.5`  
**Selection branch hit:** `elif rating > avg_rating` → `True`  
**Expected output:**

```json
{
  "your_rating": 5,
  "average_rating": 3.5,
  "status": "overprepared",
  "message": "Great! You rated 5/5 while most felt 3.5/5. You're well-prepared!"
}
```

**Result:** Correct — the `overprepared` branch fires, producing a different `status` and a different `message` than Call 1.

---

### Why the Outputs Differ

The two calls produce different results because the `if/elif/else` chain compares `rating` against `avg_rating`. Call 1 satisfies `rating < avg_rating` (1 < 3.5), while Call 2 satisfies `rating > avg_rating` (5 > 3.5). Each branch assigns a unique `status` string and a unique `message` string, so the final JSON response is different in both the `status` and `message` fields. This confirms the algorithm correctly distinguishes all three cases.

---

## Summary Table

| CB Row | Requirement | Where It Appears in Code |
|--------|-------------|--------------------------|
| 3 | Program purpose + input/output | HTML form → `/api/performance/submit` → JSON response |
| 4 | List/collection storing data | `getPerformances()` returns `list` of `dict` objects |
| 5 | List manages complexity | `getAverageRating()`, `getRatingDistribution()` aggregate entire table |
| 6 | Named procedure with parameters | `addPerformance(rating, user_id)` in `performances.py` |
| 7 | Sequencing + selection + iteration | `_Submit.post()` in `performance.py` + `getRatingDistribution()` |
| 8 | Two test calls, different conditions/outputs | `rating=1` vs `rating=5` against `avg=3.5` |