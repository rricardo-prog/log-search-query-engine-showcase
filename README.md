# High-Throughput Log Search & Management Engine (C++)

[![Language: C++17](https://img.shields.io/badge/Language-C%2B%2B17-blue.svg)](https://isocpp.org/)
[![Focus: Data Structures & Algorithms](https://img.shields.io/badge/Focus-High--Performance%20Indexing-green.svg)]()

A high-performance command-line log query engine developed in C++ designed to ingest, index, and interactively query massive server log datasets. The engine supports sub-linear multi-criteria search (exact timestamps, temporal ranges, categorical tags, and multi-keyword intersections) and maintains dynamic user excerpt collections under strict memory and runtime budgets.

---

> ### 🔒 Academic Integrity Notice
> This project was developed as part of **EECS 281 (Data Structures and Algorithms) at the University of Michigan**. In compliance with the University of Michigan College of Engineering Honor Code and course copyright policies, the source code repository is kept strictly private. 
> 
> Detailed architecture, architectural trade-offs, and profiling results are outlined below. Live technical walkthroughs or private code demonstrations are available upon request for interview purposes.

---

## ⚡ Engineering Highlights & Key Metrics

* **Fast Parsing & Memory Footprint:** Employs zero-copy tokenization and numeric timestamp compression (converting string timestamps into unified 64-bit integers) to eliminate string allocation overhead across hundreds of thousands of entries.
* **$O(\log N)$ Temporal Search:** Implements binary search algorithms (`std::lower_bound` / `std::upper_bound`) across a master sorted index for arbitrary time-window queries.
* **Amortized $O(1)$ Category Retrieval:** Leverages hashed lookup tables mapping normalized categories directly to contiguous entry ID vectors.
* **Fast Multi-Keyword Inverted Index:** Implements tokenized keyword parsing with set intersection optimization, retrieving multi-word log intersections in $O(k \cdot m)$ time.
* **Memory Safety:** Validated clean under `valgrind` with zero memory leaks, dangling pointers, or invalid heap reads/writes.

---

## 🛠️ System Architecture & Data Structure Design

The engine separates data storage from query indexing to minimize redundancy and maximize cache locality:

### 1. Master Log Repository
* Rather than duplicating raw log data across indices, the engine stores raw log records once in a centralized, contiguous `std::vector<LogEntry>`.
* Log records store timestamps as packed 64-bit integers (`uint64_t`), compressing the raw `mm:dd:hh:mm:ss` representation into total monotonic seconds/ticks. This accelerates comparative evaluations into single CPU register instructions.

### 2. Multi-Index Architecture
* **Timestamp Range Queries ($O(\log N)$):** Pointers/indices sorted chronologically. Range and exact queries resolve through binary search bounds (`std::equal_range` / `std::lower_bound`).
* **Category Hashing ($O(1)$ lookup):** An `std::unordered_map<std::string, std::vector<uint32_t>>` maps case-insensitive categories to arrays of master log indices.
* **Inverted Keyword Index ($O(k \cdot m)$):** Alphanumeric tokens are extracted from message bodies and mapped via hash table to sorted vectors of entry IDs. When evaluating multi-keyword queries (`k1 AND k2 AND k3`), the engine performs a sorted two-pointer intersection starting with the smallest candidate posting list to prune the search space early.

### 3. Excerpt List Management
* The excerpt list tracks an active investigation workspace. Rather than copying log structs, it stores lightweight 32-bit `entryID` handles.
* Supports reordering, deletion, sorting (by timestamp, category, and entry ID tie-breakers), and batch-appending previous search results in linear time.

---

## ⏱️ Algorithmic Complexity

| Operation | Query Type | Time Complexity | Auxiliary Space |
| :--- | :--- | :--- | :--- |
| **Ingestion & Indexing** | Batch startup file ingestion | $O(N \log N + W)$ | $O(N)$ |
| **Exact Timestamp (`m`)** | Point search | $O(\log N + R)$ | $O(1)$ |
| **Timestamp Range (`t`)** | Temporal window `[t1, t2]` | $O(\log N + R)$ | $O(1)$ |
| **Category Search (`c`)** | Tag match | $O(1)$ lookup + $O(R)$ | $O(1)$ |
| **Keyword Search (`k`)** | Multi-token conjunction | $O(k \cdot m)$ | $O(R)$ |
| **Excerpt Sort (`s`)** | Dynamic workspace sort | $O(E \log E)$ | $O(1)$ |

*Where $N$ is total master logs, $R$ is matching entries, $k$ is query keywords, $m$ is average postings length, and $E$ is excerpt list size.*

---

## 💻 Tech Stack & Tooling

* **Language:** C++ (C++17 standard)
* **Standard Library:** Containers (`vector`, `unordered_map`, `string_view`), Algorithms (`lower_bound`, `sort`, `set_intersection`)
* **Build System:** GNU `Make` / `g++` (`-O3`, `-Wall`, `-Wextra`, `-pedantic`)
* **Testing & Diagnostics:** custom edge-case test suites (handling malformed CLI flags, boundary timestamps, non-alphanumeric tokenizers).

---

## 📈 Key Takeaways & Systems Learnings

* **Memory Hierarchy & Cache Locality:** Minimized heap fragmentation by relying on flat contiguous vectors and index handles rather than pointer-heavy node-based trees (`std::map`, `std::list`).
* **Index Trade-Offs:** Balanced up-front indexing overhead during initialization to guarantee sub-millisecond CLI responsiveness during live interactive filtering.
* **Defensive Error Handling:** Built robust CLI and stream-state validation (`std::cin.fail()`) to handle unexpected tokens and malformed queries without crashing.
