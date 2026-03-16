# Requirements Document

## Introduction

A computer science training curriculum delivered as a collection of well-structured markdown documents, designed to cover roughly a year of study. The curriculum provides a structured, daily practice plan for mastering algorithms, data structures, and design concepts. It is split into two tracks: "Algorithms & Data Structures" and "Design Concepts." Each concept module focuses on a single topic (e.g., backtracking, divide and conquer, caching strategies), and contains a set of daily problems related to that theme. Some concepts may need more or fewer than 5 days depending on their depth and complexity. The daily time commitment is 30 minutes to 1 hour per problem. The deliverable is a directory of markdown files organized by track and concept, readable by humans and navigable via any markdown viewer.

## Glossary

- **Curriculum**: The complete structured training plan containing all tracks and concept modules, delivered as a collection of markdown files. Designed to cover roughly 52 weeks but flexible in pacing.
- **Track**: A top-level grouping of concept modules. The curriculum has two tracks: "Algorithms & Data Structures" and "Design Concepts".
- **Concept_Module**: A study unit within a track, focused on one Concept, containing a set of daily problems. A module typically spans about a week (5 problems) but may be shorter or longer depending on the concept's depth.
- **Daily_Problem**: A single practice problem within a Concept_Module, solvable in 30–60 minutes.
- **Concept**: A computer science topic that serves as the theme for a Concept_Module (e.g., backtracking, binary search, load balancing).
- **Difficulty_Level**: A classification of a Daily_Problem as "Easy", "Medium", or "Hard".

## Requirements

### Requirement 1: Curriculum Structure

**User Story:** As a learner, I want the curriculum organized into two distinct tracks, so that I can follow a structured training plan.

#### Acceptance Criteria

1. THE Curriculum SHALL contain exactly two Tracks: "Algorithms & Data Structures" and "Design Concepts".
2. THE Curriculum SHALL contain approximately 52 Concept_Modules distributed across the two Tracks.
3. THE Curriculum SHALL assign each Concept_Module to exactly one Track.
4. THE Curriculum SHALL identify each Concept_Module by its Concept name rather than a sequential week number, allowing learners to navigate topics in any order guided by the prerequisite graph.

### Requirement 2: Weekly Concept Themes

**User Story:** As a learner, I want each module to focus on a single concept, so that I can build deep understanding of one topic at a time.

#### Acceptance Criteria

1. THE Concept_Module SHALL have exactly one associated Concept.
2. THE Concept_Module SHALL contain between 3 and 10 Daily_Problems, depending on the depth and complexity of the Concept.
3. THE Concept_Module SHALL include the Concept name and a brief description explaining the Concept and its relevance.
4. THE Curriculum SHALL assign a unique Concept to each Concept_Module within the same Track (no repeated Concepts within a Track).
5. THE Concept_Module SHALL specify a recommended prerequisite list of prior Concept_Modules (may be empty for foundational modules).

### Requirement 3: Daily Problem Definition

**User Story:** As a learner, I want each day to present a different problem within the weekly theme, so that I can practice varied applications of the same concept.

#### Acceptance Criteria

1. THE Daily_Problem SHALL include a title, a problem statement, a Difficulty_Level, an estimated completion time, and at least one hint.
2. THE Daily_Problem SHALL have an estimated completion time between 30 and 60 minutes.
3. THE Daily_Problem SHALL belong to exactly one Concept_Module and relate directly to that module's Concept.
4. THE Concept_Module SHALL contain Daily_Problems with a mix of Difficulty_Levels: at least one "Easy" and at least one "Hard" problem per module.
5. THE Daily_Problem SHALL include a solution outline or approach description to guide self-study.
6. WHEN multiple Daily_Problems exist within a Concept_Module, THE module SHALL order the problems from lower to higher Difficulty_Level.
7. THE Daily_Problem SHALL include at least one example input/output pair (for algorithmic problems) or at least one concrete scenario (for design problems).
8. THE Concept_Module SHALL contain Daily_Problems that exercise different aspects of the Concept (no two problems in the same module shall test the identical sub-skill).

### Requirement 4: Algorithms & Data Structures Track Content

**User Story:** As a learner, I want the Algorithms & Data Structures track to cover core CS concepts progressively, so that I can systematically train on fundamental problem-solving techniques.

#### Acceptance Criteria

1. THE "Algorithms & Data Structures" Track SHALL include Concept_Modules covering at minimum the following Concepts: arrays, linked lists, stacks, queues, hash maps, trees, binary search trees, graphs, heaps, tries, sorting algorithms, binary search, two pointers, sliding window, recursion, backtracking, divide and conquer, dynamic programming, greedy algorithms, bit manipulation, union find, topological sort, interval problems, string manipulation, and matrix traversal.
2. THE "Algorithms & Data Structures" Track SHALL order Concept_Modules from foundational data structures to advanced algorithmic techniques.
3. THE "Algorithms & Data Structures" Track SHALL allocate approximately 26 Concept_Modules.
4. WHEN a Concept has natural prerequisites (e.g., graphs before topological sort), THE Concept_Module SHALL list those prerequisites explicitly.

### Requirement 5: Design Concepts Track Content

**User Story:** As a learner, I want the Design Concepts track to cover system design and software design topics, so that I can build skills in designing scalable and maintainable systems.

#### Acceptance Criteria

