# Database Schema Design

**Track:** Design Concepts
**Difficulty Tier:** Beginner
**Prerequisites:** None

## Concept Overview

Database schema design is the process of defining how data is organized, stored, and related within a database. A well-designed schema captures the essential entities of a domain, the attributes that describe them, and the relationships that connect them. It serves as the blueprint that every query, report, and application feature ultimately depends on.

The core building blocks are tables (relations), columns (attributes), primary keys, foreign keys, and constraints. Normalization — the practice of structuring tables to eliminate redundancy and prevent update anomalies — is a central discipline. The most commonly applied normal forms are First Normal Form (1NF), Second Normal Form (2NF), and Third Normal Form (3NF). Knowing when to normalize and when to intentionally denormalize for read performance is a judgment call that separates competent designers from beginners.

Beyond normalization, schema design involves choosing appropriate data types, defining indexes for query performance, and modeling relationships (one-to-one, one-to-many, many-to-many) correctly. Poor schema decisions are expensive to fix after a system is in production because they ripple through application code, migration scripts, and downstream analytics. The problems in this module give you practice translating real-world requirements into clean, maintainable schemas.

---

## Problem 1 — Modeling a Library Catalog

**Difficulty:** Easy
**Estimated Time:** 30 minutes

### Problem Statement

Design a database schema for a public library's catalog system. The library needs to track books, authors, and which authors wrote which books. A book can have multiple authors, and an author can have written multiple books. Each book has a title, ISBN, publication year, and genre. Each author has a name and birth year.

### Scenario

**Context:** A city library is migrating from a paper card catalog to a digital system. Librarians need to search for books by title, author, or genre, and view all books by a given author.

**Requirements:** Capture the many-to-many relationship between books and authors. Ensure every book has a unique ISBN. Support efficient lookups by author name and book title.

**Expected Approach:** Identify the core entities, define their attributes and primary keys, and introduce a junction table to model the many-to-many relationship.

<details>
<summary>Hints</summary>

1. You need three tables: one for books, one for authors, and a junction table that links them.
2. The junction table holds pairs of foreign keys — one pointing to the book and one pointing to the author. Its primary key is the combination of both foreign keys.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Model the many-to-many relationship with a junction table.

1. Create a `books` table with columns: `book_id` (PK), `title`, `isbn` (unique), `publication_year`, `genre`.
2. Create an `authors` table with columns: `author_id` (PK), `name`, `birth_year`.
3. Create a `book_authors` junction table with columns: `book_id` (FK → books), `author_id` (FK → authors). The composite primary key is `(book_id, author_id)`.
4. Add an index on `authors.name` for author-name lookups.
5. Add an index on `books.title` for title searches.
6. The `isbn` unique constraint prevents duplicate book entries.

This cleanly separates entity data from relationship data and avoids storing author names redundantly inside the books table.

</details>

---

## Problem 2 — Designing an E-Commerce Product Schema

**Difficulty:** Easy
**Estimated Time:** 30 minutes

### Problem Statement

Design a schema for an e-commerce platform that sells products organized into categories. Each product belongs to exactly one category. Products have a name, description, price, and stock quantity. Categories have a name and an optional parent category (to support nested categories like Electronics → Laptops → Gaming Laptops).

### Scenario

**Context:** A startup is building an online store. The product team wants a category tree that can be browsed hierarchically — clicking "Electronics" shows subcategories, and clicking "Laptops" shows products. They also need to display product counts per category.

**Requirements:** Model the self-referencing category hierarchy. Ensure product prices are stored with appropriate precision. Prevent orphaned products (every product must belong to a valid category).

**Expected Approach:** Recognize the self-referencing foreign key pattern for hierarchical categories and the one-to-many relationship between categories and products.

<details>
<summary>Hints</summary>

1. A category's optional parent is modeled as a nullable foreign key that references the same `categories` table.
2. Use a `DECIMAL` or `NUMERIC` type for price to avoid floating-point rounding issues with currency.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Use a self-referencing foreign key for the category tree and a standard one-to-many link for products.

1. Create a `categories` table: `category_id` (PK), `name`, `parent_category_id` (FK → categories, nullable). A null parent means the category is a root node.
2. Create a `products` table: `product_id` (PK), `name`, `description`, `price` (DECIMAL(10,2)), `stock_quantity` (INTEGER, default 0), `category_id` (FK → categories, NOT NULL).
3. The NOT NULL constraint on `category_id` prevents orphaned products.
4. Add an index on `products.category_id` to speed up queries that list products within a category.
5. To display the full category path (e.g., "Electronics → Laptops → Gaming Laptops"), the application performs a recursive query walking `parent_category_id` up to a null root.

The self-referencing pattern supports arbitrary nesting depth without schema changes.

</details>

---

## Problem 3 — Normalizing a Student Enrollment System

**Difficulty:** Easy
**Estimated Time:** 35 minutes

### Problem Statement

