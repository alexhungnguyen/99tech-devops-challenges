# Problem 4 - Investigation Report

## Problems Found

### 1. Nginx proxy port mismatch (Critical)

The Nginx reverse proxy forwards `/api/` traffic to `http://api:3001`, but the Express API listens on port **3000**. Every request to `/api/` returns a **502 Bad Gateway**. This is the **primary reason** the API is inaccessible.

- **File:** `nginx/conf.d/default.conf` line 9
- **Misconfiguration:** `proxy_pass http://api:3001;`
- **Expected:** `proxy_pass http://api:3000;`

### 2. Database connection leak (High)

In `api/src/index.js`, `db.release()` is called inside the `try` block. If `db.query()` or `redis.set()` throws, the acquired connection is never released back to the pool. Over time this exhausts the connection pool (default size 10), causing all subsequent requests to hang indefinitely waiting for a free connection.

### 3. No restart policy (Medium)

None of the services in `docker-compose.yml` define a `restart` policy. If the API process crashes due to an unhandled exception, the container stays down permanently until manually restarted.

### 4. No readiness-based dependency (Low)

> **Note:** This is not a direct cause of the persistent API inaccessibility - it is a startup race condition that may cause transient failures when containers first come up. Once postgres and redis are ready, this issue has no further effect.

`depends_on` in docker-compose only ensures containers **start** in order - it does not wait for postgres or redis to be **ready to accept connections**. If the API boots faster than postgres, initial connection attempts will fail, potentially crashing the process with no retry mechanism.

### 5. Postgres init script not mounted (Low)

> **Note:** This does not cause the API inaccessibility. The init script is never executed, but the default postgres configuration (100 max connections) works without issue.

`postgres/init.sql` exists locally but is never volume-mounted into the container at `/docker-entrypoint-initdb.d/`. The intended `ALTER SYSTEM SET max_connections = 20;` is never executed.

---

## How I Diagnosed Them

1. **Compared port numbers** between `nginx/conf.d/default.conf` (`proxy_pass` target port 3001) and `api/src/index.js` (`app.listen(3000)`). Immediate mismatch.
2. **Traced the error-handling flow** in the `/api/users` handler - `db.release()` is not in a `finally` block, so any exception after `pool.connect()` leaks the connection.
3. **Reviewed `docker-compose.yml`** for operational best practices: no `restart` policies, no health checks, no volume mount for `init.sql`.
4. **Checked `nginx/nginx.conf`** - file is empty and not mounted, confirming only `conf.d/` is used (harmless but misleading).

---

## Fixes Applied

### Fix 1: Correct the Nginx proxy port

```diff
# nginx/conf.d/default.conf
- proxy_pass http://api:3001;
+ proxy_pass http://api:3000;
```

### Fix 2: Prevent connection leaks with `finally`

```diff
# api/src/index.js
  app.get("/api/users", async (req, res) => {
+   let db;
    try {
-     const db = await pool.connect();
+     db = await pool.connect();
      const result = await db.query("SELECT NOW()");
-     db.release();
      await redis.set("last_call", Date.now());
      res.json({ ok: true, time: result.rows[0] });
    } catch (err) {
      console.error(err);
      res.status(500).json({ ok: false, error: err.message });
+   } finally {
+     if (db) db.release();
    }
  });
```

### Fix 3: Add restart policies and health checks

```diff
# docker-compose.yml
  api:
    build: ./api
+   restart: unless-stopped
+   healthcheck:
+     test: ["CMD", "wget", "-qO-", "http://localhost:3000/status"]
+     interval: 5s
+     timeout: 3s
+     retries: 5
    depends_on:
-     - postgres
-     - redis
+     postgres:
+       condition: service_healthy
+     redis:
+       condition: service_healthy
    ...

  nginx:
    image: nginx:1.25
    depends_on:
-     - api
+     api:
+       condition: service_healthy
    ...

  postgres:
    image: postgres:15
+   restart: unless-stopped
+   volumes:
+     - ./postgres/init.sql:/docker-entrypoint-initdb.d/init.sql
+   healthcheck:
+     test: ["CMD-SHELL", "pg_isready -U postgres"]
+     interval: 5s
+     timeout: 3s
+     retries: 5
    ...

  redis:
    image: redis:7
+   restart: unless-stopped
+   healthcheck:
+     test: ["CMD", "redis-cli", "ping"]
+     interval: 5s
+     timeout: 3s
+     retries: 5
```

### Result

After applying all fixes, rebuild and restart the stack:

```bash
docker compose up --build -d
```

**All containers are running and healthy:**

```bash
$ docker ps
CONTAINER ID   IMAGE          STATUS                   PORTS                    NAMES
5ecfc5f06674   nginx:1.25     Up 7 minutes             0.0.0.0:8080->80/tcp     problem4-nginx-1
32f24ceeb203   problem4-api   Up 7 minutes (healthy)                            problem4-api-1
918258f94b8b   postgres:15    Up 7 minutes (healthy)   5432/tcp                 problem4-postgres-1
d1d66bf3b773   redis:7        Up 7 minutes (healthy)   6379/tcp                 problem4-redis-1
```

**API is now accessible and returning data:**

```bash
$ curl localhost:8080/api/users
{"ok":true,"time":{"now":"2026-04-09T17:25:00.310Z"}}
```

---

## Monitoring / Alerts

- **Uptime / HTTP checks:** Monitor the `/status` endpoint with an external prober. Alert if it returns non-200 or response time exceeds a threshold. This can be done via an external 3rd party tool (e.g: openstatus - https://www.openstatus.dev) or self-hosted uptime monitor tool (e.g: Uptime Kuma - https://github.com/louislam/uptime-kuma)
- **Nginx error rate:** Parse Nginx access logs for 5xx responses. Alert if the 5xx rate exceeds 1% over a 5-minute window.
- **Connection pool metrics:** Expose `pool.waitingCount` and `pool.totalCount` from the `pg` pool as Prometheus metrics. Alert when waiting connections approach pool size. For example:
```js
app.get("/metrics/pool", (req, res) => {
  res.json({
    total: pool.totalCount,
    idle: pool.idleCount,
    waiting: pool.waitingCount,
  });
});
```
- **Container health:** Use Docker health checks (as added above) and alert on `unhealthy` state via a container monitoring tool.
- **Resource utilization:** Track CPU, memory, and disk per container. Alert on sustained high usage or OOM kills.

---

## How I Would Prevent This in Production

1. **Code reviews:** Require PR reviews. A port mismatch is easily caught in review.
2. **Integration tests in CI:** Run `docker compose up` in CI and hit the `/api/users` endpoint. A 502 would fail the pipeline before merge.
3. **Health check gates:** Always define health checks on dependencies and use `condition: service_healthy` so services don't start before their backends are ready.
4. **Restart policy:** Default to `restart: unless-stopped` for all services.
5. **Observability:** Deploy structured logging, metrics collection, and alerting alongside the application.
