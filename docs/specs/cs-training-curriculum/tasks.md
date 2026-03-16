# Implementation Plan: CS Training Curriculum

## Overview

Create a complete 52-module computer science training curriculum as a collection of markdown documents, organized into two tracks ("Algorithms & Data Structures" and "Design Concepts"), with Mermaid prerequisite graphs rendered natively by GitHub/GitLab/VS Code. Each concept module is identified by its concept name (not a week number) and contains 3–10 daily problems depending on concept depth.

## Tasks

- [x] 1. Set up directory structure and foundational files
  - [x] 1.1 Create the curriculum directory tree
    - Create `curriculum/` root directory
    - Create `curriculum/algorithms-and-data-structures/` track directory
    - Create `curriculum/design-concepts/` track directory
    - _Requirements: 8.1_

  - [x] 1.2 Create the top-level `curriculum/README.md`
    - Write curriculum title, introduction, and overview of the two-track structure
    - Add a table listing all ~52 concept modules with columns: Track, Concept, Difficulty Tier, Link
    - Modules are identified by concept name, not week number
    - Add a simplified Mermaid overview diagram showing high-level progression across both tracks
    - Include links to each track's README
    - _Requirements: 8.2, 9.6_

- [x] 2. Create Algorithms & Data Structures track index
  - [x] 2.1 Create `curriculum/algorithms-and-data-structures/README.md`
    - Write track title and description
    - Add a table listing all ~26 concept modules with columns: Concept, Difficulty Tier, Prerequisites, Link
    - Add the full Mermaid prerequisite graph showing directed edges from prerequisite to dependent concept (nodes labeled by concept name, not week number)
    - Prerequisite graph must be consistent with the prerequisite lists in each module document
    - _Requirements: 4.1, 4.2, 4.4, 8.4, 9.1, 9.3, 9.4, 9.5_

- [x] 3. Create Design Concepts track index
  - [x] 3.1 Create `curriculum/design-concepts/README.md`
    - Write track title and description
    - Add a table listing all ~26 concept modules with columns: Concept, Difficulty Tier, Prerequisites, Link
    - Add the full Mermaid prerequisite graph showing directed edges from prerequisite to dependent concept (nodes labeled by concept name, not week number)
    - Prerequisite graph must be consistent with the prerequisite lists in each module document
    - _Requirements: 5.1, 5.2, 5.4, 8.4, 9.2, 9.3, 9.4, 9.5_

- [x] 4. Checkpoint — Verify track structure
  - Ensure the directory structure, top-level README, and both track READMEs are complete and internally consistent. Ask the user if questions arise.