A university stores enrollment data in a single flat table with columns: `student_name`, `student_email`, `course_name`, `instructor_name`, `instructor_email`, `semester`, and `grade`. This table has significant redundancy — the same instructor email appears in every row for every student in that course. Redesign the schema to Third Normal Form (3NF).

### Scenario

**Context:** The registrar's office has been using a spreadsheet-style table for years. They recently discovered that updating an instructor's email requires changing hundreds of rows, and a typo in one row caused a student's transcript to show the wrong instructor. They want a properly normalized schema.

**Requirements:** Eliminate all redundant data storage. Ensure that updating an instructor's email requires changing exactly one row. Preserve the ability to query a student's transcript (all courses, grades, and semesters) and a course's roster (all enrolled students and their grades).

**Expected Approach:** Identify the functional dependencies in the flat table, decompose it into separate entity tables, and link them with foreign keys.

<details>
<summary>Hints</summary>

1. Identify three entities hiding in the flat table: students, courses, and enrollments. Instructor data is an attribute of a course, not of an enrollment.
2. A student is uniquely identified by their email. A course is uniquely identified by its name (or better, a course code). An enrollment is the combination of a student, a course, and a semester.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Decompose the flat table by identifying functional dependencies and extracting entities.

1. Create a `students` table: `student_id` (PK), `name`, `email` (unique).
2. Create an `instructors` table: `instructor_id` (PK), `name`, `email` (unique). Extracting instructors into their own table means updating an email touches exactly one row.
3. Create a `courses` table: `course_id` (PK), `course_name`, `instructor_id` (FK → instructors). Each course has one instructor.
4. Create an `enrollments` table: `enrollment_id` (PK), `student_id` (FK → students), `course_id` (FK → courses), `semester`, `grade`. A composite unique constraint on `(student_id, course_id, semester)` prevents duplicate enrollments.
5. Verify 3NF:
   - 1NF: All columns are atomic, no repeating groups.
   - 2NF: No partial dependencies — every non-key attribute depends on the full primary key of its table.
   - 3NF: No transitive dependencies — instructor data lives in its own table, not repeated inside courses or enrollments.

Transcript query: join `enrollments` → `courses` → `instructors` filtered by `student_id`. Roster query: join `enrollments` → `students` filtered by `course_id` and `semester`.

</details>

---

## Problem 4 — Designing a Multi-Tenant SaaS Billing Schema

**Difficulty:** Medium
**Estimated Time:** 45 minutes

### Problem Statement

Design a database schema for a SaaS application that supports multiple tenant organizations. Each tenant has users, a subscription plan, and a billing history. Users belong to exactly one tenant. Subscription plans define a monthly price and feature limits (e.g., max users, max storage). The billing history records each monthly invoice with its amount, status (paid, pending, overdue), and payment date.

### Scenario

**Context:** A B2B project management tool serves hundreds of companies. Each company (tenant) signs up for a plan (Free, Pro, Enterprise), adds team members, and receives a monthly invoice. The finance team needs to query total revenue by plan tier, identify overdue accounts, and track plan changes over time.

**Requirements:** Isolate tenant data logically so queries for one tenant never accidentally return another tenant's data. Track plan changes — if a tenant upgrades from Pro to Enterprise mid-month, the system must record both the old and new plan associations with timestamps. Support querying revenue aggregated by plan tier and by month.

**Expected Approach:** Model tenants, users, plans, and invoices as separate entities. Introduce a plan-history or subscription table that tracks which plan a tenant is on over time, rather than storing a single current-plan foreign key on the tenant.

<details>
<summary>Hints</summary>

1. A `subscriptions` table with `start_date` and `end_date` columns lets you track plan changes over time. The current subscription is the one where `end_date` is null.
2. Every table that holds tenant-specific data should include a `tenant_id` foreign key. This makes tenant-scoped queries straightforward and supports row-level security policies.
3. Invoices reference the subscription that was active when they were generated, not the plan directly — this preserves historical accuracy even if the tenant later changes plans.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Separate concerns into tenants, users, plans, subscriptions (plan history), and invoices.

1. Create a `plans` table: `plan_id` (PK), `name` (e.g., Free, Pro, Enterprise), `monthly_price` (DECIMAL(10,2)), `max_users` (INTEGER), `max_storage_gb` (INTEGER).
2. Create a `tenants` table: `tenant_id` (PK), `company_name`, `created_at`.
3. Create a `users` table: `user_id` (PK), `tenant_id` (FK → tenants, NOT NULL), `name`, `email` (unique), `role`.
4. Create a `subscriptions` table: `subscription_id` (PK), `tenant_id` (FK → tenants), `plan_id` (FK → plans), `start_date`, `end_date` (nullable). A null `end_date` indicates the currently active subscription. Add a constraint or application logic ensuring a tenant has at most one active subscription at a time.
5. Create an `invoices` table: `invoice_id` (PK), `subscription_id` (FK → subscriptions), `tenant_id` (FK → tenants), `amount` (DECIMAL(10,2)), `status` (ENUM: paid, pending, overdue), `issued_date`, `payment_date` (nullable).
6. Revenue by plan tier: join `invoices` → `subscriptions` → `plans`, group by `plans.name` and month of `issued_date`, sum `amount` where `status = 'paid'`.
7. Overdue accounts: query `invoices` where `status = 'overdue'`, join to `tenants` for company details.
8. Index `invoices.tenant_id` and `invoices.status` for common query patterns.