1. THE "Design Concepts" Track SHALL include Concept_Modules covering at minimum the following Concepts: object-oriented design principles, SOLID principles, API design, database schema design, caching strategies, load balancing, message queues, microservices architecture, rate limiting, consistent hashing, CAP theorem, data partitioning, replication strategies, design patterns (creational), design patterns (structural), design patterns (behavioral), URL shortener design, chat system design, notification system design, search system design, news feed design, distributed file storage design, video streaming design, and payment system design.
2. THE "Design Concepts" Track SHALL order Concept_Modules from foundational design principles to complex end-to-end system design exercises.
3. THE "Design Concepts" Track SHALL allocate approximately 26 Concept_Modules.
4. WHEN a design Concept builds on prior knowledge (e.g., caching before CDN design), THE Concept_Module SHALL list those prerequisites explicitly.

### Requirement 6: Difficulty Progression

**User Story:** As a learner, I want the curriculum to progress in difficulty over the year, so that I build confidence early and tackle harder material as my skills grow.

#### Acceptance Criteria

1. THE Curriculum SHALL assign each Concept_Module an overall difficulty tier: "Beginner", "Intermediate", or "Advanced".
2. WITHIN each difficulty tier, THE Concept_Modules SHALL progress from simpler to more complex Concepts.
3. THE Daily_Problems in "Beginner" tier modules SHALL skew toward "Easy" problems.
4. THE Daily_Problems in "Advanced" tier modules SHALL skew toward "Hard" problems.

### Requirement 7: Problem Quality and Variety

**User Story:** As a learner, I want each problem to be well-defined and varied, so that I get meaningful practice without repetitive exercises.

#### Acceptance Criteria

1. THE Daily_Problem SHALL have a problem statement that is self-contained and understandable without external references.
2. THE Daily_Problem SHALL include at least one example input/output pair (for algorithmic problems) or at least one concrete scenario (for design problems).
3. THE Concept_Module SHALL contain Daily_Problems that exercise different aspects of the Concept (no two problems in the same module shall test the identical sub-skill).

### Requirement 8: Markdown Document Organization

**User Story:** As a learner, I want the curriculum files organized in a clear directory structure, so that I can easily navigate to any week or track using a file browser or markdown viewer.

#### Acceptance Criteria

1. THE Curriculum SHALL be organized as markdown files in a directory structure grouped by Track and then by Concept_Module.
2. THE Curriculum SHALL include a top-level README.md that lists all Tracks, Concept_Modules, and links to each module document.
3. EACH Concept_Module markdown document SHALL contain all Daily_Problems for that module, including the Concept description, prerequisites, and each problem's full definition.
4. EACH Track SHALL have a README.md listing all Concept_Modules in that Track with links and a brief summary of each module's Concept.
5. THE Curriculum markdown files SHALL use consistent heading levels and formatting conventions across all documents.

### Requirement 9: Prerequisite Knowledge Graph

**User Story:** As a learner, I want to see a visual graph of concept prerequisites for each track, so that I can understand the learning path and know which topics I need to master before moving to the next ones.

#### Acceptance Criteria

1. THE "Algorithms & Data Structures" Track README.md SHALL include a Mermaid diagram showing the prerequisite relationships between all Concepts in that Track, from foundational (root) concepts to advanced (leaf) concepts.
2. THE "Design Concepts" Track README.md SHALL include a Mermaid diagram showing the prerequisite relationships between all Concepts in that Track, from foundational (root) concepts to advanced (leaf) concepts.
3. THE Mermaid diagrams SHALL represent Concepts as nodes and prerequisite relationships as directed edges (from prerequisite to dependent Concept).
4. THE Mermaid diagrams SHALL be renderable by standard Mermaid-compatible markdown viewers (e.g., GitHub, GitLab, VS Code).
5. THE prerequisite graph SHALL be consistent with the prerequisite lists defined in each Concept_Module document.
6. THE top-level README.md SHALL include a simplified overview Mermaid diagram showing the high-level progression across both Tracks.

### Requirement 10: SVG Diagram Assets

**User Story:** As a learner, I want the prerequisite graphs available as SVG images embedded in the markdown files, so that the diagrams render correctly in any markdown viewer regardless of Mermaid support.

#### Acceptance Criteria

1. THE Curriculum SHALL include an `assets/` directory for storing generated diagram files.
2. EACH Mermaid prerequisite diagram SHALL be exported as an SVG file and stored in the `assets/` directory.
3. THE markdown files SHALL embed the SVG diagrams using standard markdown image syntax, referencing the corresponding file in `assets/`.
4. THE SVG files SHALL be named descriptively to match their corresponding Track (e.g., `algorithms-prerequisite-graph.svg`, `design-prerequisite-graph.svg`, `overview-graph.svg`).
5. THE markdown files SHALL include both the inline Mermaid code block and the embedded SVG image, so that the diagram is viewable via either Mermaid rendering or direct SVG display.

### Requirement 11: Automated SVG Generation on Commit

**User Story:** As a contributor, I want the Mermaid SVG diagrams to be automatically regenerated before each git commit, so that the SVG assets are always in sync with the Mermaid source in the markdown files.

#### Acceptance Criteria

1. THE repository SHALL include a pre-commit git hook that regenerates all SVG diagram files from the Mermaid code blocks in the markdown files.
2. THE pre-commit hook SHALL update the SVG files in the `assets/` directory before the commit is finalized.
3. IF the Mermaid code blocks have not changed since the last generation, THE hook SHALL skip regeneration to avoid unnecessary processing.
4. THE pre-commit hook SHALL fail the commit with a descriptive error message if the Mermaid CLI tool is not installed or SVG generation fails.
