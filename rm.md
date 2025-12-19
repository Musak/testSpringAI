# LO AI Sandbox Application

A Spring Boot application demonstrating Retrieval-Augmented Generation (RAG) using Spring AI, PostgreSQL with pgvector extension, and Model Context Protocol (MCP) integration for travel services.

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Features](#features)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
  - [Docker Compose (Recommended)](#docker-compose-recommended)
  - [Local Development](#local-development)
- [Configuration](#configuration)
- [API Endpoints](#api-endpoints)
  - [RAG Endpoints](#rag-endpoints)
  - [Travel Assistant](#travel-assistant)
  - [Travel MCP Service](#travel-mcp-service)
- [MCP Integration](#mcp-integration)
- [UI Frontend](#ui-frontend)
- [Logging and Observability](#logging-and-observability)
- [Troubleshooting](#troubleshooting)

## Overview

This application provides:

- **RAG (Retrieval-Augmented Generation)**: Store documents in a vector database and query them using semantic search
- **Travel Assistant**: AI-powered travel planning with MCP tool integration
- **Travel MCP Service**: Microservice exposing hotel, flight, and booking operations via MCP protocol
- **Streamlit UI**: Web interface for document management and assistant queries

## Architecture

The application consists of three main components:

1. **Spring Boot Main Application** (`ailoc-spring`)
   - Port: `8080`
   - RAG functionality with pgvector
   - Travel assistant with MCP client
   - Document upload and query endpoints

2. **Travel MCP Service** (`travel-mcp-service`)
   - Port: `8081`
   - MCP server exposing travel tools
   - REST API for hotels, flights, and bookings
   - Shared PostgreSQL database

3. **Streamlit UI** (`ui`)
   - Port: `8501`
   - Web interface for document management
   - Assistant query interface

All services share a PostgreSQL database with pgvector extension for vector storage.

## Features

- **Document Management**: Upload text documents or files, store embeddings in pgvector
- **Semantic Search**: Query documents using similarity search with configurable thresholds
- **Travel Planning**: AI assistant that can search hotels, flights, and create bookings via MCP tools
- **MCP Protocol**: Model Context Protocol integration for tool-based AI interactions
- **Observability**: OpenTelemetry tracing, Logstash integration, request ID tracking
- **Conversation Memory**: Multi-turn conversations with conversation ID support

## Prerequisites

- **Java 24** (with `--enable-preview` flag)
- **Maven 3.9+**
- **Docker** or **Podman** with Compose support
- **PostgreSQL** with pgvector extension (or use Docker image)
- **API Key** for Azure OpenAI (or compatible endpoint)

### Environment Variables

Required environment variables:

- `YOUR_API_KEY`: Azure OpenAI API key (required)
- `AZURE_ENDPOINT`: Azure OpenAI endpoint (default: `https://ai-proxy.lab.epam.com`)
- `CHAT_MODEL`: Chat model name (default: `gpt-4o-mini-2024-07-18`)
- `EMBEDDING_MODEL`: Embedding model (default: `text-embedding-005`)
- `API_VERSION`: API version (default: `2023-07-01-preview`)

Optional:

- `MCP_SERVER_URL`: Travel MCP service URL (default: `http://travel-mcp-service:8081`)
- `OTEL_EXPORTER_OTLP_ENDPOINT`: OpenTelemetry endpoint for Langfuse
- `OTEL_EXPORTER_OTLP_HEADERS`: OpenTelemetry authentication headers
- `LOGSTASH_HOST`: Logstash host (default: `localhost`)
- `LOGSTASH_PORT`: Logstash port (default: `5100`)

## Installation

### Docker Compose (Recommended)

The easiest way to run the entire stack is using Docker Compose.

#### Prerequisites

1. Create the external ELK network (if using ELK/Logstash):

```bash
# Docker
docker network create elk

# Podman
podman network create elk
```

2. Set required environment variables:

```bash
export YOUR_API_KEY="your-api-key-here"
export AZURE_ENDPOINT="https://ai-proxy.lab.epam.com"  # Optional, has default
export CHAT_MODEL="gpt-4o-mini-2024-07-18"  # Optional, has default
```

#### Start Services

Start all services (Spring app, Postgres, Travel MCP service, and UI):

```bash
docker compose --profile local-spring --profile local-pg --profile local-ui up -d
```

Start only backend services (without UI):

```bash
docker compose --profile local-spring --profile local-pg up -d
```

#### Stop Services

```bash
docker compose down                      # Preserves volumes (keeps pgvector data)
docker compose down --volumes            # Removes volumes (⚠️ deletes pgvector data)
```

#### View Logs

```bash
docker compose logs -f spring-boot-app      # Spring app logs
docker compose logs -f travel-mcp-service   # Travel MCP service logs
docker compose logs -f postgres-pgvector    # PostgreSQL logs
docker compose logs -f frontend             # UI logs
```

#### Services Overview

- **spring-boot-app**: Main application on port `8080`
- **travel-mcp-service**: MCP service on port `8081`
- **postgres-pgvector**: PostgreSQL with pgvector on port `6432`
- **frontend**: Streamlit UI on port `8501`

### Local Development

#### 1. Database Setup

Start PostgreSQL with pgvector:

```bash
docker run -d \
  --name postgres-pgvector \
  -e POSTGRES_USER=raguser \
  -e POSTGRES_PASSWORD=ragpassword \
  -e POSTGRES_DB=rag_database_spring \
  -p 6432:5432 \
  -v pgvector_data:/var/lib/postgresql/data \
  pgvector/pgvector:0.8.0-pg17
```

Initialize the database schema (run once):

```sql
CREATE EXTENSION IF NOT EXISTS vector;
CREATE EXTENSION IF NOT EXISTS hstore;
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

CREATE TABLE IF NOT EXISTS vector_store (
  id uuid DEFAULT uuid_generate_v4() PRIMARY KEY,
  content text,
  metadata jsonb,
  embedding vector(768)
);

CREATE INDEX IF NOT EXISTS vector_store_embedding_idx 
  ON vector_store USING HNSW (embedding vector_cosine_ops);
```

#### 2. Build Main Application

```bash
./mvnw clean package -DskipTests
```

#### 3. Run Main Application

```bash
export YOUR_API_KEY="your-api-key"
export SPRING_DATASOURCE_URL="jdbc:postgresql://localhost:6432/rag_database_spring"
java -jar target/ailoc-0.0.1-SNAPSHOT.jar
```

#### 4. Build and Run Travel MCP Service

```bash
cd travel-mcp-service
../mvnw clean package -DskipTests
export SPRING_DATASOURCE_URL="jdbc:postgresql://localhost:6432/rag_database_spring"
java -jar target/travel-mcp-service-*.jar
```

#### 5. Run UI (Optional)

```bash
cd ui
pip install -r requirements.txt
export API_URL="http://localhost:8080"
streamlit run streamlit_client.py
```

## Configuration

### Application Configuration

Main application configuration is in `src/main/resources/application.yml`:

- **Vector Store**: pgvector configuration (table name, schema, dimensions)
- **AI Models**: OpenAI/Azure endpoint configuration
- **MCP Client**: Travel MCP service connection
- **Similarity Search**: Top-K, threshold, retry settings

### Database Configuration

The application uses PostgreSQL with pgvector. Key settings:

- **Database**: `rag_database_spring`
- **User**: `raguser`
- **Password**: `ragpassword`
- **Port**: `6432` (Docker) or `5432` (local)
- **Vector Dimensions**: `768` (for text-embedding-005)

## API Endpoints

### RAG Endpoints

#### Add Text Document

Add a text document to the vector store:

```bash
curl -X POST http://localhost:8080/add-text \
  -H "Content-Type: application/json" \
  -d '{
    "content": "The capital of France is Paris. It is known for the Eiffel Tower.",
    "metadata": {
      "source": "manual",
      "document_id": "doc-001",
      "category": "geography"
    }
  }'
```

#### Add File Document

Upload a file to the vector store:

```bash
curl -F "file=@/path/to/document.txt" \
  http://localhost:8080/add-document
```

#### Query Assistant (RAG)

Query the assistant using RAG:

```bash
curl -X POST http://localhost:8080/ask \
  -H "Content-Type: application/json" \
  -d '{
    "query": "What is the capital of France?"
  }'
```

### Travel Assistant

The travel assistant uses MCP tools to search hotels, flights, and create bookings.

#### Basic Travel Query

```bash
curl -X POST http://localhost:8080/travel \
  -H "Content-Type: application/json" \
  -d '{
    "query": "I want to travel from Frankfurt to Budapest on 2025-12-25. I will travel alone. Find me a hotel and flight.",
    "conversationId": "user-123"
  }'
```

#### Multi-Step Travel Planning

**Step 1: Initial Query**

```bash
curl -X POST http://localhost:8080/travel \
  -H "Content-Type: application/json" \
  -d '{
    "query": "Hello, I am currently in Frankfurt and want to travel to Budapest. Taxi to airport is 100 euro, from Frankfurt airport to Budapest is 200 euro. What is the total amount I need to spend? What hotel and flight will I use?",
    "conversationId": "user-123"
  }'
```

**Step 2: Provide Travel Details**

```bash
curl -X POST http://localhost:8080/travel \
  -H "Content-Type: application/json" \
  -d '{
    "query": "Departure date is 2025-12-25 from Frankfurt. I will travel alone. Any hotel is good for me.",
    "conversationId": "user-123"
  }'
```

**Step 3: Complete Booking**

```bash
curl -X POST http://localhost:8080/travel \
  -H "Content-Type: application/json" \
  -d '{
    "query": "Proceed with booking the hotel. The rest I will manage.",
    "conversationId": "user-123"
  }'
```

### Travel MCP Service

Direct REST API endpoints for travel services (also exposed via MCP):

#### Search Hotels

```bash
curl -X GET "http://localhost:8081/travel/hotels?city=Budapest&minRating=4.0" \
  -H "Accept: application/json"
```

Parameters:
- `city` (optional): Filter by city
- `country` (optional): Filter by country
- `minRating` (optional): Minimum rating (e.g., `4.0`)

#### Search Flights

```bash
curl -X GET "http://localhost:8081/travel/flights?departure=Frankfurt&arrival=Budapest&departureDate=2025-12-25" \
  -H "Accept: application/json"
```

Parameters:
- `departure` (optional): Departure city
- `arrival` (optional): Arrival city
- `departureDate` (optional): Departure date (format: `YYYY-MM-DD`)

#### Create Booking

```bash
curl -X POST http://localhost:8081/travel/bookings \
  -H "Content-Type: application/json" \
  -d '{
    "hotelId": 1,
    "flightId": 1,
    "travelerName": "Ada Lovelace",
    "travelerEmail": "ada@example.com",
    "tripName": "Business Trip",
    "checkInDate": "2025-02-01",
    "checkOutDate": "2025-02-05",
    "status": "NEW"
  }'
```

## MCP Integration

The application uses Model Context Protocol (MCP) to expose travel services as tools to the AI assistant.

### MCP Server (Travel MCP Service)

The travel MCP service exposes three tools:

1. **`searchHotels`**: Search hotels by city, country, and minimum rating
2. **`searchFlights`**: Search flights by departure, arrival, and date
3. **`createBooking`**: Create a booking linking hotel and/or flight

MCP endpoint: `http://travel-mcp-service:8081/mcp-v1`

### MCP Client (Main Application)

The main application connects to the travel MCP service as a client:

- **Type**: `SYNC`
- **Protocol**: `STATELESS`
- **Endpoint**: Configured via `MCP_SERVER_URL` environment variable

### MCP Inspector

To inspect and test MCP tools, use the MCP Inspector:

```bash
npx @modelcontextprotocol/inspector
```

Configure it to connect to: `http://localhost:8081/mcp-v1`

## UI Frontend

The Streamlit UI provides a web interface for:

- **Ask Assistant**: Query the RAG assistant
- **Upload Document**: Add text or file documents to the vector store

Access the UI at: `http://localhost:8501`

Features:
- User authentication (alice@epam.com, bob@epam.com)
- Text document upload with metadata
- File upload
- Assistant query interface

## Logging and Observability

### Request ID Tracking

All requests include an `X-Request-ID` header and MDC context for tracing:

```bash
curl -X POST http://localhost:8080/ask \
  -H "Content-Type: application/json" \
  -H "X-Request-ID: my-request-123" \
  -d '{"query": "test"}'
```

### Logstash Integration

Logs are sent to Logstash for ELK stack integration:

- Configure `LOGSTASH_HOST` and `LOGSTASH_PORT` environment variables
- Logback configuration in `logback-spring.xml`
- JSON formatted logs with request ID

### OpenTelemetry Tracing

Tracing is configured for Langfuse integration:

1. Get credentials from Langfuse UI (Settings > API Keys)
2. Generate base64 auth header:
   ```bash
   echo -n 'pk-lf-xxx:sk-lf-xxx' | base64
   ```
3. Set environment variable:
   ```bash
   export OTEL_EXPORTER_OTLP_HEADERS="Authorization=Basic <generated-base64>"
   ```

### HTTP Client Logging

Enable detailed HTTP logging for debugging:

```yaml
logging:
  level:
    http-client-outgoing: DEBUG
    webclient-outgoing: DEBUG
    org.apache.http.wire: DEBUG
    reactor.netty.http.client: DEBUG
```

⚠️ **Warning**: Wire logging can expose sensitive data. Use only for debugging.

## Troubleshooting

### Common Issues

#### 1. Proxying Final Method / GenericFilterBean.init NPE

**Error**: `Error creating bean with name 'requestIdFilter'`

**Solution**: Set in `application.yml`:
```yaml
spring:
  aop:
    proxy-target-class: false
```

#### 2. BadSqlGrammarException on INSERT for pgvector

**Error**: SQL errors when inserting into `vector_store`

**Solutions**:
- Verify table schema matches configuration (dimensions, column types)
- Ensure `metadata` column is `jsonb` type
- Check PostgreSQL JDBC driver version compatibility
- Verify `table-name` and `schema-name` in `application.yml`

#### 3. MCP Connection Errors

**Error**: Cannot connect to MCP server

**Solutions**:
- Verify `MCP_SERVER_URL` environment variable is set correctly
- Check that travel-mcp-service is running on port `8081`
- Ensure both services are on the same Docker network
- Check network connectivity: `curl http://travel-mcp-service:8081/travel/hotels`

#### 4. Vector Store Not Initialized

**Error**: Vector store operations fail

**Solutions**:
- Run database initialization SQL (see [Local Development](#local-development))
- Check `spring.ai.vectorstore.pgvector.initialize-schema` setting
- Verify pgvector extension is installed: `SELECT * FROM pg_extension WHERE extname = 'vector';`

#### 5. API Key Not Found

**Error**: `API key is required`

**Solution**: Set `YOUR_API_KEY` environment variable:
```bash
export YOUR_API_KEY="your-api-key-here"
```

### Database Backup and Restore

**Backup pgvector data**:
```bash
docker exec postgres-pgvector pg_dump -U raguser rag_database_spring > backup.sql
```

**Restore**:
```bash
docker exec -i postgres-pgvector psql -U raguser rag_database_spring < backup.sql
```

### Preserving Data

⚠️ **Important**: To preserve pgvector data:

- Use `docker compose down` (not `--volumes`)
- Use named volumes (configured in `docker-compose.yaml`)
- Avoid `docker compose down --volumes` unless you want to delete all data

### Network Issues

**Check container connectivity**:
```bash
docker compose ps
docker network inspect ailoc-spring_app-network
docker network inspect elk
```

**Test service endpoints**:
```bash
# From host
curl http://localhost:8080/actuator/health
curl http://localhost:8081/travel/hotels

# From container
docker compose exec spring-boot-app curl http://travel-mcp-service:8081/travel/hotels
```

---

## Additional Resources

- [Spring AI Documentation](https://docs.spring.io/spring-ai/reference/)
- [Model Context Protocol](https://modelcontextprotocol.io/)
- [pgvector Documentation](https://github.com/pgvector/pgvector)
- [Streamlit Documentation](https://docs.streamlit.io/)