- [x] 5. Create Algorithms & Data Structures Beginner-tier concept modules
  - [x] 5.1 Create `curriculum/algorithms-and-data-structures/arrays/README.md`
    - Concept: Arrays — Difficulty Tier: Beginner — Prerequisites: None
    - Include concept overview, prerequisite list, and 3–10 daily problems (at least 1 Easy, at least 1 Hard, ordered by difficulty)
    - Beginner modules skew toward Easy problems
    - Each problem: title, problem statement, difficulty, estimated time (30–60 min), example input/output, at least one hint, solution outline with pseudocode
    - Hints and solution outlines must be wrapped in collapsible `<details>` elements (hidden by default)
    - Problems must exercise different aspects of the concept (no duplicate sub-skills)
    - _Requirements: 1.4, 2.1, 2.2, 2.3, 2.4, 2.5, 3.1, 3.2, 3.3, 3.4, 3.5, 3.6, 3.7, 3.8, 6.1, 6.3, 7.1, 7.2, 7.3, 8.3, 8.5_

  - [x] 5.2 Create `curriculum/algorithms-and-data-structures/linked-lists/README.md`
    - Concept: Linked Lists — Difficulty Tier: Beginner — Prerequisites: None
    - Same structure and quality requirements as 5.1
    - _Requirements: 1.4, 2.1, 2.2, 2.3, 2.4, 2.5, 3.1–3.8, 6.1, 6.3, 7.1–7.3, 8.3, 8.5_

  - [x] 5.3 Create `curriculum/algorithms-and-data-structures/stacks/README.md`
    - Concept: Stacks — Difficulty Tier: Beginner — Prerequisites: None
    - Same structure and quality requirements as 5.1
    - _Requirements: 1.4, 2.1–2.5, 3.1–3.8, 6.1, 6.3, 7.1–7.3, 8.3, 8.5_

  - [x] 5.4 Create `curriculum/algorithms-and-data-structures/queues/README.md`
    - Concept: Queues — Difficulty Tier: Beginner — Prerequisites: None
    - Same structure and quality requirements as 5.1
    - _Requirements: 1.4, 2.1–2.5, 3.1–3.8, 6.1, 6.3, 7.1–7.3, 8.3, 8.5_

  - [x] 5.5 Create `curriculum/algorithms-and-data-structures/hash-maps/README.md`
    - Concept: Hash Maps — Difficulty Tier: Beginner — Prerequisites: None
    - Same structure and quality requirements as 5.1
    - _Requirements: 1.4, 2.1–2.5, 3.1–3.8, 6.1, 6.3, 7.1–7.3, 8.3, 8.5_

  - [x] 5.6 Create `curriculum/algorithms-and-data-structures/string-manipulation/README.md`
    - Concept: String Manipulation — Difficulty Tier: Beginner — Prerequisites: Arrays
    - Same structure and quality requirements as 5.1
    - _Requirements: 1.4, 2.1–2.5, 3.1–3.8, 4.4, 6.1, 6.3, 7.1–7.3, 8.3, 8.5_

  - [x] 5.7 Create `curriculum/algorithms-and-data-structures/sorting-algorithms/README.md`
    - Concept: Sorting Algorithms — Difficulty Tier: Beginner — Prerequisites: Arrays
    - Same structure and quality requirements as 5.1
    - _Requirements: 1.4, 2.1–2.5, 3.1–3.8, 4.4, 6.1, 6.3, 7.1–7.3, 8.3, 8.5_

  - [x] 5.8 Create `curriculum/algorithms-and-data-structures/recursion/README.md`
    - Concept: Recursion — Difficulty Tier: Beginner — Prerequisites: None
    - Same structure and quality requirements as 5.1
    - _Requirements: 1.4, 2.1–2.5, 3.1–3.8, 6.1, 6.3, 7.1–7.3, 8.3, 8.5_

  - [x] 5.9 Create `curriculum/algorithms-and-data-structures/binary-search/README.md`
    - Concept: Binary Search — Difficulty Tier: Beginner — Prerequisites: Arrays, Sorting Algorithms
    - Same structure and quality requirements as 5.1
    - _Requirements: 1.4, 2.1–2.5, 3.1–3.8, 4.4, 6.1, 6.3, 7.1–7.3, 8.3, 8.5_

- [x] 6. Checkpoint — Verify Beginner Algorithms modules
  - Ensure all 9 Beginner-tier Algorithms & Data Structures modules are complete, consistently formatted, and prerequisite lists match the track README graph. Ask the user if questions arise.

