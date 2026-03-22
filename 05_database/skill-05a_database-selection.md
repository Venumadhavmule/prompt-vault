# SKILL 05-A — Database Selection: Choosing the Right Store for the Job
Category: Database
Applies to: PostgreSQL, MySQL, MongoDB, Redis, DynamoDB, Neo4j, InfluxDB, Elasticsearch

## What this skill covers
Database selection is one of the most consequential architectural decisions — it affects query patterns, scaling strategy, consistency guarantees, and operational complexity for the entire life of the project. This skill provides a decision framework for choosing between relational, document, key-value, graph, and time-series databases based on data shape, access patterns, consistency requirements, and team expertise.

## When to activate this skill
- Starting a new service or feature that needs data persistence
- Deciding whether to add a second database type alongside an existing one
- Evaluating a rewrite from one database technology to another
- Reviewing an architecture proposal involving database choices

## Core principles
1. **Default to PostgreSQL.** It handles relational, JSONB (document), full-text search, and time-series (with TimescaleDB) — use a specialized database only when PostgreSQL demonstrably cannot meet requirements.
2. **Data shape drives schema design, not the other way.** Fixed schema → relational. Variable schema with similar queries → document. Highly connected data → graph. Sequential write-heavy metrics → time-series.
3. **Access pattern is the primary driver.** Design the schema around how data is queried, not just how it is stored.
4. **Operational burden is a real cost.** Every additional database type adds infrastructure, monitoring, backup, and expertise requirements.
5. **NoSQL is not faster by default.** MongoDB, DynamoDB, and Redis excel in specific read patterns; they perform worse than PostgreSQL in many complex query scenarios.

## Step-by-step guide
**Decision tree:**

1. **Is the data highly relational with complex queries?** → PostgreSQL (or MySQL if team knows it)
2. **Is the primary pattern: simple key lookup at very high scale?** → DynamoDB or Redis (cache/session)
3. **Is the schema highly variable and documents are the query unit?** → MongoDB (but verify PostgreSQL JSONB doesn't meet needs first)
4. **Is the data a graph of relationships (social, recommendations)?** → Neo4j or Amazon Neptune
5. **Is the data time-sequential metrics with high write throughput?** → TimescaleDB (PostgreSQL extension) or InfluxDB
6. **Is the use case search / full-text ranking?** → PostgreSQL full-text search for basic needs; Elasticsearch/OpenSearch for advanced relevance scoring

**Polyglot persistence guidelines:**
- Use PostgreSQL as the primary store; add specialist databases only for a specific access pattern PostgreSQL cannot serve well
- Always define which database is the "source of truth" for any given entity
- Sync between databases via event/queue, not dual-write in application code

## The right way vs the wrong way

| Wrong | Right |
|-------|-------|
| Choose MongoDB to "avoid schema design" | Schema design is required in any database — data modeling happens in the application |
| Add Redis, Elasticsearch, and Mongo to a new project before proving the need | Start with PostgreSQL; add specialty stores when a specific perf/scale requirement is demonstrated |
| Use a document store for data with many cross-references | Highly relational data with join queries belongs in a relational database |
| Store all user session data in PostgreSQL at 100k req/s | Redis for session store; PostgreSQL for the underlying user data |
| One database for all concerns "to simplify" | Separate caching layer (Redis) from transactional store (PostgreSQL) for cache invalidation clarity |

## Stack-specific notes
**Node.js:** Prisma supports PostgreSQL, MySQL, SQLite, MongoDB. Use the Prisma-supported database that matches your data model. Drizzle for lighter-weight TypeScript-native ORM.
**Python:** SQLAlchemy 2 for relational databases (PostgreSQL, MySQL). Motor (async) or pymongo for MongoDB. `redis-py` for Redis.
**Go:** `database/sql` + `pgx` for PostgreSQL. `mongo-go-driver` for MongoDB. `go-redis` for Redis.

## Common mistakes
1. **Choosing MongoDB for a project because it "doesn't need a schema."** Most applications need data integrity — lack of schema leads to data inconsistency at scale.
2. **Using Redis as a primary database.** Redis is in-memory with optional persistence. It is a cache/session store; use a durable database as the source of truth.
3. **Adding Elasticsearch for any search need.** PostgreSQL `tsvector` handles most full-text search needs. Elasticsearch adds significant operational complexity.
4. **Designing for scale you don't have.** DynamoDB's access pattern limitations are painful — only adopt it when you have the actual read/write scale that justifies its constraints.

## Checklist
- [ ] Data shape analyzed (relational, document, key-value, graph, time-series)
- [ ] Primary access patterns documented before selecting database
- [ ] PostgreSQL evaluated first before specialty databases
- [ ] Source of truth defined for every entity in a polyglot architecture
- [ ] Operational cost of each database type assessed
- [ ] Team expertise with each candidate database factored in
- [ ] Consistency requirements (strong vs eventual) mapped to database capabilities
- [ ] Backup, replication, and failover requirements evaluated per database type
