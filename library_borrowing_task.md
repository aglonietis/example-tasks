
# C2 - Phase one - PHP API Development

## Competitor Information
The final website has to be available at `http://<hostname>/module-b/phase1`.  
For example, the books endpoint is reachable at `http://<hostname>/module-b/phase1/api/v1/books`.

Tests are provided that check for correct implementation of the following requirements.  
To run the tests, execute `composer test`. If the API is available under a different URL for development, it can be overwritten with the environment variable `BASE_URL` (e.g., `BASE_URL=http://localhost:8080/ composer test`). Do not include `/api/v1` in the path in this case.

Please note that when running the tests, the database gets reset to ensure having exactly the same state for every test run. So your changes to the database will be lost.

---

## Database design
A database dump is already provided to you. Import it into your database and use the provided schema and data for developing the application.

- Database name: `library`  
- MySQL user: `competitor` (all privileges on this database)

---

## API

To allow parallel development of a mobile app, you are provided an API specification and it is therefore important that it is exactly followed. Provided tests will verify a correct implementation.

**General information:**
- The response body contains example data below. In the application, dynamic data from the database must be used.
- Placeholder parameters in the URL are marked with curly braces (e.g. `{book-id}`).
- The order of properties in objects does not matter, but the order in an array does.
- If not specified otherwise, the `Content-Type` header of a response is always `application/json`.

### Entities
```
Book: { id, title, author, published_year, category }
Shelf: { id, name, order }
Copy: numbered per shelf (1..total)
Reservation: temporary hold for one or more copies (per book)
Loan: confirmed borrowing created from a reservation; grouped into a checkout
```

---

## Endpoints

### GET /api/v1/books
Returns a list of all available books and all their shelves with copy availability.

**Sorting:** Books by `title` ASC; shelves by `order` ASC.

**200 Response**
```json
{
  "books": [
    {
      "id": 1,
      "title": "Clean Code",
      "author": "Robert C. Martin",
      "published_year": 2008,
      "category": "Software",
      "shelves": [
        {
          "id": 1,
          "name": "Shelf A",
          "copies": {
            "total": 5,
            "unavailable": [1, 2]
          }
        }
      ]
    }
  ]
}
```

---

### GET /api/v1/books/{book-id}
Returns a single book with shelf/copy availability.

**Path parameters**
- `book-id` (int) – Database ID of the book.

**200 Response**
```json
{
  "book": {
    "id": 1,
    "title": "Clean Code",
    "author": "Robert C. Martin",
    "published_year": 2008,
    "category": "Software",
    "shelves": [
      {
        "id": 1,
        "name": "Shelf A",
        "copies": {
          "total": 5,
          "unavailable": [1, 2]
        }
      }
    ]
  }
}
```

**404 Response**
```json
{ "error": "A book with this ID does not exist" }
```

---

### GET /api/v1/books/{book-id}/availability
Returns availability information for the given book and contains all shelves and their capacity.

**Path parameters**
- `book-id` (int)

**200 Response**
```json
{
  "shelves": [
    {
      "id": 1,
      "name": "Shelf A",
      "copies": {
        "total": 5,
        "unavailable": [1, 2, 4]
      }
    }
  ]
}
```

**404 Response**
```json
{ "error": "A book with this ID does not exist" }
```

---

### POST /api/v1/books/{book-id}/reservation
Creates or replaces a reservation for one or more copies so they cannot be borrowed by someone else.

**Body**
```json
{
  "reservation_token": "...",
  "reservations": [
    { "shelf": 1, "copy": 3 },
    { "shelf": 1, "copy": 5 }
  ],
  "duration": 300
}
```

**201 Response**
```json
{
  "reserved": true,
  "reservation_token": "...",
  "reserved_until": "2025-09-24T10:25:46Z"
}
```

**403 Response**
```json
{ "error": "Invalid reservation token" }
```

**404 Response**
```json
{ "error": "A book or shelf with this ID does not exist" }
```

**422 Response**
```json
{
  "error": "Validation failed",
  "fields": {
    "reservations": "Copy 3 on shelf 1 is already taken.",
    "duration": "The duration must be between 1 and 300."
  }
}
```

---

### POST /api/v1/books/{book-id}/borrow
Promotes a reservation to one or more loans for the specified book. All created loans are grouped into a checkout.

**Body**
```json
{
  "reservation_token": "...",
  "name": "John Doe",
  "address": "Bahnhofstrasse 15",
  "city": "Graz",
  "zip": "8010",
  "country": "Austria"
}
```

**201 Response**
```json
{
  "loans": [
    {
      "id": 1,
      "code": "QVLJTWK4Y7",
      "name": "John Doe",
      "created_at": "2025-09-24T11:02:20Z",
      "due_at": "2025-10-08T23:59:59Z",
      "shelf": { "id": 1, "name": "Shelf A" },
      "copy": 5,
      "book": {
        "id": 1,
        "title": "Clean Code",
        "author": "Robert C. Martin"
      }
    }
  ]
}
```

**401 Response**
```json
{ "error": "Unauthorized" }
```

**404 Response**
```json
{ "error": "A book with this ID does not exist" }
```

**422 Response**
```json
{
  "error": "Validation failed",
  "fields": {
    "reservation_token": "The reservation token field is required.",
    "name": "The name field is required.",
    "city": "The city must be a string.",
    "zip": "The zip must be a string."
  }
}
```

---

### POST /api/v1/loans
Returns all loans belonging to the same checkout when the user provides their name and one loan code.

**Body**
```json
{ "code": "QVLJTWK4Y7", "name": "John Doe" }
```

**200 Response**
```json
{
  "loans": [
    {
      "id": 1,
      "code": "QVLJTWK4Y7",
      "name": "John Doe",
      "created_at": "2025-09-24T11:02:20Z",
      "due_at": "2025-10-08T23:59:59Z",
      "shelf": { "id": 1, "name": "Shelf A" },
      "copy": 5,
      "book": { "id": 1, "title": "Clean Code", "author": "Robert C. Martin" }
    }
  ]
}
```

**401 Response**
```json
{ "error": "Unauthorized" }
```

---

### POST /api/v1/loans/{loan-id}/return
Returns (cancels) the specified loan and frees the copy.

**Body**
```json
{ "code": "QVLJTWK4Y7", "name": "John Doe" }
```

**204 Response**
_No body_

**401 Response**
```json
{ "error": "Unauthorized" }
```

**404 Response**
```json
{ "error": "A loan with this ID does not exist" }
```

---