- [x] 7. Create Algorithms & Data Structures Intermediate-tier concept modules
  - [x] 7.1 Create `curriculum/algorithms-and-data-structures/two-pointers/README.md`
    - Concept: Two Pointers — Difficulty Tier: Intermediate — Prerequisites: Arrays
    - Include concept overview, prerequisite list, and 3–10 daily problems with mixed difficulty (Intermediate tier)
    - Each problem: title, problem statement, difficulty, estimated time (30–60 min), example input/output, at least one hint, solution outline
    - At least 1 Easy and at least 1 Hard problem; problems ordered by difficulty; no duplicate sub-skills
    - _Requirements: 1.4, 2.1–2.5, 3.1–3.8, 4.4, 6.1, 6.2, 7.1–7.3, 8.3, 8.5_

  - [x] 7.2 Create `curriculum/algorithms-and-data-structures/sliding-window/README.md`
    - Concept: Sliding Window — Difficulty Tier: Intermediate — Prerequisites: Arrays
    - Same structure and quality requirements as 7.1
    - _Requirements: 1.4, 2.1–2.5, 3.1–3.8, 4.4, 6.1, 6.2, 7.1–7.3, 8.3, 8.5_

  - [x] 7.3 Create `curriculum/algorithms-and-data-structures/trees/README.md`
    - Concept: Trees — Difficulty Tier: Intermediate — Prerequisites: Linked Lists, Recursion
    - Same structure and quality requirements as 7.1
    - _Requirements: 1.4, 2.1–2.5, 3.1–3.8, 4.4, 6.1, 6.2, 7.1–7.3, 8.3, 8.5_

  - [x] 7.4 Create `curriculum/algorithms-and-data-structures/binary-search-trees/README.md`
    - Concept: Binary Search Trees — Difficulty Tier: Intermediate — Prerequisites: Binary Search, Trees
    - Same structure and quality requirements as 7.1
    - _Requirements: 1.4, 2.1–2.5, 3.1–3.8, 4.4, 6.1, 6.2, 7.1–7.3, 8.3, 8.5_

  - [x] 7.5 Create `curriculum/algorithms-and-data-structures/heaps/README.md`
    - Concept: Heaps — Difficulty Tier: Intermediate — Prerequisites: Trees, Binary Search Trees
    - Same structure and quality requirements as 7.1
    - _Requirements: 1.4, 2.1–2.5, 3.1–3.8, 4.4, 6.1, 6.2, 7.1–7.3, 8.3, 8.5_

  - [x] 7.6 Create `curriculum/algorithms-and-data-structures/tries/README.md`
    - Concept: Tries — Difficulty Tier: Intermediate — Prerequisites: Hash Maps, Trees
    - Same structure and quality requirements as 7.1
    - _Requirements: 1.4, 2.1–2.5, 3.1–3.8, 4.4, 6.1, 6.2, 7.1–7.3, 8.3, 8.5_

  - [x] 7.7 Create `curriculum/algorithms-and-data-structures/matrix-traversal/README.md`
    - Concept: Matrix Traversal — Difficulty Tier: Intermediate — Prerequisites: Arrays
    - Same structure and quality requirements as 7.1
    - _Requirements: 1.4, 2.1–2.5, 3.1–3.8, 4.4, 6.1, 6.2, 7.1–7.3, 8.3, 8.5_

  - [x] 7.8 Create `curriculum/algorithms-and-data-structures/graphs/README.md`
    - Concept: Graphs — Difficulty Tier: Intermediate — Prerequisites: Queues, Trees
    - Same structure and quality requirements as 7.1
    - _Requirements: 1.4, 2.1–2.5, 3.1–3.8, 4.4, 6.1, 6.2, 7.1–7.3, 8.3, 8.5_

  - [x] 7.9 Create `curriculum/algorithms-and-data-structures/bit-manipulation/README.md`
    - Concept: Bit Manipulation — Difficulty Tier: Intermediate — Prerequisites: None
    - Same structure and quality requirements as 7.1
    - _Requirements: 1.4, 2.1–2.5, 3.1–3.8, 6.1, 6.2, 7.1–7.3, 8.3, 8.5_

- [x] 8. Checkpoint — Verify Intermediate Algorithms modules
  - Ensure all 9 Intermediate-tier Algorithms & Data Structures modules are complete and consistent. Ask the user if questions arise.

