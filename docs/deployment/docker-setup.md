# Docker Setup Guide

## Prerequisites
- Docker Desktop installed
- 8GB RAM minimum
- 20GB disk space

## Services in Docker Compose

### Infrastructure Services
```yaml
# PostgreSQL (5 databases)
postgres:
  image: postgres:15
  environment:
    - POSTGRES_USER=admin
    - POSTGRES_PASSWORD=password
  ports:
    - "5432:5432"
  volumes:
    - postgres_data:/var/lib/postgresql/data
    - ./scripts/init-postgres.sql:/docker-entrypoint-initdb.d/init.sql

# ClickHouse
clickhouse:
  image: clickhouse/clickhouse-server:latest
  ports:
    - "8123:8123"
    - "9000:9000"

# Redis
redis:
  image: redis:7-alpine
  ports:
    - "6379:6379"

# RabbitMQ
rabbitmq:
  image: rabbitmq:3.12-management
  ports:
    - "5672:5672"    # AMQP
    - "15672:15672"  # Management UI
  environment:
    - RABBITMQ_DEFAULT_USER=admin
    - RABBITMQ_DEFAULT_PASS=password

# Airflow for ETL / Analytics Orchestration
  airflow:
    image: apache/airflow:2.7.0
    environment:
      - AIRFLOW__CORE__LOAD_EXAMPLES=False
      - AIRFLOW__CORE__EXECUTOR=LocalExecutor
      - AIRFLOW__CORE__SQL_ALCHEMY_CONN=postgresql+psycopg2://admin:password@postgres:5432/airflow
    volumes:
      - airflow_logs:/opt/airflow/logs
    ports:
      - "8080:8080"
    command: >
      bash -c "airflow db init && airflow webserver & airflow scheduler"
    depends_on:
      - postgres
```

### Application Services
```yaml
order-service:
    image: sample/order-service:latest
    ports:
      - "5001:5001"
    environment:
      - REDIS_HOST=redis
      - RABBITMQ_HOST=rabbitmq
      - POSTGRES_HOST=postgres
    depends_on:
      - postgres
      - redis
      - rabbitmq

  payment-service:
    image: sample/payment-service:latest
    ports:
      - "5085:5085"
    environment:
      - RABBITMQ_HOST=rabbitmq
      - POSTGRES_HOST=postgres
    depends_on:
      - postgres
      - rabbitmq

  product-service:
    image: sample/product-service:latest
    ports:
      - "5062:5062"
    environment:
      - POSTGRES_HOST=postgres
    depends_on:
      - postgres

  inventory-service:
    image: sample/inventory-service:latest
    ports:
      - "5002:5002"
    environment:
      - RABBITMQ_HOST=rabbitmq
      - POSTGRES_HOST=postgres
    depends_on:
      - postgres
      - rabbitmq

  qr-service:
    image: sample/qr-service:latest
    ports:
      - "5003:5003"
    environment:
      - POSTGRES_HOST=postgres
      - RABBITMQ_HOST=rabbitmq
    depends_on:
      - postgres
      - rabbitmq

  analytics-service:
    image: sample/analytics-service:latest
    ports:
      - "8000:8000"
    environment:
      - CLICKHOUSE_HOST=clickhouse
      - POSTGRES_HOST=postgres
    depends_on:
      - postgres
      - clickhouse
```

## Quick Start
```bash
# Clone repository
git clone https://github.com/alno09/FnB-automation-system.git
cd fb-automation-system

# Start all services
docker-compose up -d

# Check status
docker-compose ps

# View logs
docker-compose logs -f order-service

# Stop all
docker-compose down
```

## Access Points
- Order Service: http://localhost:5001
- Payment Service: http://localhost:5002
- Product Service: http://localhost:5003
- Inventory Service: http://localhost:5004
- QR Service: http://localhost:5005
- Analytics Service: http://localhost:8000
- RabbitMQ Management: http://localhost:15672
- Airflow Web UI: http://localhost:8080