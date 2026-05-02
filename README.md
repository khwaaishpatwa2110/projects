# NOVA — Entertainment Agency Management System

> A fully browser-based, real-time agency management platform built as a **Database Management Systems (DBMS) practical project**. No server. No installation. Open the HTML file and your SQLite database starts instantly.

---

## 🎬 Overview

**NOVA** is a single-file web application that simulates the operations of a large-scale entertainment agency — managing artists, trainees, concert tours, ticket sales, and merchandise inventory — all backed by a real **SQLite database running entirely in the browser** via [SQL.js](https://sql-wasm.netlify.app/).

Every action you take (adding an artist, issuing a concert, adjusting stock) directly executes SQL against the live database. Your data **persists across sessions** via `localStorage` — close the tab, reopen it, your records are still there.

---

## ✨ Features

| Module | What It Does |
|---|---|
| **Dashboard** | Live stats (artists, trainees, concerts, merch), revenue bar chart via `LEFT JOIN` aggregate, recent activity panels |
| **Artist Profiles** | Add / delete artists with genre, debut year, contract dates, status. `UNIQUE` constraint on stage name |
| **Trainees** | Enroll trainees, log daily attendance (`INSERT` into Attendance), graduate trainees via `UPDATE` |
| **Evaluations** | Submit vocal / dance / visual / overall scores. Table uses `INNER JOIN` between Evaluations and Trainees |
| **Concerts & Tours** | Schedule concerts linked to artists via `FOREIGN KEY`. Delete cascades to Tickets |
| **Ticket Sales** | Record live ticket sales. Revenue computed as `quantity × ticket_price` via `JOIN` |
| **Merchandise** | Add inventory items. Real-time stock adjustment using `UPDATE … MAX(0, stock + adj)` to prevent negative stock |
| **SQL Terminal** | Full raw SQL shell — run any `DDL`, `DML`, `DQL`, `JOIN`, or aggregate query with formatted output |
| **DB Schema** | ASCII entity-relationship overview of all 7 tables and their FK connections |

---

## 🗄️ Database Design

### Tables (7 total)

```
Artists          (artist_id PK, stage_name UNIQUE, real_name, genre, debut_year, contract_end, status)
    │
    ├──(1:N)──▶ Concerts     (concert_id PK, name, artist_id FK, venue, city, concert_date,
    │                          capacity, ticket_price, status)
    │                │
    │                └──(1:N)──▶ Tickets   (ticket_id PK, concert_id FK, buyer_name,
    │                                        quantity, seat_category, sale_date)
    │
    └──(1:N)──▶ Merchandise  (merch_id PK, item_name, artist_id FK, category, price, stock_quantity)

Trainees         (trainee_id PK, full_name, dob, specialty, enrolled_date, monthly_fee, status)
    │
    ├──(1:N)──▶ Evaluations  (eval_id PK, trainee_id FK, eval_date, vocal_score,
    │                          dance_score, visual_score, overall_score, notes)
    │
    └──(1:N)──▶ Attendance   (att_id PK, trainee_id FK, att_date, att_status)
```

### Constraints Used

| Type | Example |
|---|---|
| `PRIMARY KEY AUTOINCREMENT` | Every table |
| `FOREIGN KEY` | `Concerts.artist_id → Artists`, `Tickets.concert_id → Concerts` |
| `ON DELETE CASCADE` | Deleting a concert removes its tickets automatically |
| `UNIQUE` | `Artists.stage_name` |
| `NOT NULL` | `Concerts.venue`, `Trainees.full_name` |
| `CHECK` | `Evaluations.vocal_score BETWEEN 0 AND 100`, `Merchandise.stock_quantity >= 0` |
| `DEFAULT` | `Loans.status = 'Active'`, `date('now')` on sale dates |

### Normalisation

The schema satisfies **Third Normal Form (3NF)**:
- **1NF** — All attributes are atomic; no repeating groups
- **2NF** — All non-key attributes fully depend on the whole primary key (all PKs are single-column)
- **3NF** — No transitive dependencies; `ticket_price` lives in `Concerts`, not duplicated in `Tickets`

---

## 🧑‍💻 SQL Operations Demonstrated

### DDL — Data Definition Language
```sql
CREATE TABLE Artists (
  artist_id    INTEGER PRIMARY KEY AUTOINCREMENT,
  stage_name   TEXT    NOT NULL UNIQUE,
  genre        TEXT    NOT NULL,
  debut_year   INTEGER CHECK(debut_year >= 1990 AND debut_year <= 2030),
  status       TEXT    DEFAULT 'Active' CHECK(status IN ('Active','Hiatus','Retired'))
);
```

### DML — Data Manipulation Language
```sql
-- INSERT
INSERT INTO Artists (stage_name, real_name, genre, debut_year, status)
VALUES ('ECLIPSE', 'Kim Soo-Jin', 'K-Pop', 2019, 'Active');

-- UPDATE — real-time stock adjustment (prevents negative stock)
UPDATE Merchandise
SET stock_quantity = MAX(0, stock_quantity + (-50))
WHERE merch_id = 1;

-- DELETE — cascades to all related Tickets
DELETE FROM Concerts WHERE concert_id = 3;
```

### DQL — Data Query Language
```sql
-- SELECT with WHERE + ORDER BY
SELECT stage_name, genre, debut_year
FROM Artists
WHERE status = 'Active'
ORDER BY debut_year ASC;

-- GROUP BY + HAVING
SELECT concert_id, SUM(quantity) AS total_sold
FROM Tickets
GROUP BY concert_id
HAVING total_sold > 5;
```

### JOIN Queries
```sql
-- Artist revenue — 3-table LEFT JOIN + aggregate
SELECT a.stage_name,
       COUNT(DISTINCT c.concert_id)              AS Concerts,
       COALESCE(SUM(t.quantity), 0)               AS Tickets_Sold,
       COALESCE(SUM(t.quantity * c.ticket_price), 0) AS Revenue
FROM Artists a
LEFT JOIN Concerts c ON a.artist_id  = c.artist_id
LEFT JOIN Tickets  t ON c.concert_id = t.concert_id
GROUP BY a.artist_id
ORDER BY Revenue DESC;
```

### Subquery
```sql
-- Trainees scoring above the group average
SELECT full_name, specialty
FROM Trainees
WHERE trainee_id IN (
  SELECT trainee_id FROM Evaluations
  WHERE overall_score > (SELECT AVG(overall_score) FROM Evaluations)
);
```

---

## 💾 Data Persistence

NOVA saves your database to `localStorage` automatically after every change — add an artist, close the tab, reopen it, the artist is still there.

| Event | What Happens |
|---|---|
| First open | Fresh database with seed data, saved to `localStorage` |
| Every INSERT / UPDATE / DELETE | `db.export()` → Base64 → `localStorage` |
| Reopen the file | Reads from `localStorage`, restores full database |
| Status bar shows | **"nova_agency.db · Restored ✓"** when loaded from storage |
| **Reset DB** button | Clears `localStorage` and reloads with default seed data |

> **Note:** Data is tied to the browser + file path. Opening the file in a different browser or from a different folder will start fresh.

---

## 🚀 Getting Started

No installation required.

```bash
# 1. Clone or download the repository
git clone https://github.com/your-username/nova-entertainment-agency.git

# 2. Open the file in any modern browser
open entertainment_agency.html
# or just double-click the file
```

That's it. The SQLite engine loads via CDN ([sql-wasm](https://cdnjs.cloudflare.com/ajax/libs/sql.js/1.10.2/)) and the database initialises automatically.

### Browser Compatibility

| Browser | Status |
|---|---|
| Chrome / Edge | ✅ Fully supported |
| Firefox | ✅ Fully supported |
| Safari | ✅ Fully supported |
| Mobile browsers | ✅ Responsive layout |

---

## 🧪 Testing

The SQL Terminal (sidebar → **⌥ SQL Terminal**) lets you run arbitrary queries live against the database. Three built-in sample queries are included:

| Sample | Query Type |
|---|---|
| **JOIN Example** | `LEFT JOIN` across Artists → Concerts → Tickets with revenue aggregation |
| **Aggregate** | `AVG` scores per trainee with `HAVING` filter |
| **Subquery** | Nested `SELECT` for above-average scorers |

Use **Ctrl + Enter** to run a query quickly.

---

## 🗂️ Project Structure

```
nova-entertainment-agency/
└── entertainment_agency.html    # Entire application — HTML + CSS + JS + SQL schema
```

Everything is intentionally in a single self-contained file for maximum portability.

---

## 🛠️ Tech Stack

| Technology | Purpose |
|---|---|
| **HTML5 / CSS3 / Vanilla JS** | Frontend — no frameworks |
| **SQL.js v1.10.2** | SQLite compiled to WebAssembly, runs in browser |
| **localStorage** | Cross-session data persistence |
| **Bebas Neue** | Display typeface |
| **DM Sans** | Body typeface |
| **JetBrains Mono** | Code / data display |

---

## 📋 Seed Data

The application comes pre-loaded with sample data for demonstration:

- **4 Artists** — ECLIPSE, VORTEX, LUNAR, CIPHER across K-Pop, R&B, Hip-Hop, Pop
- **4 Trainees** — with specialties in Dance, Vocal, All-Rounder, Rap
- **4 Evaluations** — scored across vocal, dance, visual, and overall
- **5 Concerts** — spanning Seoul, Tokyo, New York, Paris
- **7 Ticket Sales** — across completed and upcoming concerts
- **6 Merchandise Items** — photobooks, albums, lightsticks, apparel

---

## 📚 Academic Context

This project was built as part of a **2nd Semester DBMS Practical** submission, demonstrating:

- ✅ Real-time application directly connected to a database
- ✅ Proper database schema with tables, relationships, and constraints
- ✅ SQL operations: DDL, DML, DQL, JOINs, Aggregates, Subqueries
- ✅ Data integrity, consistency, and 3NF normalisation
- ✅ Features: insertion, updating, deletion, retrieval
- ✅ Usability and real-world applicability

---

## 📄 License

This project is open source and available under the [MIT License](LICENSE).

---

<div align="center">

**NOVA Entertainment Agency Management System**  
Built with SQL.js · Runs entirely in the browser · No server required

</div>
