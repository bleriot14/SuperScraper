

````markdown
# 🕸️ SuperScraper Framework

A modular, object-oriented web-scraping framework designed for distributed crawling using **Selenium Grid**.  
It provides a clean separation between scraping logic, task management, data persistence, and logging — all orchestrated by a single composition root.

---

## 🚀 Overview

SuperScraper coordinates multiple **Scraper** modules running on a Selenium Grid with multiple Chrome instances.  
Each scraper defines *what* to crawl, while the **TaskManager** decides *how* and *when* to execute those requests.  
Results are pushed back to the scrapers asynchronously, then persisted by the **DataManager**.

The design follows several key **Gang of Four (GoF)** patterns to ensure scalability, clarity, and maintainability.

---

## 🧩 Core Architecture

### **1. SuperScraper**
- Acts as the **composition root** and light **Facade**.
- Initializes and wires:
  - `AppLogger`
  - `TaskManager`
  - `DataManager`
  - One or more `ScraperBase` implementations.
- Starts and stops the entire system lifecycle.

### **2. TaskManager**
- Central orchestrator for scraping tasks.
- Holds direct connections to the Selenium Grid (e.g., 3 Chrome nodes).
- Maintains an internal queue of `TaskRequest` objects.
- Assigns each request to an available driver.
- Pushes back results (`TaskResult`) to the corresponding scraper once completed.
- Implements an **Observer pattern** (push model) toward scrapers.

### **3. ScraperBase (Abstract Class)**
- Defines the **Template Method** structure for all scrapers.
- Handles task submission, result reception, and error routing.
- Derived classes (e.g., `AmazonMenuBFSScraper`) override `handlePage()` to define specific logic.
- Can enqueue multiple URLs concurrently to the `TaskManager`.

### **4. DataManager**
- Single entry point for persistence.
- Supports multiple storage targets (e.g., SQL and Neo4j) — effectively a light **Strategy** layer.
- Each scraper can write to one or more stores in parallel via `DataManager`.

### **5. AppLogger**
- Provides contextual logging across all modules.
- Designed as a lightweight **Singleton** via dependency injection.
- Offers simple methods: `info()`, `error()`, `withComponent()`.

---

## 🧠 Design Patterns Used

| Pattern | Role in Architecture |
|----------|----------------------|
| **Observer** | TaskManager pushes results to Scrapers (`onTaskResult`) |
| **Template Method** | ScraperBase defines workflow, subclasses override `handlePage()` |
| **Dependency Injection** | SuperScraper wires and injects dependencies |
| **Facade** | SuperScraper exposes a unified system interface |
| **Strategy (light)** | DataManager writes to multiple storage types |
| **Singleton (light)** | Shared AppLogger instance |

---

## 🔧 Example Flow

```text
[SuperScraper]
    ↓ creates
[TaskManager] — manages —> [3 Selenium Chrome Nodes]
    ↑                       ↑
registers                  executes
[ScraperBase] <—— results —— [TaskManager]
    ↓ writes
[DataManager] → SQL / Neo4j
````

1. **SuperScraper** bootstraps everything.
2. **ScraperBase** submits multiple `TaskRequest`s.
3. **TaskManager** distributes them to free Chrome drivers.
4. When pages finish loading, **TaskManager** pushes back **TaskResult** objects to the scraper.
5. The scraper processes the data and saves it via **DataManager**.

---

## 📦 Example Classes

```python
class SuperScraper:
    def configure(self):
        self.logger = AppLogger()
        self.data_manager = DataManager(self.logger)
        self.task_manager = TaskManager(self.logger, self.data_manager)
        self.scrapers = {
            "amazon_bfs": AmazonMenuBFSScraper(
                "amazon_bfs", self.task_manager, self.data_manager, self.logger
            )
        }

    def start(self):
        for s in self.scrapers.values():
            self.task_manager.register_scraper(s)
            s.start()
```

---

## ⚙️ Tech Stack

* **Language:** Python / C++ / C# adaptable architecture
* **Web Automation:** Selenium Grid
* **Persistence:** SQL, Neo4j
* **Logging:** Contextual AppLogger
* **Concurrency:** Internal async/threads (configurable)

---

## 🧪 Example Scraper

```python
class AmazonMenuBFSScraper(ScraperBase):
    def start(self):
        self.queue_url("https://www.amazon.com/menu")

    def handle_page(self, result):
        links = extract_links(result.html)
        for url in links:
            if url not in self.visited:
                self.visited.add(url)
                self.queue_url(url)
                self.data_manager.save_category(url)
```

---

## 🧱 Future Extensions

* Retry policies per scraper
* Advanced driver health monitoring
* REST API layer for external control
* Configurable persistence strategies
* Plugin loader for dynamic scraper modules

---

## 🧭 Folder Structure (suggested)

```
super_scraper/
│
├── core/
│   ├── task_manager.py
│   ├── scraper_base.py
│   ├── data_manager.py
│   ├── logger.py
│   └── super_scraper.py
│
├── scrapers/
│   ├── amazon_bfs_scraper.py
│
└── README.md
```

---

## 🧑‍💻 Author

**Developed by:** Bleriot14
**Purpose:** Modular, scalable scraper framework designed for distributed crawling via Selenium Grid.

---

## 🏁 License

MIT License © 2025 Bleriot14

```

```