The subscription history pattern avoids losing information when a tenant changes plans and keeps invoices tied to the exact plan that was active when they were generated.

</details>

---

## Problem 5 — Designing a Schema for a Social Media Platform

**Difficulty:** Hard
**Estimated Time:** 60 minutes

### Problem Statement

Design a database schema for a social media platform that supports user profiles, posts, comments, likes (on both posts and comments), follower relationships, and hashtag-based content discovery. The schema must handle the fact that likes can target two different entity types, followers form a directed graph (A follows B does not imply B follows A), and hashtags create a many-to-many relationship with posts.

### Scenario

**Context:** A startup is building a Twitter-like platform. Users create profiles, publish short text posts, comment on posts, like posts or comments, follow other users, and search for content by hashtag. The product team wants a feed feature that shows recent posts from followed users, a trending-hashtags feature, and per-post engagement metrics (like count, comment count).

**Requirements:**
- Users have a username (unique), display name, bio, and join date.
- Posts have text content, a timestamp, and belong to one author.
- Comments belong to one post and one author, with text and a timestamp.
- Likes can target either a post or a comment. A user can like a given post or comment at most once.
- The follower relationship is directed: a `follows` table records who follows whom.
- Posts can be tagged with multiple hashtags, and a hashtag can appear on multiple posts.
- The schema must support efficient queries for: a user's feed (posts from followed users, reverse-chronological), trending hashtags (most used in the last 24 hours), and engagement metrics per post.

**Expected Approach:** Identify all entities and relationships. Decide how to model polymorphic likes (a like that can point to either a post or a comment). Consider the trade-offs between a single polymorphic likes table versus separate `post_likes` and `comment_likes` tables. Design indexes that support the key query patterns.

<details>
<summary>Hints</summary>

1. For polymorphic likes, you have two main options: (a) a single `likes` table with `target_type` (post or comment) and `target_id` columns, or (b) two separate tables `post_likes` and `comment_likes`. Option (b) allows proper foreign key constraints; option (a) is simpler but sacrifices referential integrity at the database level.
2. The `follows` table needs two user foreign keys: `follower_id` and `followed_id`. The composite primary key `(follower_id, followed_id)` prevents duplicate follow relationships.
3. For the feed query, you need an index on `posts.author_id` and `posts.created_at` so you can efficiently find recent posts by a set of followed user IDs.
4. For trending hashtags, an index on the junction table's timestamp (or the post's timestamp) lets you count hashtag usage within a time window.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Model each entity and relationship explicitly, using separate like tables for referential integrity.

1. **Users table:** `user_id` (PK), `username` (unique), `display_name`, `bio`, `joined_at`.

2. **Posts table:** `post_id` (PK), `author_id` (FK → users), `content` (VARCHAR with length limit), `created_at`. Index on `(author_id, created_at DESC)` for feed queries.

3. **Comments table:** `comment_id` (PK), `post_id` (FK → posts), `author_id` (FK → users), `content`, `created_at`. Index on `post_id` for loading comments on a post.

4. **Post likes table:** `post_id` (FK → posts), `user_id` (FK → users), `created_at`. Composite PK `(post_id, user_id)` enforces one-like-per-user-per-post.

5. **Comment likes table:** `comment_id` (FK → comments), `user_id` (FK → users), `created_at`. Composite PK `(comment_id, user_id)` enforces one-like-per-user-per-comment.

   Using two separate tables instead of a polymorphic table preserves foreign key constraints — the database itself guarantees that a liked post or comment actually exists.

6. **Follows table:** `follower_id` (FK → users), `followed_id` (FK → users), `created_at`. Composite PK `(follower_id, followed_id)`. Index on `followed_id` to efficiently find a user's followers.

7. **Hashtags table:** `hashtag_id` (PK), `tag` (unique, lowercase-normalized).

8. **Post hashtags junction table:** `post_id` (FK → posts), `hashtag_id` (FK → hashtags). Composite PK `(post_id, hashtag_id)`. Include `created_at` (denormalized from the post or stored directly) to support time-windowed trending queries.

9. **Key query support:**
   - *Feed:* Select from `posts` where `author_id IN (select followed_id from follows where follower_id = ?)` order by `created_at DESC`, using the composite index on `(author_id, created_at)`.
   - *Trending hashtags:* Join `post_hashtags` → `posts`, filter `posts.created_at > NOW() - 24 hours`, group by `hashtag_id`, order by count descending.
   - *Engagement metrics:* Count rows in `post_likes` and `comments` for a given `post_id`. These can be materialized as counter columns on the `posts` table if read performance is critical, at the cost of write complexity.

This schema balances normalization with practical query performance and maintains full referential integrity.

</details>