- [x] 9. Create Algorithms & Data Structures Advanced-tier concept modules
  - [x] 9.1 Create `curriculum/algorithms-and-data-structures/backtracking/README.md`
    - Concept: Backtracking — Difficulty Tier: Advanced — Prerequisites: Stacks, Recursion
    - Include concept overview, prerequisite list, and 3–10 daily problems skewing toward Hard (Advanced tier)
    - Each problem: title, problem statement, difficulty, estimated time (30–60 min), example input/output, at least one hint, solution outline
    - At least 1 Easy and at least 1 Hard problem; problems ordered by difficulty; no duplicate sub-skills
    - _Requirements: 1.4, 2.1–2.5, 3.1–3.8, 4.4, 6.1, 6.4, 7.1–7.3, 8.3, 8.5_

  - [x] 9.2 Create `curriculum/algorithms-and-data-structures/divide-and-conquer/README.md`
    - Concept: Divide and Conquer — Difficulty Tier: Advanced — Prerequisites: Recursion
    - Same structure and quality requirements as 9.1
    - _Requirements: 1.4, 2.1–2.5, 3.1–3.8, 4.4, 6.1, 6.4, 7.1–7.3, 8.3, 8.5_

  - [x] 9.3 Create `curriculum/algorithms-and-data-structures/greedy-algorithms/README.md`
    - Concept: Greedy Algorithms — Difficulty Tier: Advanced — Prerequisites: Sorting Algorithms
    - Same structure and quality requirements as 9.1
    - _Requirements: 1.4, 2.1–2.5, 3.1–3.8, 4.4, 6.1, 6.4, 7.1–7.3, 8.3, 8.5_

  - [x] 9.4 Create `curriculum/algorithms-and-data-structures/dynamic-programming/README.md`
    - Concept: Dynamic Programming — Difficulty Tier: Advanced — Prerequisites: Recursion, Backtracking, Divide and Conquer
    - Same structure and quality requirements as 9.1
    - _Requirements: 1.4, 2.1–2.5, 3.1–3.8, 4.4, 6.1, 6.4, 7.1–7.3, 8.3, 8.5_

  - [x] 9.5 Create `curriculum/algorithms-and-data-structures/interval-problems/README.md`
    - Concept: Interval Problems — Difficulty Tier: Advanced — Prerequisites: Sorting Algorithms, Greedy Algorithms
    - Same structure and quality requirements as 9.1
    - _Requirements: 1.4, 2.1–2.5, 3.1–3.8, 4.4, 6.1, 6.4, 7.1–7.3, 8.3, 8.5_

  - [x] 9.6 Create `curriculum/algorithms-and-data-structures/union-find/README.md`
    - Concept: Union Find — Difficulty Tier: Advanced — Prerequisites: Graphs
    - Same structure and quality requirements as 9.1
    - _Requirements: 1.4, 2.1–2.5, 3.1–3.8, 4.4, 6.1, 6.4, 7.1–7.3, 8.3, 8.5_

  - [x] 9.7 Create `curriculum/algorithms-and-data-structures/topological-sort/README.md`
    - Concept: Topological Sort — Difficulty Tier: Advanced — Prerequisites: Graphs
    - Same structure and quality requirements as 9.1
    - _Requirements: 1.4, 2.1–2.5, 3.1–3.8, 4.4, 6.1, 6.4, 7.1–7.3, 8.3, 8.5_

  - [x] 9.8 Create `curriculum/algorithms-and-data-structures/advanced-graph-algorithms/README.md`
    - Concept: Advanced Graph Algorithms — Difficulty Tier: Advanced — Prerequisites: Graphs, Union Find, Topological Sort
    - Same structure and quality requirements as 9.1
    - _Requirements: 1.4, 2.1–2.5, 3.1–3.8, 4.4, 6.1, 6.4, 7.1–7.3, 8.3, 8.5_

- [x] 10. Checkpoint — Verify all Algorithms & Data Structures modules
  - Ensure all 26 Algorithms & Data Structures concept modules exist, are consistently formatted, prerequisite lists are correct, and the track README table/graph matches. Ask the user if questions arise.

