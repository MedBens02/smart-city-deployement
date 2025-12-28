# Smart City Application - Docker Setup

## About This Setup

This application uses **pre-built Docker images** from Docker Hub. Images are automatically built and pushed by GitHub Actions whenever code is updated. This ensures:
- ✅ Fast startup (pull images instead of building from source)
- ✅ Works on all platforms (Windows, Mac, Linux)
- ✅ Latest code automatically available (every git push triggers new builds)

## Prerequisites

- Docker Desktop installed
- 8GB RAM minimum
- Ports available: 3000, 3001, 8081, 3306, 9092, 9000

## Setup Steps

### 1. Create environment file

```bash
copy .env.example .env
```

### 2. Edit `.env` and add runtime secrets

Open `.env` and replace the placeholder values:

```bash
# Runtime secrets (provided by developer)
CLERK_SECRET_KEY=<provided_by_developer>
R2_ACCOUNT_ID=<provided_by_developer>
R2_ACCESS_KEY_ID=<provided_by_developer>
R2_SECRET_ACCESS_KEY=<provided_by_developer>
R2_BUCKET_NAME=<provided_by_developer>
R2_PUBLIC_URL=<provided_by_developer>
```

**Note:** Build-time secrets (NEXT_PUBLIC_*) are already baked into the Docker images!

### 3. Pull latest images from Docker Hub

```bash
docker-compose pull
```

This downloads all 3 service images (takes ~2 minutes).

### 4. Start all services

```bash
docker-compose up -d
```

### 5. Wait for services to be healthy (~2 minutes)

```bash
docker-compose ps
```

**What happens during startup:**
1. MySQL and Kafka start first
2. Kafka-Init container creates all 16 Kafka topics and exits (you'll see it as "Exited (0)")
3. Backend, Frontend, and Electricity services start

All services should show "healthy" or "Up" status. Example output:

```
NAME                   STATUS
smartcity-backend      Up (healthy)
smartcity-electricity  Up
smartcity-frontend     Up
smartcity-kafka        Up (healthy)
smartcity-kafka-init   Exited (0)  # This is normal - it ran and completed
smartcity-mysql        Up (healthy)
```

### 6. Access the application

- **Frontend:** http://localhost:3000
- **Backend API:** http://localhost:8081
- **Backend Health:** http://localhost:8081/actuator/health
- **Electricity Service:** http://localhost:3001
- **Kafdrop (Kafka UI):** http://localhost:9000

### 7. Monitoring Kafka with Kafdrop

Kafdrop provides a web UI for monitoring Kafka:
- **View all 16 topics:** claims.SPK, claims.RFM, claims.TRM, claims.ENV, claims.WM, etc.
- **Inspect messages:** Browse messages in each topic in real-time
- **Monitor consumers:** See consumer groups and lag
- **Broker info:** View broker configuration and health

Access Kafdrop at http://localhost:9000

## Getting Latest Updates

When the developer pushes code changes to GitHub:

```bash
docker-compose pull   # Pull latest images from Docker Hub
docker-compose up -d  # Restart with new images
```

**No rebuild needed** - just pull and restart!

## Troubleshooting

### Port Conflicts

If you get port binding errors, ensure these ports are not in use:
```bash
# Windows
netstat -ano | findstr "3000 3001 8081 3306 9092 9000"

# Linux/Mac
lsof -i :3000 -i :3001 -i :8081 -i :3306 -i :9092 -i :9000
```

### Services Not Starting

**Check logs:**
```bash
docker-compose logs -f backend
docker-compose logs -f frontend
docker-compose logs -f mysql
docker-compose logs -f kafka
```

**Common issues:**
- **MySQL not ready:** Wait 30 seconds for MySQL to initialize on first run
- **Kafka not ready:** Kafka takes ~20 seconds to start in KRaft mode
- **kafka-init failed:** Check logs with `docker-compose logs kafka-init` - topics may not have been created
- **Backend fails:** Check that MySQL and Kafka are healthy AND kafka-init has completed successfully

### Full Reset

To completely reset the environment (removes all data):

```bash
docker-compose down -v  # Stops containers and removes volumes
docker-compose up -d    # Starts fresh
```

### Verify Kafka Topics Were Created

To check that all 16 topics were created successfully:

```bash
docker exec smartcity-kafka /opt/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --list
```

You should see all 16 topics listed with the `claims.` prefix.

### Restart a Single Service

```bash
docker-compose restart backend   # or frontend, electricity-service, etc.
```

## Architecture

```
MySQL (3306) → Kafka (9092) → Kafka-Init (creates topics) → Backend (8081) → Frontend (3000) + Electricity (3001)
```

**Startup Sequence:**
1. MySQL starts and becomes healthy
2. Kafka starts and becomes healthy
3. Kafka-Init creates all 16 topics and exits
4. Backend starts (waits for topics to be ready)
5. Frontend and Electricity services start

All services run in an isolated Docker network with persistent data volumes:
- **smartcity-mysql-data:** Stores database data
- **smartcity-kafka-data:** Stores Kafka logs and topics

## Technical Details

### Kafka (KRaft Mode)

This application uses modern **KRaft Kafka** without Zookeeper:
- **Cluster ID:** MkU3OEVBNTcwNTJENDM2Qk
- **Topics:** Automatically created by kafka-init container on startup
- **Topic Count:** 16 topics total (13 service-specific + 3 system topics)
- **Replication:** Single-node setup (replication factor = 1)

**Topics Created:**
- Service topics: SPK, RFM, TRM, ENV, WM, PAT, PRP, STR, WEM, GDD, MTU, AGD, AEP
- System topics: responses, status-updates, dead-letter
- All topics prefixed with `claims.` (e.g., `claims.SPK`)

### Database

- **Engine:** MySQL 8.0
- **Database:** smartcity_claims
- **Schema:** Auto-created by Spring Boot on startup
- **Credentials:** Root with no password (development only)

### Health Checks

All services have health checks:
- **MySQL:** `mysqladmin ping`
- **Kafka:** `kafka-cluster.sh cluster-id`
- **Backend:** `/actuator/health` endpoint
- **Frontend/Electricity:** HTTP GET to `/`

## Stopping the Application

```bash
# Stop all services (keeps data)
docker-compose down

# Stop and remove all data
docker-compose down -v
```

## Support

If you encounter issues:
1. Check the logs using `docker-compose logs -f <service-name>`
2. Verify all ports are available
3. Ensure Docker Desktop has at least 8GB RAM allocated
4. Contact the developer for API key verification

---

Built with Docker Compose | KRaft Kafka | Spring Boot | Next.js
