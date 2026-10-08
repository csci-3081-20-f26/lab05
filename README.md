# Lab 05: Data Layer Integration & Querying

## Goal

In this lab you will connect a FastAPI service to PostgreSQL, expose database-backed endpoints, and verify each endpoint with a small custom `httpx` command line client. By the end of the lab, your API should be able to read, create, update, and delete records while returning JSON responses that prove the database changed.

## What You Will Learn

1. How to run PostgreSQL and FastAPI with Docker Compose.
2. How to inject database configuration with environment variables.
3. How to initialize tables and seed data from SQL.
4. How to organize a simple data layer with DTOs, repositories, services, and routes.
5. How to call an API from a custom Python `httpx` client.
6. How to verify CRUD behavior with read-after-write checks.

## What's in lab05

- `app/main.py`: FastAPI application factory and startup database pool wiring.
- `app/routes/db.py`: API routes for database-backed resources.
- `app/db/dtos.py`: Pydantic request and response data models.
- `app/db/repos/`: Repository classes that run SQL queries.
- `app/db/service/db_table_service.py`: Service layer that combines repository results into API responses.
- `app/deps/db_pool.py`: Async PostgreSQL connection pool dependency.
- `sql/v1.sql`: Schema and seed data for students, courses, and enrollments.
- `docker-compose.yml`: Local PostgreSQL and API runtime.
- `Dockerfile.db` and `Dockerfile.server`: Container build files.
- `requirements.txt`: Python dependencies.

## Python venv

Create and activate a local development environment from the `lab05` directory:

```bash
cd lab05
python3 -m venv .venv/dev
source .venv/dev/bin/activate
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
```

If your shell prompt does not show the virtual environment, confirm that `which python` points inside `lab05/.venv/dev`.

## Usage Guide

**TROUBLESHOOTING**: (dev) docker compose up --build 
Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?

**SOLUTION**:
```
sudo systemctl start docker
```
-or-
```
open -a docker
```
```
docker info
```

The expected workflow for this lab is:

* **Milestone 1** - Launch a Database and the API with Docker Compose.
* **Milestone 2** - Create and Load the Database.
* **Milestone 3** - Make API Calls.
* **Milestone 4** - Explore a custom CLI client.

Run all commands from the `lab05` directory unless a step says otherwise.

## Milestone 1: Launch the Database and API using Docker Compose

### Docker Compose

Docker compose is a simple application that allows us to manage a simple server that runs multiple containers simultaneously.  Docker Compose is similar to a Dockerfile, but uses the YAML format for configuration.  We can bring an entire environment up or down with the simple `docker compose up` or `docker compose down` commands.  In this case, we can avoid starting and stoping the services separately.  In general, docker compose allows us to simplify maintanence on a single machine.  For more complicated, multi-server systems for test, stage or production, a container maintanence tool like Kubernates would scale better.  However, Docker Compose is a great option for development environments.

Open up [docker-compose.yaml](docker-compose.yaml).  You will notices that two containers are defined (`db: lab05-postgress` and `api: lab05-fastapi`):

```
services:
  db:
    image: postgres:16-alpine
    container_name: lab05-postgres
    environment:
        ...
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ...

  api:
    image: python:3.12-slim
    container_name: lab05-fastapi
    depends_on:
      db:
        condition: service_healthy
    environment:
        ...

```

Notice that we can define base images, enable ports, environment variables specific to containers, volumes (persistent storage locations), and other attributes similar to a `Dockerfile`.

When we call `docker compose up` for this application, we are loading up all the containers, specifically the `db` and the `api`.

### Launch the Postgress database and API

Start the database and API:

```bash
docker compose up --build
```

In a second terminal, verify both containers are running:

```bash
docker compose ps
```

The API should be available at:

```text
http://localhost:8000
```

Check the root health response:

```bash
curl http://localhost:8000/
```

Expected response:

```json
{"status":"ok"}
```

To stop the lab environment:

```bash
docker compose down
```

To remove the database volume and reset all data (Only do this command if you want to restart and remove all persistent database data):

```bash
docker compose down -v
```

### Configuring the Environment (If you would like to change the environment settings.)

The FastAPI app reads database settings from environment variables:

```text
DB_HOST
DB_PORT
DB_NAME
DB_USER
DB_PASSWORD
```

For local development outside Docker, create `.env` from the example values and use the PostgreSQL service exposed on port `5432`:

```bash
cp .env.example .env
```

Use these values if `.env.example` is empty:

```text
DB_HOST=localhost
DB_PORT=5432
DB_NAME=lab05
DB_USER=lab05
DB_PASSWORD=lab05
```

When running inside Docker Compose, the API container should use the database host name `db`:

```text
DB_HOST=db
DB_PORT=5432
DB_NAME=lab05
DB_USER=lab05
DB_PASSWORD=lab05
```

If you update `docker-compose.yml`, make sure the `api` service receives these variables and still depends on the healthy `db` service.

---

For **Milestone 1** you should be able to start and stop both the `db` and the `api` with `docker compose` and check the health of the api.  If the `api` is healthy, you have passed this milestone.