- [x] 11. Create Design Concepts Beginner-tier concept modules
  - [x] 11.1 Create `curriculum/design-concepts/object-oriented-design-principles/README.md`
    - Concept: Object-Oriented Design Principles — Difficulty Tier: Beginner — Prerequisites: None
    - Include concept overview, prerequisite list, and 3–10 daily problems (Beginner tier skews toward Easy)
    - Each problem: title, problem statement, difficulty, estimated time (30–60 min), at least one concrete scenario, at least one hint, solution outline
    - Design problems use scenario format instead of input/output pairs
    - At least 1 Easy and at least 1 Hard problem; problems ordered by difficulty; no duplicate sub-skills
    - _Requirements: 1.4, 2.1–2.5, 3.1–3.8, 5.4, 6.1, 6.3, 7.1–7.3, 8.3, 8.5_

  - [x] 11.2 Create `curriculum/design-concepts/solid-principles/README.md`
    - Concept: SOLID Principles — Difficulty Tier: Beginner — Prerequisites: Object-Oriented Design Principles
    - Same structure and quality requirements as 11.1
    - _Requirements: 1.4, 2.1–2.5, 3.1–3.8, 5.4, 6.1, 6.3, 7.1–7.3, 8.3, 8.5_

  - [x] 11.3 Create `curriculum/design-concepts/api-design/README.md`
    - Concept: API Design — Difficulty Tier: Beginner — Prerequisites: SOLID Principles
    - Same structure and quality requirements as 11.1
    - _Requirements: 1.4, 2.1–2.5, 3.1–3.8, 5.4, 6.1, 6.3, 7.1–7.3, 8.3, 8.5_

  - [x] 11.4 Create `curriculum/design-concepts/database-schema-design/README.md`
    - Concept: Database Schema Design — Difficulty Tier: Beginner — Prerequisites: None
    - Same structure and quality requirements as 11.1
    - _Requirements: 1.4, 2.1–2.5, 3.1–3.8, 6.1, 6.3, 7.1–7.3, 8.3, 8.5_

  - [x] 11.5 Create `curriculum/design-concepts/design-patterns-creational/README.md`
    - Concept: Design Patterns (Creational) — Difficulty Tier: Beginner — Prerequisites: Object-Oriented Design Principles, SOLID Principles
    - Same structure and quality requirements as 11.1
    - _Requirements: 1.4, 2.1–2.5, 3.1–3.8, 5.4, 6.1, 6.3, 7.1–7.3, 8.3, 8.5_

  - [x] 11.6 Create `curriculum/design-concepts/design-patterns-structural/README.md`
    - Concept: Design Patterns (Structural) — Difficulty Tier: Beginner — Prerequisites: SOLID Principles, Design Patterns (Creational)
    - Same structure and quality requirements as 11.1
    - _Requirements: 1.4, 2.1–2.5, 3.1–3.8, 5.4, 6.1, 6.3, 7.1–7.3, 8.3, 8.5_

  - [x] 11.7 Create `curriculum/design-concepts/design-patterns-behavioral/README.md`
    - Concept: Design Patterns (Behavioral) — Difficulty Tier: Beginner — Prerequisites: SOLID Principles, Design Patterns (Structural)
    - Same structure and quality requirements as 11.1
    - _Requirements: 1.4, 2.1–2.5, 3.1–3.8, 5.4, 6.1, 6.3, 7.1–7.3, 8.3, 8.5_

  - [x] 11.8 Create `curriculum/design-concepts/caching-strategies/README.md`
    - Concept: Caching Strategies — Difficulty Tier: Beginner — Prerequisites: None
    - Same structure and quality requirements as 11.1
    - _Requirements: 1.4, 2.1–2.5, 3.1–3.8, 6.1, 6.3, 7.1–7.3, 8.3, 8.5_

- [x] 12. Checkpoint — Verify Beginner Design Concepts modules
  - Ensure all 8 Beginner-tier Design Concepts modules are complete and consistent. Ask the user if questions arise.

