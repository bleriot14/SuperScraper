
```markdown
# 🕷️ SuperScraper Framework

A modular, multi-mission web-scraping framework built around **Selenium Grid**, **Task dispatching**, and **Strategy-based traversal** (BFS / DFS).  
Designed for extensibility and long-running distributed scraping tasks.

---

## 🧩 Core Idea

**SuperScraper** manages the overall scraping ecosystem:

- Multiple **Scrape Missions** (e.g. Amazon / eBay)
- A shared **TaskManager** that distributes URL requests to Selenium Grid nodes
- Pluggable **Traversal Strategies** (BFS, DFS, or custom)
- Centralized **DataManager** (writes to SQL + Neo4j)
- Unified **AppLogger** for contextual logging

Missions only define *what to scrape* and *how to extract* data.  
They never deal with Selenium directly — that’s handled by the TaskManager.

---

## ⚙️ Architecture Overview

```

SuperScraper
├── AppLogger          → Centralized contextual logging
├── DataManager        → SQL + Neo4j data persistence
├── TaskManager        → Manages Selenium Grid + task queue
│     └── RemoteWebDrivers[3]
├── ScrapeMissionBase  → Abstract mission class
│     ├── TraversalStrategy (BFS / DFS)
│     ├── AmazonMenuTreeScraper
│     │      └── AmazonMenuTreeBFSScraper (preset)
│     └── EbayMenuTreeScraper
│            └── EbayMenuTreeDFSScraper (preset)
└── main.py            → Composition root

````

---

## 🧠 Key Design Principles

| Concept | Pattern Used | Responsibility |
|----------|---------------|----------------|
| **SuperScraper** | *Facade / Composition Root* | Bootstraps system, wires components, runs missions |
| **ScrapeMissionBase** | *Template Method* | Defines scraping workflow: start → parse → enqueue |
| **TraversalStrategy** | *Strategy* | Controls traversal order (BFS, DFS, etc.) |
| **TaskManager** | *Observer (push model)* | Dispatches URLs, notifies correct mission on completion |
| **DataManager** | *Facade* | Abstracts multiple storage systems |
| **AppLogger** | *Singleton (DI)* | Provides context-aware logging across components |

---

## 🚀 Runtime Flow

1. **Initialization**
   - `SuperScraper` creates `Logger`, `DataManager`, `TaskManager`
   - Connects to Selenium Grid (`http://localhost:4444/wd/hub`)
   - Launches 3 Chrome sessions

2. **Mission Registration**
   ```python
   amazon1 = AmazonMenuTreeBFSScraper(task_manager, data_manager, logger)
   amazon2 = AmazonMenuTreeBFSScraper(task_manager, data_manager, logger)
   ebay1   = EbayMenuTreeDFSScraper(task_manager, data_manager, logger)
   scraper.registerMission(amazon1)
   scraper.registerMission(amazon2)
   scraper.registerMission(ebay1)
````

3. **Execution**

   * `SuperScraper.startAll()` starts the TaskManager worker loop.
   * Each mission’s `start()` submits root URLs to the TaskManager.
   * TaskManager dispatches jobs to free Selenium drivers.
   * On completion, it **pushes results back** to the relevant mission via `onTaskResult()`.

4. **Processing**

   * The mission parses the HTML, saves entities via `DataManager`, and enqueues new URLs via its traversal strategy.

5. **Persistence**

   * All extracted data is saved to both SQL and Neo4j (parallel writes).

---

## 🧱 Class Responsibilities

### `ScrapeMissionBase`

* Defines `start()`, `onTaskResult()`, `parseAndSchedule()`
* Keeps:

  * `visited` set
  * `TraversalStrategy` instance (queue/stack)
* Never directly uses Selenium; interacts through `TaskManager`.

### `TraversalStrategy`

* Abstract interface:

  ```python
  addInitial(urls)
  addNew(urls)
  hasNext()
  next()
  ```
* Implementations:

  * `BFSTraversal`: FIFO queue
  * `DFSTraversal`: LIFO stack

### `TaskManager`

* Holds `drivers = [driver1, driver2, driver3]`
* Round-robin or first-free assignment
* After fetch:

  * Builds `TaskResult`
  * Calls `mission.onTaskResult(result)`

### `DataManager`

* Single interface for all persistence:

  ```python
  saveProduct(...)
  saveCategory(...)
  saveRelation(...)
  ```
* Handles SQL + Neo4j connections internally.

### `AppLogger`

* Provides:

  ```python
  logger.info("message")
  logger.withComponent("TaskManager").error("failed to acquire driver")
  ```

---

## 🧪 Example `main.py`

```python
from core import SuperScraper, AmazonMenuTreeBFSScraper, EbayMenuTreeDFSScraper

if __name__ == "__main__":
    scraper = SuperScraper()
    scraper.configure()

    amazon1 = AmazonMenuTreeBFSScraper(scraper.taskManager, scraper.dataManager, scraper.logger)
    amazon2 = AmazonMenuTreeBFSScraper(scraper.taskManager, scraper.dataManager, scraper.logger)
    ebay1   = EbayMenuTreeDFSScraper(scraper.taskManager, scraper.dataManager, scraper.logger)

    scraper.registerMission(amazon1)
    scraper.registerMission(amazon2)
    scraper.registerMission(ebay1)

    scraper.startAll()
```

---

## 🧭 Extending the Framework

| Goal                   | What to Implement                          |
| ---------------------- | ------------------------------------------ |
| Add a new site         | Create new subclass of `ScrapeMissionBase` |
| Change traversal order | Plug in different `TraversalStrategy`      |
| Support another DB     | Extend `DataManager` methods               |
| Add logging filters    | Modify `AppLogger.withComponent()`         |
| Scale driver count     | Adjust `TaskManager.drivers` init logic    |

---

## ⚡ Future Enhancements

* [ ] Async / await driver dispatching
* [ ] Distributed message queue (RabbitMQ / Redis)
* [ ] Health monitor & retry system
* [ ] Configurable retry policies per mission
* [ ] Web dashboard for monitoring active missions

---

## 🧑‍💻 Author Notes

* Clean, extensible architecture — small classes, clear boundaries.
* Strategy pattern ensures traversal logic can evolve without rewriting missions.
* TaskManager remains driver-agnostic and reusable for any grid topology.

---

> “Don’t let scraping code rot into spaghetti.
> Build it like a distributed system — even if it runs on your laptop.”

```
