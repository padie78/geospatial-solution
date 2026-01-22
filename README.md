# Geospatial Asset Management System

A high-performance, event-driven solution for processing and visualizing geospatial data. The system is fully containerized and leverages a microservices architecture to handle file monitoring, asynchronous processing, and real-time API delivery.

---

## Architecture Overview

The system follows a **Monorepo** pattern with a microservices-oriented approach, divided into four main components:

1.  **Watcher (Service):** A Node.js background worker that monitors the `data/incoming` directory for new geospatial files.
2.  **Worker (Service):** The processing engine that parses raw data, performs transformations, and stores results.
3.  **API (Backend):** A NestJS-based RESTful API that serves processed geospatial data and handles client requests.
4.  **Frontend (App):** An Angular application for visualizing assets on an interactive map.
5.  **Infrastructure:** **Redis** acts as the message broker and caching layer, while **PostgreSQL/PostGIS** (optional/recommended) handles spatial persistence.

---

## File Processing Flow

The system implements a reliable "Inbox/Outbox" pattern:

1.  **Ingestion:** A file is dropped into `./data/incoming`.
2.  **Detection:** The **Watcher** service detects the file, validates its format, and pushes a "New Job" event to a **Redis Queue**.
3.  **Processing:** The **Worker** picks up the job from Redis, extracts the geospatial coordinates, and moves the file to `./data/processed`.
4.  **Indexing:** The processed data is cached in Redis for immediate API availability.
5.  **Notification:** The API is notified of the new data, making it instantly visible to the Frontend.

<img width="990" height="620" alt="image" src="https://github.com/user-attachments/assets/bf7ef005-3d35-4bec-a86e-7fafeb47c76c" />


---

## Redis Usage and Cache Strategy

Redis is the backbone of the system's performance, used in two ways:

* **Message Broker (BullMQ/PubSub):** To decouple the Watcher from the Worker. This ensures that if the Worker is busy or down, jobs are not lost—they stay queued in Redis.
* **Cache Strategy (Write-Through):**
    * **Geospatial Indexing:** Processed coordinates are stored in Redis using `GEOADD` for ultra-fast proximity queries.
    * **TTL (Time-To-Live):** Data is cached with an expiration policy to ensure the frontend always reflects fresh data without overloading the primary database.

---

## How to Run Locally

### Prerequisites
* **Docker & Docker Compose V2** (Installed and running)
* **Node.js v18+** (For local orchestration)

### Setup & Execution
1.  **Clone the repository:**
    ```bash
    git clone https://github.com/padie78/geospatial-solution
    cd geospatial-solution
    ```

2.  **Install all dependencies:**
    ```bash
    npm run install-all
    ```

3.  **Launch the System:**
    ```bash
    npm run docker:up
    ```
    *This command will automatically fix directory permissions, build all images, and start the containers.*

4.  **Access the Application:**
    * **Frontend:** `http://localhost:4200`
    * **API:** `http://localhost:3000`

5. **Admin & Monitoring Dashboards:**
   You can monitor the infrastructure and data state through the following management interfaces:
    * **Redis Stack Browser:** [http://localhost:8001/redis-stack/browser](http://localhost:8001/redis-stack/browser)
        *Inspect the cache layer, geospatial indexes, and real-time keys.*
    * **Mongo Express:** [http://localhost:8082/db/geospatial_metadata](http://localhost:8082/db/geospatial_metadata)
        *Explore the metadata documents stored for each processed asset.*
    * **RabbitMQ Management:** [http://localhost:15672/](http://localhost:15672/)
        * **User:** `admin` | **Password:** `admin123`
        *Monitor the `create_asset` queue, exchange status, and message rates.*
    ---

## Key Assumptions and Trade-offs

* **Assumption - Local Storage:** We assume the host system shares a volume with Docker for the `data/` folder. In a production cloud environment, this would be replaced by **AWS S3** or **Google Cloud Storage**.
* **Trade-off - Eventual Consistency:** Using Redis for immediate visualization provides speed, but there is a slight delay between file ingestion and the final database persistence (Eventual Consistency).
* **Assumption - File Formats:** The current implementation assumes standard geospatial formats (JSON/CSV). Validation is performed at the Watcher level to prevent Worker crashes.
* **Scalability Trade-off:** We chose a single-worker setup for this MVP. However, the architecture is designed to scale horizontally by simply increasing the `worker` replicas in Docker Compose.