- [x] 13. Create Design Concepts Intermediate-tier concept modules
  - [x] 13.1 Create `curriculum/design-concepts/load-balancing/README.md`
    - Concept: Load Balancing — Difficulty Tier: Intermediate — Prerequisites: Caching Strategies
    - Include concept overview, prerequisite list, and 3–10 daily problems with mixed difficulty (Intermediate tier)
    - Design problems use scenario format; at least 1 Easy and at least 1 Hard; ordered by difficulty; no duplicate sub-skills
    - Each problem: title, problem statement, difficulty, estimated time (30–60 min), at least one concrete scenario, at least one hint, solution outline
    - _Requirements: 1.4, 2.1–2.5, 3.1–3.8, 5.4, 6.1, 6.2, 7.1–7.3, 8.3, 8.5_

  - [x] 13.2 Create `curriculum/design-concepts/message-queues/README.md`
    - Concept: Message Queues — Difficulty Tier: Intermediate — Prerequisites: None
    - Same structure and quality requirements as 13.1
    - _Requirements: 1.4, 2.1–2.5, 3.1–3.8, 6.1, 6.2, 7.1–7.3, 8.3, 8.5_

  - [x] 13.3 Create `curriculum/design-concepts/rate-limiting/README.md`
    - Concept: Rate Limiting — Difficulty Tier: Intermediate — Prerequisites: API Design
    - Same structure and quality requirements as 13.1
    - _Requirements: 1.4, 2.1–2.5, 3.1–3.8, 5.4, 6.1, 6.2, 7.1–7.3, 8.3, 8.5_

  - [x] 13.4 Create `curriculum/design-concepts/consistent-hashing/README.md`
    - Concept: Consistent Hashing — Difficulty Tier: Intermediate — Prerequisites: Load Balancing
    - Same structure and quality requirements as 13.1
    - _Requirements: 1.4, 2.1–2.5, 3.1–3.8, 5.4, 6.1, 6.2, 7.1–7.3, 8.3, 8.5_

  - [x] 13.5 Create `curriculum/design-concepts/cap-theorem/README.md`
    - Concept: CAP Theorem — Difficulty Tier: Intermediate — Prerequisites: None
    - Same structure and quality requirements as 13.1
    - _Requirements: 1.4, 2.1–2.5, 3.1–3.8, 6.1, 6.2, 7.1–7.3, 8.3, 8.5_

  - [x] 13.6 Create `curriculum/design-concepts/data-partitioning/README.md`
    - Concept: Data Partitioning — Difficulty Tier: Intermediate — Prerequisites: Database Schema Design, Consistent Hashing, CAP Theorem
    - Same structure and quality requirements as 13.1
    - _Requirements: 1.4, 2.1–2.5, 3.1–3.8, 5.4, 6.1, 6.2, 7.1–7.3, 8.3, 8.5_

  - [x] 13.7 Create `curriculum/design-concepts/replication-strategies/README.md`
    - Concept: Replication Strategies — Difficulty Tier: Intermediate — Prerequisites: Database Schema Design, CAP Theorem
    - Same structure and quality requirements as 13.1
    - _Requirements: 1.4, 2.1–2.5, 3.1–3.8, 5.4, 6.1, 6.2, 7.1–7.3, 8.3, 8.5_

  - [x] 13.8 Create `curriculum/design-concepts/microservices-architecture/README.md`
    - Concept: Microservices Architecture — Difficulty Tier: Intermediate — Prerequisites: Load Balancing, Message Queues
    - Same structure and quality requirements as 13.1
    - _Requirements: 1.4, 2.1–2.5, 3.1–3.8, 5.4, 6.1, 6.2, 7.1–7.3, 8.3, 8.5_

  - [x] 13.9 Create `curriculum/design-concepts/url-shortener-design/README.md`
    - Concept: URL Shortener Design — Difficulty Tier: Intermediate — Prerequisites: API Design, Caching Strategies, Rate Limiting, Microservices Architecture
    - Same structure and quality requirements as 13.1
    - _Requirements: 1.4, 2.1–2.5, 3.1–3.8, 5.4, 6.1, 6.2, 7.1–7.3, 8.3, 8.5_

- [x] 14. Checkpoint — Verify Intermediate Design Concepts modules
  - Ensure all 9 Intermediate-tier Design Concepts modules are complete and consistent. Ask the user if questions arise.

