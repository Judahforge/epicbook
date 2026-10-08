# Environment Variables and Ports

## 1. Environment Variables

The application uses environment variables to configure the database connection and application environment without exposing sensitive credentials in the project documentation.

| Environment Variable | Purpose                                        |
| -------------------- | ---------------------------------------------- |
| `DB_NAME`            | Name of the MySQL database                     |
| `DB_USER`            | MySQL application username                     |
| `DB_PASSWORD`        | MySQL application password                     |
| `DB_ROOT_PASSWORD`   | MySQL root password                            |
| `NODE_ENV`           | Sets the backend environment to production     |
| `PORT`               | Port used by the backend application           |
| `JAWSDB_URL`         | Database connection string used by the backend |

**Security Note:** Real passwords, credentials, and secret values are not included in this document. Only the environment-variable names and their purposes are documented.

## 2. Ports

| Service               | Port | Exposure                     |
| --------------------- | ---: | ---------------------------- |
| Reverse Proxy (Nginx) |   80 | Public: `80:80`              |
| Frontend              |   80 | Internal Docker network only |
| Backend               |   80 | Internal Docker network only |
| Database (MySQL)      | 3306 | Internal Docker network only |

The **reverse proxy** is the only service exposed directly to the host. Public HTTP requests enter through port 80 and are routed by Nginx to the appropriate frontend or backend service.

The frontend and backend both listen on port 80 inside their containers. The MySQL database listens on port 3306 and is accessible only through the backend network.

## 3. Docker Networks

The application uses two Docker networks:

### `front-tier`

Connects:

* `reverse-proxy`
* `frontend`
* `backend`

This network allows the reverse proxy to communicate with the frontend and backend.

### `back-tier`

Connects:

* `backend`
* `database`

The `back-tier` network is configured as an internal network, preventing direct external access to the database.

## 4. Persistent Database Storage

The MySQL database uses a Docker named volume called `db_data`:

```text
db_data → /var/lib/mysql
```

This provides persistent storage for MySQL data. The database data remains available even if the database container is removed and recreated.

The project also mounts the database initialization scripts into the MySQL container:

```text
./db/BuyTheBook_Schema.sql → /docker-entrypoint-initdb.d/01-schema.sql
./db/author_seed.sql        → /docker-entrypoint-initdb.d/02-authors.sql
./db/books_seed.sql         → /docker-entrypoint-initdb.d/03-books.sql
```

These scripts initialize the database schema and seed data when the database is first initialized.

## 5. Health Checks

Docker Compose uses health checks to verify that the services are ready before dependent services start.

### Frontend Health Check

The frontend checks whether its HTTP server is responding:

```bash
wget -q --spider http://localhost:80
```

### Backend Health Check

The backend checks whether its HTTP service is responding:

```bash
curl -f http://localhost:80/
```

### Database Health Check

MySQL is checked using:

```bash
mysqladmin ping -h localhost -uroot -p${DB_ROOT_PASSWORD}
```

The reverse proxy waits for both the frontend and backend services to become healthy before starting.

## 6. Architecture Summary

```text
                    PUBLIC USER
                         |
                         | HTTP :80
                         v
              +----------------------+
              |  Reverse Proxy       |
              |  Nginx :80           |
              +----------------------+
                         |
                  front-tier network
                    /           \
                   /             \
                  v               v
        +---------------+   +---------------+
        |   Frontend    |   |    Backend    |
        |     :80       |   |      :80      |
        +---------------+   +---------------+
                                  |
                           back-tier network
                                  |
                                  v
                         +----------------+
                         | MySQL Database |
                         |     :3306      |
                         +----------------+
                                  |
                                  v
                         +----------------+
                         | Docker Volume  |
                         |    db_data     |
                         +----------------+
```

### Key Points

* Port `80` is the public entry point through the Nginx reverse proxy.
* Frontend and backend services use internal port `80`.
* MySQL uses internal port `3306`.
* `front-tier` connects the reverse proxy, frontend, and backend.
* `back-tier` connects the backend and database and is configured as an internal network.
* `db_data` provides persistent storage for MySQL.
* Health checks verify that the frontend, backend, and database are operational.
* Sensitive credential values are intentionally excluded from this document.

