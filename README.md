# News Management System API

A robust RESTful API for managing news articles with role-based access control, built with Express.js, TypeScript, MySQL, and Redis.

## Table of Contents

- [Project Overview](#project-overview)
- [Tech Stack](#tech-stack)
- [Features](#features)
- [Getting Started](#getting-started)
    - [Prerequisites](#prerequisites)
    - [Installation](#installation)
    - [Environment Variables](#environment-variables)
    - [Running with Docker](#running-with-docker)
    - [Running Locally](#running-locally)
- [Database Design](#database-design)
- [Redis Usage](#redis-usage)
- [API Endpoints](#api-endpoints)
- [API Documentation](#api-documentation)
- [Project Structure](#project-structure)
- [Author](#author)
- [License](#license)

## Project Overview

News Management System is a full-featured backend API that enables organizations to manage news articles, categories, and users with fine-grained role-based access control. The system supports three user roles (Admin, Editor, Viewer) with different permission levels, real-time analytics tracking, and high-performance caching.

### Key Capabilities

- **Content Management**: Create, update, publish/unpublish, and manage news articles
- **Category Organization**: Hierarchical category system with parent-child relationships
- **User Management**: Complete user lifecycle management with role assignments
- **Analytics Dashboard**: Track article views, device types, peak hours, and user contributions
- **Authentication**: JWT-based authentication with access and refresh tokens
- **Performance**: Redis caching for optimized read operations

## Tech Stack

| Technology       | Purpose               |
| ---------------- | --------------------- |
| **Node.js**      | Runtime environment   |
| **Express.js 5** | Web framework         |
| **TypeScript**   | Type-safe development |
| **MySQL 8.0**    | Primary database      |
| **Redis 7**      | Caching layer         |
| **JWT**          | Authentication        |
| **Zod**          | Request validation    |
| **Winston**      | Logging               |
| **Swagger UI**   | API documentation     |
| **Docker**       | Containerization      |
| **prom-client**  | Prometheus metrics    |

## Features

- **Role-Based Access Control (RBAC)**

    - Admin: Full system access
    - Editor: Content management
    - Viewer: Read-only access

- **Authentication & Security**

    - JWT access and refresh tokens
    - Password hashing with bcrypt
    - Rate limiting protection
    - CORS configuration

- **News Management**

    - CRUD operations
    - Featured articles
    - Publish/unpublish workflow
    - View tracking and analytics
    - Search by category, author, or slug

- **Category System**

    - Hierarchical categories (parent-child)
    - Slug-based URLs
    - Active/inactive status

- **Analytics**
    - Dashboard overview
    - View counts and trends
    - Device breakdown
    - Peak hours analysis
    - User contribution metrics

## Getting Started

### Prerequisites

- Node.js 18+ or Docker
- pnpm (recommended) or npm
- MySQL 8.0+
- Redis 7+

### Installation

1. **Clone the repository**

   ```bash
   git clone https://github.com/md-rejoyan-islam/news-api
   cd news-api
   ```

2. **Install dependencies**

   ```bash
   pnpm install
   ```

3. **Set up environment variables**

   ```bash
   cp .env.example .env
   # Edit .env with your configuration
   ```

4. **Run database migrations**

   ```bash
   pnpm migration
   ```

5. **Start the development server**
   ```bash
   pnpm dev
   ```

### Environment Variables

Create a `.env` file in the root directory:

```env
# Database
MYSQL_URI=mysql://root:password@localhost:3306/news_app
DATABASE_NAME=news_app
MYSQL_ROOT_PASSWORD=password

# Redis
REDIS_URL=redis://localhost:6379

# Server
SERVER_PORT=5000
NODE_ENV=development

# CORS
CLIENT_WHITELIST=http://localhost:3000,http://localhost:4000,http://localhost:5000

# Rate Limiting
MAX_REQUESTS=100
MAX_REQUESTS_WINDOW=900000

# JWT
JWT_SECRET_KEY=your-secret-key-here
JWT_ACCESS_TOKEN_EXPIRES_IN=18000
JWT_REFRESH_TOKEN_EXPIRES_IN=86400
```

### Running with Docker

The easiest way to run the complete stack using the provided scripts:

```bash
# Start all services (development mode with hot reload)
./scripts/start.sh

# Start with rebuild
./scripts/start.sh --build

# Start in attached mode (see logs directly)
./scripts/start.sh --attach

# Run database migrations
./scripts/migrate.sh

# View logs (all services)
./scripts/logs.sh

# View logs for specific service
./scripts/logs.sh --app      # or -a
./scripts/logs.sh --mysql    # or -m
./scripts/logs.sh --redis    # or -r

# Stop all services
./scripts/stop.sh

# Stop and remove volumes (reset data)
./scripts/stop.sh --volumes

# Stop, remove volumes and images (clean slate)
./scripts/stop.sh --clean
```

This starts:

- **App**: http://localhost:5000
- **phpMyAdmin**: http://localhost:8080
- **Redis Commander**: http://localhost:8081

> **Note**: Make scripts executable first: `chmod +x scripts/*.sh`

### Running Locally

For local development without Docker:

```bash
# Start MySQL and Redis separately, then:
pnpm dev
```

### Seeding Data

Populate the database with sample data:

```bash
# Via API (after starting the server)
curl -X POST http://localhost:5000/api/v1/seed

# Or reset and reseed
curl -X POST http://localhost:5000/api/v1/seed/reset
```

## Database Design

The system uses MySQL with three primary tables and a migration history table.

### Entity Relationship Diagram

```
┌──────────────────┐       ┌──────────────────┐       ┌──────────────────┐
│      users       │       │       news       │       │    categories    │
├──────────────────┤       ├──────────────────┤       ├──────────────────┤
│ id (PK)          │───┐   │ id (PK)          │   ┌───│ id (PK)          │
│ name             │   │   │ title            │   │   │ name             │
│ email (UNIQUE)   │   │   │ content          │   │   │ description      │
│ password         │   │   │ slug (UNIQUE)    │   │   │ slug (UNIQUE)    │
│ bio              │   └──>│ author_id (FK)   │   │   │ is_active        │
│ role             │       │ category_id (FK) │<──┘   │ is_deleted       │
│ status           │       │ is_featured      │       │ parent_id (FK)   │──┐
│ is_active        │       │ is_deleted       │       │ created_at       │  │
│ last_login       │       │ published_at     │       │ updated_at       │  │
│ created_at       │       │ meta (JSON)      │       └──────────────────┘  │
│ updated_at       │       │ created_at       │              ▲              │
│ deleted_at       │       │ updated_at       │              └──────────────┘
└──────────────────┘       └──────────────────┘           (self-reference)
```

### Tables

#### Users

| Column     | Type         | Description                     |
| ---------- | ------------ | ------------------------------- |
| id         | INT          | Primary key, auto-increment     |
| name       | VARCHAR(255) | User's full name                |
| email      | VARCHAR(255) | Unique email address            |
| password   | VARCHAR(255) | Bcrypt hashed password          |
| bio        | TEXT         | Optional user biography         |
| role       | ENUM         | 'admin', 'editor', 'viewer'     |
| status     | ENUM         | 'active', 'inactive', 'pending' |
| is_active  | BOOLEAN      | Account active status           |
| last_login | TIMESTAMP    | Last login timestamp            |
| deleted_at | TIMESTAMP    | Soft delete timestamp           |

#### Categories

| Column      | Type         | Description                       |
| ----------- | ------------ | --------------------------------- |
| id          | INT          | Primary key, auto-increment       |
| name        | VARCHAR(255) | Category name                     |
| description | TEXT         | Category description              |
| slug        | VARCHAR(255) | URL-friendly unique identifier    |
| is_active   | BOOLEAN      | Category visibility               |
| is_deleted  | BOOLEAN      | Soft delete flag                  |
| parent_id   | INT          | Self-referencing FK for hierarchy |

#### News

| Column       | Type         | Description                    |
| ------------ | ------------ | ------------------------------ |
| id           | INT          | Primary key, auto-increment    |
| title        | VARCHAR(255) | Article title                  |
| content      | TEXT         | Article body                   |
| slug         | VARCHAR(255) | URL-friendly unique identifier |
| author_id    | INT          | FK to users table              |
| category_id  | INT          | FK to categories table         |
| is_featured  | BOOLEAN      | Featured article flag          |
| is_deleted   | BOOLEAN      | Soft delete flag               |
| published_at | TIMESTAMP    | Publication date               |
| meta         | JSON         | Additional metadata            |

### Indexes

Optimized indexes for common queries:

- `idx_users_email` - Fast email lookups
- `idx_users_role` - Role-based filtering
- `idx_news_slug` - Slug-based article retrieval
- `idx_news_published_at` - Date-based queries
- `idx_categories_slug` - Category URL resolution

## Redis Usage

Redis serves as the caching layer to improve read performance and reduce database load.

### Caching Strategy

```
┌──────────────┐       Cache Miss       ┌──────────────┐
│    Client    │ ─────────────────────> │    MySQL     │
│    Request   │                        │   Database   │
└──────┬───────┘                        └──────┬───────┘
       │                                       │
       │ Cache Hit                             │ Query Result
       ▼                                       ▼
┌──────────────┐                        ┌──────────────┐
│    Redis     │ <───────────────────── │ Cache Write  │
│    Cache     │      Store Result      │              │
└──────────────┘                        └──────────────┘
```

### Cache Implementation

| Feature            | Description                             |
| ------------------ | --------------------------------------- |
| **Key Generation** | SHA-256 hash of resource + query params |
| **TTL**            | 30 days default (configurable)          |
| **Invalidation**   | Automatic on data mutations             |
| **Serialization**  | JSON encoding/decoding                  |

### Cached Resources

- News article listings
- Category listings
- Featured articles
- News by category/author
- Single article lookups

### Cache Functions

```typescript
// Set cache with TTL
setCache(key: string, data: T, ttlSeconds?: number): Promise<void>

// Get cached data
getCache<T>(key: string): Promise<T | null>

// Delete specific cache
deleteCache(key: string): Promise<void>

// Clear all cache
clearCache(): Promise<void>
```

## API Endpoints

### Authentication (`/api/v1/auth`)

| Method | Endpoint       | Description              | Access        |
| ------ | -------------- | ------------------------ | ------------- |
| POST   | `/register`    | Register new user        | Public        |
| POST   | `/login`       | User login               | Public        |
| POST   | `/refresh`     | Refresh access token     | Public        |
| POST   | `/logout`      | User logout              | Authenticated |
| GET    | `/me`          | Get current user profile | Authenticated |
| PUT    | `/me`          | Update profile           | Authenticated |
| PATCH  | `/me/password` | Change password          | Authenticated |

### Users (`/api/v1/users`)

| Method | Endpoint               | Description          | Access |
| ------ | ---------------------- | -------------------- | ------ |
| GET    | `/`                    | List all users       | Admin  |
| POST   | `/`                    | Create user          | Admin  |
| GET    | `/:id`                 | Get user by ID       | Admin  |
| PUT    | `/:id`                 | Update user          | Admin  |
| DELETE | `/:id`                 | Delete user (soft)   | Admin  |
| PATCH  | `/:id/status`          | Change user status   | Admin  |
| PATCH  | `/:id/change-password` | Change user password | Admin  |

### Categories (`/api/v1/categories`)

| Method | Endpoint        | Description            | Access |
| ------ | --------------- | ---------------------- | ------ |
| GET    | `/`             | List categories        | Public |
| POST   | `/`             | Create category        | Admin  |
| GET    | `/:id`          | Get category by ID     | Public |
| GET    | `/slug/:slug`   | Get category by slug   | Public |
| GET    | `/:id/children` | Get child categories   | Public |
| PUT    | `/:id`          | Update category        | Admin  |
| DELETE | `/:id`          | Delete category (soft) | Admin  |
| PATCH  | `/:id/status`   | Toggle category status | Admin  |

### News (`/api/v1/news`)

| Method | Endpoint                | Description           | Access        |
| ------ | ----------------------- | --------------------- | ------------- |
| GET    | `/`                     | List news articles    | Public        |
| POST   | `/`                     | Create article        | Admin, Editor |
| GET    | `/featured`             | Get featured articles | Public        |
| GET    | `/slug/:slug`           | Get article by slug   | Public        |
| GET    | `/category/:categoryId` | Get by category       | Public        |
| GET    | `/author/:authorId`     | Get by author         | Public        |
| GET    | `/:id`                  | Get article by ID     | Public        |
| PUT    | `/:id`                  | Update article        | Admin, Editor |
| DELETE | `/:id`                  | Delete article (soft) | Admin         |
| PATCH  | `/:id/featured`         | Toggle featured       | Admin, Editor |
| PATCH  | `/:id/publish`          | Publish article       | Admin, Editor |
| PATCH  | `/:id/unpublish`        | Unpublish article     | Admin, Editor |
| POST   | `/:id/view`             | Track article view    | Public        |

### Analytics (`/api/v1/analytics`)

| Method | Endpoint            | Description           | Access        |
| ------ | ------------------- | --------------------- | ------------- |
| GET    | `/overview`         | Dashboard overview    | Admin, Editor |
| GET    | `/views`            | Total view statistics | Admin, Editor |
| GET    | `/views/top`        | Most viewed articles  | Admin, Editor |
| GET    | `/views/devices`    | Views by device type  | Admin, Editor |
| GET    | `/views/peak-hours` | Peak viewing hours    | Admin, Editor |
| GET    | `/users/news`       | User post analytics   | Admin, Editor |

### Seed (`/api/v1/seed`)

| Method | Endpoint      | Description          | Access   |
| ------ | ------------- | -------------------- | -------- |
| POST   | `/`           | Seed all data        | Public\* |
| POST   | `/users`      | Seed users only      | Public\* |
| POST   | `/categories` | Seed categories only | Public\* |
| POST   | `/news`       | Seed news only       | Public\* |
| DELETE | `/`           | Clear all data       | Public\* |
| POST   | `/reset`      | Reset and reseed     | Public\* |

\*Note: Seed routes should be protected in production

### System Routes

| Method | Endpoint   | Description              |
| ------ | ---------- | ------------------------ |
| GET    | `/`        | API welcome message      |
| GET    | `/health`  | Health check             |
| GET    | `/metrics` | Prometheus metrics       |
| GET    | `/docs`    | Swagger UI documentation |

## API Documentation

Interactive API documentation is available via Swagger UI:

```
http://localhost:5000/docs
```

The documentation includes:

- All endpoint descriptions
- Request/response schemas
- Example payloads
- Authentication requirements
- Try-it-out functionality

## Project Structure

```
news-management-system/
├── src/
│   ├── app/
│   │   └── types.ts              # Global type definitions
│   ├── config/
│   │   ├── db.ts                 # MySQL connection
│   │   ├── redis.ts              # Redis connection
│   │   └── secret.ts             # Environment config
│   ├── middlewares/
│   │   ├── authorized.ts         # Role authorization
│   │   ├── error-handler.ts      # Global error handler
│   │   ├── matrics-middleware.ts # Prometheus metrics
│   │   ├── validate.ts           # Zod validation
│   │   └── verify.ts             # JWT verification
│   ├── modules/
│   │   ├── analytics/            # Analytics module
│   │   ├── auth/                 # Authentication module
│   │   ├── category/             # Category module
│   │   ├── news/                 # News module
│   │   ├── seed/                 # Database seeding
│   │   └── users/                # User management
│   ├── routes/
│   │   └── index.ts              # Route aggregation
│   ├── utils/
│   │   ├── async-handler.ts      # Async error wrapper
│   │   ├── cache.ts              # Redis cache utilities
│   │   ├── logger.ts             # Winston logger
│   │   └── response-handler.ts   # Response formatting
│   ├── migration.ts              # Database migrations
│   └── server.ts                 # Application entry
├── migrations/
│   └── 001_init.sql              # Initial schema
├── data/
│   ├── users.json                # Seed data
│   ├── categories.json
│   └── news.json
├── docs/
│   └── openapi.yaml              # API specification
├── docker-compose.dev.yml        # Development stack
├── Dockerfile.dev                # Development image
└── package.json
```