- [x] 15. Create Design Concepts Advanced-tier concept modules
  - [x] 15.1 Create `curriculum/design-concepts/chat-system-design/README.md`
    - Concept: Chat System Design — Difficulty Tier: Advanced — Prerequisites: Message Queues, Microservices Architecture, URL Shortener Design
    - Include concept overview, prerequisite list, and 3–10 daily problems skewing toward Hard (Advanced tier)
    - Design problems use scenario format; at least 1 Easy and at least 1 Hard; ordered by difficulty; no duplicate sub-skills
    - Each problem: title, problem statement, difficulty, estimated time (30–60 min), at least one concrete scenario, at least one hint, solution outline
    - _Requirements: 1.4, 2.1–2.5, 3.1–3.8, 5.4, 6.1, 6.4, 7.1–7.3, 8.3, 8.5_

  - [x] 15.2 Create `curriculum/design-concepts/notification-system-design/README.md`
    - Concept: Notification System Design — Difficulty Tier: Advanced — Prerequisites: Message Queues, Microservices Architecture, Chat System Design
    - Same structure and quality requirements as 15.1
    - _Requirements: 1.4, 2.1–2.5, 3.1–3.8, 5.4, 6.1, 6.4, 7.1–7.3, 8.3, 8.5_

  - [x] 15.3 Create `curriculum/design-concepts/search-system-design/README.md`
    - Concept: Search System Design — Difficulty Tier: Advanced — Prerequisites: Caching Strategies, Microservices Architecture
    - Same structure and quality requirements as 15.1
    - _Requirements: 1.4, 2.1–2.5, 3.1–3.8, 5.4, 6.1, 6.4, 7.1–7.3, 8.3, 8.5_

  - [x] 15.4 Create `curriculum/design-concepts/news-feed-design/README.md`
    - Concept: News Feed Design — Difficulty Tier: Advanced — Prerequisites: Caching Strategies, Microservices Architecture, Search System Design
    - Same structure and quality requirements as 15.1
    - _Requirements: 1.4, 2.1–2.5, 3.1–3.8, 5.4, 6.1, 6.4, 7.1–7.3, 8.3, 8.5_

  - [x] 15.5 Create `curriculum/design-concepts/distributed-file-storage-design/README.md`
    - Concept: Distributed File Storage Design — Difficulty Tier: Advanced — Prerequisites: Data Partitioning, Replication Strategies
    - Same structure and quality requirements as 15.1
    - _Requirements: 1.4, 2.1–2.5, 3.1–3.8, 5.4, 6.1, 6.4, 7.1–7.3, 8.3, 8.5_

  - [x] 15.6 Create `curriculum/design-concepts/video-streaming-design/README.md`
    - Concept: Video Streaming Design — Difficulty Tier: Advanced — Prerequisites: Load Balancing, Caching Strategies, Microservices Architecture, Distributed File Storage Design
    - Same structure and quality requirements as 15.1
    - _Requirements: 1.4, 2.1–2.5, 3.1–3.8, 5.4, 6.1, 6.4, 7.1–7.3, 8.3, 8.5_

  - [x] 15.7 Create `curriculum/design-concepts/payment-system-design/README.md`
    - Concept: Payment System Design — Difficulty Tier: Advanced — Prerequisites: Microservices Architecture, Notification System Design
    - Same structure and quality requirements as 15.1
    - _Requirements: 1.4, 2.1–2.5, 3.1–3.8, 5.4, 6.1, 6.4, 7.1–7.3, 8.3, 8.5_

  - [x] 15.8 Create `curriculum/design-concepts/monitoring-and-observability/README.md`
    - Concept: Monitoring & Observability — Difficulty Tier: Advanced — Prerequisites: None
    - Same structure and quality requirements as 15.1
    - _Requirements: 1.4, 2.1–2.5, 3.1–3.8, 6.1, 6.4, 7.1–7.3, 8.3, 8.5_

  - [x] 15.9 Create `curriculum/design-concepts/capstone-end-to-end-system-design/README.md`
    - Concept: Capstone: End-to-End System Design — Difficulty Tier: Advanced — Prerequisites: URL Shortener Design, Chat System Design, Payment System Design, Video Streaming Design, Monitoring & Observability
    - Same structure and quality requirements as 15.1
    - _Requirements: 1.4, 2.1–2.5, 3.1–3.8, 5.4, 6.1, 6.4, 7.1–7.3, 8.3, 8.5_

- [x] 16. Checkpoint — Verify all Design Concepts modules
  - Ensure all 26 Design Concepts concept modules exist, are consistently formatted, prerequisite lists are correct, and the track README table/graph matches. Ask the user if questions arise.

- [x] 17. Final checkpoint — Full curriculum validation
  - Verify the complete curriculum: 52 concept modules across 2 tracks, all prerequisite graphs consistent between module documents and track READMEs, Mermaid diagrams render correctly, top-level README links valid. Ask the user if questions arise.

## Notes

- Concept modules are identified by concept name (not week number), per Requirement 1.4
- Each module contains 3–10 daily problems depending on concept depth, per Requirement 2.2
- The design document references a rigid week-numbering structure — follow the requirements as the source of truth
- Difficulty tier distribution: Beginner modules skew Easy, Intermediate modules are mixed, Advanced modules skew Hard
- Design Concepts track problems use concrete scenarios instead of input/output pairs
- Mermaid diagrams are rendered natively by GitHub, GitLab, and VS Code — no SVG export or pre-commit hook needed
- Checkpoints ensure incremental validation at natural breakpoints
