# BrightPath Dockerized Development Environment

## Overview

This document outlines the Dockerized development environment for the BrightPath homeschooling scheduling platform. Using Docker ensures consistency across development environments, simplifies onboarding for new team members, and closely mirrors the production environment to reduce deployment issues.

## Benefits of Dockerized Development

1. **Consistency**: Ensures all developers work in identical environments, eliminating "works on my machine" issues
2. **Quick Onboarding**: New team members can get started with a single command
3. **Isolation**: Dependencies and services are isolated, preventing conflicts with other projects
4. **Production Parity**: Development environment closely resembles production, reducing deployment surprises
5. **Service Integration**: Easy integration of services like PostgreSQL, Valkey, and ML components

## Docker Components

### 1. Docker Compose Configuration

The primary configuration for the development environment is defined in `docker-compose.yml`:

```yaml
version: '3.8'

services:
  # Frontend service
  frontend:
    build:
      context: ./client
      dockerfile: Dockerfile.dev
    volumes:
      - ./client:/app
      - /app/node_modules
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=development
      - VITE_API_URL=http://localhost:8000
    depends_on:
      - api

  # Backend API service
  api:
    build:
      context: ./api
      dockerfile: Dockerfile.dev
    volumes:
      - ./api:/app
    ports:
      - "8000:8000"
    environment:
      - DJANGO_SETTINGS_MODULE=brightpath.settings.development
      - DATABASE_URL=postgres://postgres:postgres@db:5432/brightpath
      - VALKEY_HOST=valkey
      - VALKEY_PORT=6379
      - GOOGLE_OAUTH_CLIENT_ID=${GOOGLE_OAUTH_CLIENT_ID}
      - GOOGLE_OAUTH_CLIENT_SECRET=${GOOGLE_OAUTH_CLIENT_SECRET}
    depends_on:
      - db
      - valkey

  # Worker service for background tasks
  worker:
    build:
      context: ./api
      dockerfile: Dockerfile.dev
    command: python -m tasks.worker
    volumes:
      - ./api:/app
    environment:
      - DJANGO_SETTINGS_MODULE=brightpath.settings.development
      - DATABASE_URL=postgres://postgres:postgres@db:5432/brightpath
      - VALKEY_HOST=valkey
      - VALKEY_PORT=6379
    depends_on:
      - db
      - valkey

  # ML service
  ml:
    build:
      context: ./services/ml
      dockerfile: Dockerfile.dev
    volumes:
      - ./services/ml:/app
    ports:
      - "8001:8001"
    environment:
      - VALKEY_HOST=valkey
      - VALKEY_PORT=6379
    depends_on:
      - valkey

  # Database service
  db:
    image: postgres:14
    volumes:
      - postgres_data:/var/lib/postgresql/data
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
      - POSTGRES_DB=brightpath
    ports:
      - "5432:5432"

  # Valkey service (message bus)
  valkey:
    image: valkey/valkey:latest
    ports:
      - "6379:6379"
    volumes:
      - valkey_data:/data

volumes:
  postgres_data:
  valkey_data:
```

### 2. Dockerfiles

#### Frontend Dockerfile (client/Dockerfile.dev)

```dockerfile
FROM node:16-alpine

WORKDIR /app

# Copy package.json and package-lock.json
COPY package*.json ./

# Install dependencies
RUN npm install

# Copy the rest of the code
COPY . .

# Expose port
EXPOSE 3000

# Start development server
CMD ["npm", "run", "dev", "--", "--host", "0.0.0.0"]
```

#### Backend Dockerfile (api/Dockerfile.dev)

```dockerfile
FROM python:3.12-slim

# Set environment variables
ENV PYTHONDONTWRITEBYTECODE 1
ENV PYTHONUNBUFFERED 1

# Set work directory
WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    libpq-dev \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Install pipenv
RUN pip install --no-cache-dir pipenv

# Copy Pipfile and Pipfile.lock
COPY Pipfile Pipfile.lock ./

# Install dependencies
RUN pipenv install --dev --system --deploy

# Copy the rest of the code
COPY . .

# Expose port
EXPOSE 8000

# Start development server
CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
```

#### ML Service Dockerfile (services/ml/Dockerfile.dev)

```dockerfile
FROM python:3.12-slim

# Set environment variables
ENV PYTHONDONTWRITEBYTECODE 1
ENV PYTHONUNBUFFERED 1

# Set work directory
WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Install pipenv
RUN pip install --no-cache-dir pipenv

# Copy Pipfile and Pipfile.lock
COPY Pipfile Pipfile.lock ./

# Install dependencies
RUN pipenv install --dev --system --deploy

# Copy the rest of the code
COPY . .

# Expose port
EXPOSE 8001

# Start development server
CMD ["uvicorn", "api.main:app", "--host", "0.0.0.0", "--port", "8001", "--reload"]
```

## Development Workflow

### Initial Setup

1. **Clone the repository**

```bash
git clone https://github.com/your-organization/brightpath.git
cd brightpath
```