---

## Loading the Database

After Docker is running, load the schema and seed rows:

```bash
docker compose exec -T db psql -U lab05 -d lab05 < sql/v1.sql
```

Confirm the tables exist:

```bash
docker compose exec db psql -U lab05 -d lab05 -c "\dt"
```

Confirm seed data loaded:

```bash
docker compose exec db psql -U lab05 -d lab05 -c "SELECT id, first_name, last_name, major FROM students ORDER BY id;"
```

## Querying database

All API verification for this lab must be done through a custom Python CLI client that uses `httpx`. Do not rely on `curl`, browser clicks, or Swagger UI for your final evidence.

Create `client.py` in the `lab05` directory. It should support commands similar to:

```bash
python client.py students list
python client.py students create --first-name Mae --last-name Jemison --email mae.jemison@example.edu --major Engineering --credits 64
python client.py students update 6 --first-name Mae --last-name Jemison --email mae.jemison@example.edu --major Computer Science --credits 80 --active true
python client.py students delete 6
python client.py courses list
python client.py enrollments list
```

Minimum CLI requirements:

- Use `httpx.Client` or `httpx.AsyncClient`.
- Accept a configurable base URL with a default of `http://localhost:8000`.
- Print formatted JSON for every response.
- Exit with a non-zero status code when the API returns an error status.
- Include at least one command for each required CRUD operation.

## API Requirements

Implement or verify these endpoint groups:

```text
GET    /db/students
POST   /db/students
PUT    /db/students/{student_id}
DELETE /db/students/{student_id}

GET    /db/courses
POST   /db/courses
PUT    /db/courses/{course_id}
DELETE /db/courses/{course_id}

GET    /db/enrollments
POST   /db/enrollments
PUT    /db/enrollments/{enrollment_id}
DELETE /db/enrollments/{enrollment_id}
```

The starter code already includes DTOs, repository patterns, service methods, and several read/create/update routes. Add the missing delete path through the same route -> service -> repository structure. Keep SQL parameterized; do not build SQL by concatenating user input.

## CRUD: Read Query

Use the CLI client to read seeded data:

```bash
python client.py students list
python client.py courses list
python client.py enrollments list
```

Required evidence:

- The students response includes seeded students from `sql/v1.sql`.
- The courses response includes course codes such as `CSCI 3081`.
- The enrollments response includes nested student and course data.

## CRUD: Write Query -> Read Query

Create one new student:

```bash
python client.py students create --first-name Mae --last-name Jemison --email mae.jemison@example.edu --major Engineering --credits 64
```

Then read students again:

```bash
python client.py students list
```

Required evidence:

- The create response returns a new `id`.
- The follow-up read includes the new student.
- The database assigns `created_at`.

## CRUD: Update Query -> Read Query

Update the student you created:

```bash
python client.py students update 6 --first-name Mae --last-name Jemison --email mae.jemison@example.edu --major Computer Science --credits 80 --active true
```

Then read students again:

```bash
python client.py students list
```

Required evidence:

- The update response contains the changed fields.
- The follow-up read shows the same changed fields.
- Updating an unknown id returns `404`.

## CRUD: Delete Query -> Read Query

Delete the student you created:

```bash
python client.py students delete 6
```

Then read students again:

```bash
python client.py students list
```

Required evidence:

- The delete response identifies the deleted row or returns a success message.
- The follow-up read no longer includes that student.
- Deleting an unknown id returns `404`.

## Implementation Requirements

Your final submission must include:

- A working FastAPI app that starts without tracebacks.
- PostgreSQL schema and seed data loaded from `sql/v1.sql`.
- Read, create, update, and delete routes for at least students.
- Read routes for courses and enrollments.
- A custom `httpx` CLI client named `client.py`.
- Parameterized SQL in all repository methods.
- JSON responses for every API route.
- Read-after-write evidence for create, update, and delete.

Stretch goal: implement full CRUD for courses and enrollments using the same patterns as students.

## Suggested Checkoff Script

Before asking for checkoff, run:

```bash
docker compose down -v
docker compose up --build
docker compose exec -T db psql -U lab05 -d lab05 < sql/v1.sql
python client.py students list
python client.py students create --first-name Mae --last-name Jemison --email mae.jemison@example.edu --major Engineering --credits 64
python client.py students list
python client.py students update 6 --first-name Mae --last-name Jemison --email mae.jemison@example.edu --major Computer Science --credits 80 --active true
python client.py students list
python client.py students delete 6
python client.py students list
```

If those commands complete and the JSON output proves each database change, the lab satisfies the core requirements.

## Starter `client.py`

The repository includes an initial `client.py` that supports the core checkoff commands:

```bash
python client.py students list
python client.py students create --first-name Mae --last-name Jemison --email mae.jemison@example.edu --major Engineering --credits 64
python client.py students update 6 --first-name Mae --last-name Jemison --email mae.jemison@example.edu --major "Computer Science" --credits 80 --active true
python client.py students delete 6
python client.py courses list
python client.py enrollments list
```

Use `--base-url` before the resource name if your API is not running at `http://localhost:8000`.
