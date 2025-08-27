
# C2 – Phase two – Frontend Development (Library)

After completing the REST API, implement a frontend that **consumes the Library API**. Users must be able to:

- Discover books with filters  
- Reserve copies temporarily and then borrow them (loans)  
- Retrieve their loans by code + name  
- Return a specific loan  

A finished backend (or mocked responses) will be used for evaluation; your own phase one result is not required. Part of the evaluation will use **Cypress** with mocked REST calls; selectors/text must match this spec. The final site must be available at `http://<hostname>/module-b/phase2`. To run tests:  
`npm start -- --config baseUrl=http://<hostname>/module-b/phase2` (or `http://localhost:4200`).

---

## Competitor Information
- Deploy website at: `http://<hostname>/module-b/phase2`  
- Upload source files to: `/var/www/module-b/phase2-src`  
- Tests use Cypress and will mock REST calls in places; ensure selectors/text below exist and are unique per page.

---

## Non-Functional Requirements
- You may use any frontend tech; **implement an SPA**.  
- Must run in the provided Chrome and Firefox versions.  
- **Minimize API traffic** (cache results/state; books/availability don’t change often).  
- **Accessibility**: follow axe-core rules; all interactive elements must be reachable with **Tab** and actionable with **Enter**.

---

## Page Structure (all pages)
- **Header**: App title `EuroSkills Library`.  
- **Page Title** below header.  
- **Loans Button**: “Get Loans” (to view already borrowed items).  
**Selectors** (exact text/roles):  
- App Title: text contains `EuroSkills Library`  
- Loans hint: text contains `Already borrowed?`  
- Loans button: text contains `Get Loans`  

---

## API you must consume
Use the Library API you defined earlier (paths relative to `…/api/v1`):

- `GET /books` — list books with shelves & copy availability  
- `GET /books/{book-id}` — single book with availability  
- `GET /books/{book-id}/availability` — shelves with `{ total, unavailable }`  
- `POST /books/{book-id}/reservation` — create/replace reservation; returns `reservation_token` + `reserved_until`  
- `POST /books/{book-id}/borrow` — confirm reservation → creates **loans[]** (each with 10-char `code`)  
- `POST /loans` — lookup all loans in the same checkout by `{ code, name }`  
- `POST /loans/{loan-id}/return` — return a single loan

Follow request/response shapes exactly as in the backend task.

---

## Pages

### 1) Landing Page — Discover books
**Route**: `/module-b/phase2/`  
**Purpose**: Display all **books** and allow discovery/filtering.

**Card content (per book):**
- Title  
- Author  
- Category  
- Oldest publication year visible as `Published: yyyy`  
- Availability summary: total copies vs. unavailable (computed from shelves)  

**Filters:**
- **Author** selector (distinct authors present in the dataset; default `All Authors`)  
- **Category** selector (distinct categories; default `All Categories`)  
- **Search** input (substring on title)  
**Rules:**
- Filters apply immediately (no “Apply” button).  
- Show a **Clear** button only when any filter is active.  
- If nothing matches: text `No books match the current filter criteria.`  

**API requirement**: Fetch **once** on app load (`GET /books`) and derive filters/options client-side. Cache in memory to minimize repeated calls.

**Selectors**
- Page Title: text contains `Discover books`  
- Clear button: text contains `Clear`  
- Empty state: text contains `No books match the current filter criteria.`

---

### 2) Book Detail — Availability & Reservation
**Route**: `/module-b/phase2/books/{book-id}`  
**Purpose**: Show **shelves** for the selected book and allow **copy reservation**.

**Availability board**:
- For each **shelf**: show shelf name and a **grid of copy slots** numbered `1..total`.  
- **Unavailable** copies must have a distinct style and be non-interactive.  
- **Available** copies are clickable to **toggle inclusion** in the current reservation.  
- Each click **updates the reservation** via `POST /books/{book-id}/reservation`.

**Selected Copies panel**:
- Title: `Selected Copies`  
- If none: `No copies selected. Click on a copy to make a reservation.`  
- Otherwise list items as: `Shelf: {shelfName}, Copy: {copy}`

**Timer**:
- After first successful reservation response, show `Your copies expire in mm:ss` counting down from `reserved_until`.  
- On expiry: show alert `Your reservation expired. It has been cancelled.` and clear selection.

**Proceed**:
- Button `Enter Borrower Details` → navigates to Borrow form.

**Selectors (CSS/text):**
- Copy slot: `.copy` with `[data-shelf="<shelfId>"]` and `[data-copy="<copyNumber>"]`  
- Available copy: `.copy-available`  
- Unavailable copy: `.copy-unavailable`  
- Selected copy: `.copy-selected`  
- Selected panel title: text `Selected Copies`  
- Empty-selection text: `No copies selected. Click on a copy to make a reservation.`  
- Countdown text: `Your copies expire in` (format `mm:ss`)  
- Expired message: `Your reservation expired. It has been cancelled.`  
- Proceed button: text `Enter Borrower Details`  

---

### 3) Borrow Form — Confirm borrowing (create loans)
**Route**: `/module-b/phase2/books/{book-id}/borrow`  
**Purpose**: Submit borrower details to **promote reservation → loans** via `POST /books/{book-id}/borrow`.

**Form fields (all required):**
- Name  
- Address  
- ZIP Code  
- City  
- Country (dropdown populated from `resources/countries.csv`)  

**Rules:**
- Keep the **Selected Copies** panel visible while filling the form.  
- Form can submit only when all fields are valid.  
- If user clicks **Book** with invalid fields, mark invalid inputs with a **red border** and block submission.  
- On success, show the **Loans page** immediately with data returned by the API (`loans[]`).  

**Selectors**
- Form title: `Please enter your details`  
- Field labels: `Name`, `Address`, `ZIP Code`, `City`, `Country`  
- Invalid fields: `.validated :invalid`  
- Submit: button text `Book`  
- Change selection: button text `Change Copies`  

---

### 4) Loans Page — Show current loans
**Route**: `/module-b/phase2/loans`  
**Purpose**: Display all loans from the user’s **checkout**.

**Display each loan card with:**
- Loan code (10-char)  
- Book title & author  
- Shelf name and copy number  
- Created date/time, and **Due at** date/time  

**Actions**:
- **Return** button per loan → calls `POST /loans/{loan-id}/return`.  
- **Back to Catalog** link/button.

---

### 5) Loan Retrieval (later access)
**Route**: `/module-b/phase2/loans/retrieve`  
**Purpose**: Let a user retrieve all loans later using `POST /loans` with `{ code, name }`.

**Form**:
- Name  
- One Loan Code  
**On success**: navigate to **Loans Page** with fetched loans.  

---

## General Testing & Notes
- Use the **provided API server URL**. Cypress will also run with **mocked** REST calls, so **stick to selectors/text** exactly as above.  
- Ensure **unique** selectors per page to avoid ambiguous matches.  
- Follow accessibility best practices (axe-core) on **every page**; all interactive elements must be tabbable and actionable via Enter.

---

## Acceptance Summary
1. Load **books** once and render cards with filters. (`GET /books`)  
2. Navigate to **book detail** and render shelf/copy availability. (`GET /books/{book-id}/availability`)  
3. Click copies to create/replace **reservation**; show **timer**; handle expiry; maintain **Selected Copies** panel. (`POST /books/{book-id}/reservation`)  
4. Submit **Borrow form**; on success, show **Loans page** with returned `loans[]`. (`POST /books/{book-id}/borrow`)  
5. **Retrieve loans** later via `{ code, name }`. (`POST /loans`)  
6. **Return** a loan. (`POST /loans/{loan-id}/return`)  
