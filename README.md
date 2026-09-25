# Database Internals & Query Optimization

A video + slides series that opens the black box of relational databases: how data is stored, how queries are costed, and how engines keep your data fast, consistent, and recoverable. Examples run on **PostgreSQL**.

Made by **AbdulRahman Tamer**.

---

## Series Roadmap

| # | Topic | Status |
|---|-------|--------|
| 1 | Query Cost: pages, disk I/O, and the heap | Released |
| 2 | Indexes: B+ trees | Planned |
| 3 | Index scans, index-only scans, and bitmap scans | Planned |
| 4 | Query optimization: how the planner picks a plan | Planned |
| 5 | Transactions and ACID | Planned |
| 6 | Concurrency control | Planned |
| 7 | Write-Ahead Logging (WAL) | Planned |
| 8 | Recovery techniques | Planned |

The list will grow as the series does.

---

## Episode 1: Understanding Query Cost

**Video:** Coming soon <!-- replace with: [Watch on YouTube](https://www.youtube.com/watch?v=XXXXXXXXXXX) -->
**Slides:** [Query_Optimizaiton_Presentation_1.pdf](https://github.com/AbdulRahman-cy/Database_Internals/blob/main/Query_Optimizaiton_Presentation_1.pdf)
**Usage file (commands):** [EXPLAIN_Cost_Calculation_Experiment.pdf](https://github.com/AbdulRahman-cy/Database_Internals/blob/main/EXPLAIN_Cost_Calculation_Experiment.pdf)

To optimize a query you first have to know what you are optimizing. This episode builds the cost model from scratch, first on paper and then against a real PostgreSQL instance.

### What it covers

- What "cost" means: CPU time, memory, and disk I/O
- How rows are grouped into fixed-size **pages** (8 KB in PostgreSQL, 16 KB in MySQL)
- Why the database reads whole pages, not single rows, and what that does to I/O
- The **heap** and why traversing it is expensive
- Cached page reads: cheaper than disk, but not free (deserialization still costs CPU)
- Theoretical cost: blocking factor, block count, average-case full scan
- Practical cost: reading `EXPLAIN` output and reconciling it with the theory
- Why `SELECT *` hurts (row width goes over the wire)
- A first look at an index scan vs a sequential scan

### Key formulas

```
Blocking factor     Bfr = floor(B / R)              -- B = block size, R = record size
Number of blocks    b   = r / Bfr                   -- r = number of records
Full scan (avg)     b / 2 disk I/Os to find one record
Full scan (worst)   b disk I/Os

Practical cost for a seq scan WITHOUT an index
  Cost = (cpu_tuple_cost * rows) + (pages)
```

### Worked example (theory)

`EMPLOYEE(NAME, SSN, ADDRESS, JOB, SAL, ...)` with R = 150 B, B = 512 B, r = 30,000:

| Step | Calculation | Result |
|------|-------------|--------|
| Blocking factor | 512 div 150 | 3 records/block |
| Data blocks | 30,000 / 3 | 10,000 |
| Linear scan, average | 10,000 / 2 | 5,000 I/Os |
| Index entry size | 9 B (SSN) + 7 B (pointer) | 16 B |
| Index entries per block | 512 div 16 | 32 |
| Index blocks | 30,000 / 32 | ~938 |
| Binary search on index | log2(938) | ~10 I/Os |

The index cuts the average cost from about 5,000 to about 10 block accesses, roughly **500x**.

### Hands-on (practice)

**1. Set up the Docker container**

```bash
docker run --name my-postgres -e POSTGRES_PASSWORD=mysecretpassword -d -p 5432:5432 postgres
```

**2. Enter the psql shell**

```bash
docker exec -it my-postgres psql -U postgres
```

**3. Create the students table, load 1 million rows, and add indexes**

```sql
-- 1. Create the table
CREATE TABLE students (
    id INT,
    name VARCHAR(50),
    grade INT
);

-- 2. Insert 1 million rows
INSERT INTO students (id, name, grade)
SELECT
    x,
    'Student_' || x,
    floor(random() * 101)
FROM generate_series(1, 1000000) AS x;

-- 3. Create indexes on both 'id' and 'grade'
CREATE INDEX idx_students_id ON students(id);
CREATE INDEX idx_students_grade ON students(grade);
```

**4. Get the number of pages (the theoretical cost)**

```sql
SELECT relpages
FROM pg_class
WHERE relname = 'students';
-- ~6370 in the video
-- if you get 0, run: ANALYZE students;  and query again
```

**5. Run the commands and read the plans**

```sql
-- 1
explain select * from students;
-- 2
explain select * from students where id = 10;
```

Query 1 is a sequential scan, roughly:

```
Seq Scan on students  (cost=0.00..16370.00 rows=1000000 width=22)
```

Why 16370 and not 6370:

```sql
SHOW cpu_tuple_cost;   -- 0.01
-- (0.01 * 1,000,000 rows) + 6370 pages = 10000 + 6370 = 16370
```

Reading the output: `0.00` is the startup cost, `16370.00` is the total cost, `rows` is the estimated row count, and `width` is the average row size in bytes.

Query 2 becomes an **Index Scan** on `idx_students_id`. The startup cost is no longer zero because the engine has to descend the B+ tree first, but the total cost drops dramatically.

> Exact numbers depend on your PostgreSQL version, row layout, and settings. Your page count may differ slightly from the video.

### Takeaways

1. Cost is dominated by disk I/O, so minimize the pages you touch.
2. The theoretical model (pages) explains most of the practical cost; CPU per-row cost explains the rest.
3. A full heap scan reads every page even when you want one row.
4. Select only the columns you need.
5. Indexes tell the engine which page to fetch. That is the next episode.

---

## Repository Contents

```
.
├── README.md
├── Query_Optimizaiton_Presentation_1.pdf       # Episode 1 slides
└── EXPLAIN_Cost_Calculation_Experiment.pdf     # Episode 1 commands
```

---

## Prerequisites

- Basic SQL
- Docker (to run the examples) or a local PostgreSQL install
- No prior database-internals knowledge needed

---

## Author

**AbdulRahman Tamer**
Backend engineer and engineering student at Alexandria University.

- YouTube: [@AbdulRahmanBackend](https://www.youtube.com/@AbdulRahmanBackend)
- LinkedIn: [abdulrahman-tamer](https://www.linkedin.com/in/abdulrahman-tamer-65151b379)
- GitHub: [AbdulRahman-cy](https://github.com/AbdulRahman-cy)

---

## License

<!-- Choose a license, e.g. MIT for code and CC BY 4.0 for slides -->