2. **Create .env file**

Create a `.env` file in the project root with the following variables:

```
GOOGLE_OAUTH_CLIENT_ID=your-client-id
GOOGLE_OAUTH_CLIENT_SECRET=your-client-secret
```

3. **Start the development environment**

```bash
docker-compose up
```

This command will:
- Build all the necessary Docker images
- Create and start the containers
- Set up the network between containers
- Mount the volumes for code and data
- Forward the necessary ports

4. **Access the services**

- Frontend: http://localhost:3000
- Backend API: http://localhost:8000
- ML Service: http://localhost:8001

### Daily Development

1. **Start the environment**

```bash
docker-compose up -d
```

2. **View logs**

```bash
# All services
docker-compose logs -f

# Specific service
docker-compose logs -f api
```

3. **Run commands in containers**

```bash
# Run Django migrations
docker-compose exec api python manage.py migrate

# Create a superuser
docker-compose exec api python manage.py createsuperuser

# Run tests
docker-compose exec api pytest

# Install a new npm package
docker-compose exec frontend npm install some-package
```

4. **Stop the environment**

```bash
docker-compose down
```

### Database Management

1. **Connect to the database**

```bash
# Connect to the database using psql
docker-compose exec db psql -U postgres -d brightpath
```

2. **Reset the database**

```bash
# Stop the containers
docker-compose down

# Remove the volumes
docker volume rm brightpath_postgres_data

# Start the containers again
docker-compose up -d
```

## Advanced Configuration

### Hot Reloading

The development environment is configured for hot reloading:

- **Frontend**: Vite's built-in hot module replacement (HMR) automatically refreshes the browser when code changes
- **Backend**: Django's development server automatically reloads when Python files change
- **ML Service**: Uvicorn with `--reload` flag watches for file changes

### Debugging

#### Backend Debugging

1. **Add the debugger configuration to `docker-compose.yml`**

```yaml
api:
  # ... other configuration
  command: python -m debugpy --listen 0.0.0.0:5678 manage.py runserver 0.0.0.0:8000
  ports:
    - "8000:8000"
    - "5678:5678"
```

2. **Configure VS Code**

Add this to `.vscode/launch.json`:

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Python: Remote Attach",
      "type": "python",
      "request": "attach",
      "connect": {
        "host": "localhost",
        "port": 5678
      },
      "pathMappings": [
        {
          "localRoot": "${workspaceFolder}/api",
          "remoteRoot": "/app"
        }
      ]
    }
  ]
}
```

#### Frontend Debugging

Use the browser's developer tools for debugging the frontend code.

### Adding New Services

To add a new service to the development environment:

1. **Create a Dockerfile** for the service
2. **Add the service** to `docker-compose.yml`
3. **Configure networking** between the new service and existing services
4. **Update documentation** to include the new service

## Common Issues and Solutions

### Container Won't Start

**Issue**: A container fails to start or exits immediately.

**Solution**:
1. Check the logs: `docker-compose logs <service_name>`
2. Verify the Dockerfile and command
3. Ensure all dependencies are installed
4. Check for port conflicts

### Volume Mounting Issues

**Issue**: Code changes aren't reflected in the container.

**Solution**:
1. Verify volume mounts in `docker-compose.yml`
2. Restart the container: `docker-compose restart <service_name>`
3. Rebuild the container: `docker-compose up -d --build <service_name>`

### Database Connection Issues

**Issue**: Services can't connect to the database.

**Solution**:
1. Ensure the database container is running: `docker-compose ps`
2. Check database credentials in environment variables
3. Verify network connectivity between containers
4. Wait for the database to fully initialize

## Best Practices

1. **Use Volume Mounts** for code to enable hot reloading
2. **Keep Dependencies Updated** in Dockerfiles
3. **Use Environment Variables** for configuration
4. **Document Changes** to the Docker configuration
5. **Optimize Docker Images** for faster builds
6. **Use Docker Compose Profiles** for different development scenarios

## Onboarding New Developers

New team members can get started with the following steps:

1. **Install Prerequisites**:
   - Docker and Docker Compose
   - Git
   - VS Code or preferred IDE

2. **Clone the Repository**:
   ```bash
   git clone https://github.com/your-organization/brightpath.git
   cd brightpath
   ```

3. **Set Up Environment Variables**:
   - Copy `.env.example` to `.env`
   - Fill in the required values

4. **Start the Development Environment**:
   ```bash
   docker-compose up
   ```

5. **Access the Application**:
   - Frontend: http://localhost:3000
   - API Documentation: http://localhost:8000/api/docs/
   - ML Service: http://localhost:8001

## Conclusion

The Dockerized development environment for BrightPath ensures consistency, simplifies onboarding, and provides a development experience that closely mirrors production. By following the workflows and best practices outlined in this document, developers can focus on building features rather than configuring their environment.

This approach supports the revised technology stack, including JavaScript with React, Python 3.12 with Django, Google OAuth, Radix UI, and Valkey, while providing a consistent and reliable development experience for all team members.