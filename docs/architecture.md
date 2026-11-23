# Olex — Architecture & Engineering Decisions

## 1. Architectural Approach

Olex combines a Django REST Framework backend with a Next.js storefront.

The application separates business operations, presentation logic, asynchronous workloads, and infrastructure concerns.

### Main Responsibilities

| Component | Responsibility |
|---|---|
| Next.js | Storefront, server-side rendering, BFF authentication and API routes |
| Django REST Framework | Business operations and versioned REST API |
| PostgreSQL | Transactional data and persistent state |
| Redis | Caching and Celery message transport |
| Celery | Background task processing |
| Celery Beat | Scheduled task execution |
| nginx | Public HTTPS entry point and reverse proxy |

## 2. Transactional Checkout

Order creation involves multiple dependent operations: validating the cart, creating the order, updating inventory, and clearing the cart.

These operations must preserve database consistency.

The checkout service uses `transaction.atomic()` to group the database changes into a transaction.

Cart item queries use PostgreSQL row-level locking. Inventory changes also use conditional updates to prevent stock from being reduced below the available quantity.

This design addresses concurrency scenarios in which multiple requests attempt to purchase overlapping inventory.

### Why PostgreSQL-Specific Tests?

SQLite does not reproduce PostgreSQL `SELECT FOR UPDATE` behavior.

A dedicated PostgreSQL test suite validates locking behavior and catches database-specific errors that would not appear in the default SQLite test environment.

## 3. Business Logic Separation

The project uses service layers for key domain operations, including:

- Cart management
- Checkout and order lifecycle
- Payment creation and reconciliation
- Refund operations

This reduces the amount of business logic in API views and helps reuse the same rules in different entry points, including administrative workflows.

## 4. Authentication Boundary

In production, browser clients communicate with Next.js and its same-origin API routes.

The BFF manages authentication cookies and communicates with the Django backend over private networking.

This reduces the need to expose Django Bearer authentication endpoints directly to browser clients.

State-changing BFF routes implement same-origin checks, and the backend maintains its own authentication and permission controls.

The deployed storefront is currently protected by HTTP Basic Authentication for visitor access. This access restriction is separate from application-level JWT authentication managed by the Next.js BFF and Django backend.

## 5. Payment Integration

A dedicated YooKassa gateway adapter isolates payment-provider SDK interactions from application business logic.

The payment layer supports payment status handling, webhooks, refunds, and idempotency mechanisms.

Payment-provider activation is controlled through deployment configuration.

## 6. Asynchronous Processing

Celery workers process background workloads independently from HTTP request handling.

Redis provides the message transport and caching capabilities.

Celery Beat supports scheduled execution.

This separation helps keep HTTP endpoints focused on user-facing operations while moving suitable workloads into background processing.

## 7. CI/CD and Verification

The GitHub Actions CI pipeline includes backend tests, code-quality checks, PostgreSQL-specific tests, frontend verification, and Docker builds.

Production deployment is managed separately and requires manual triggering.

A successful CI run is a prerequisite for the deployment workflow.

## 8. Operational Considerations

The deployment architecture includes:

- Environment-specific Django settings
- Production configuration validation
- nginx reverse proxy
- Docker container orchestration
- Monitoring and application logs
- Backup and restoration procedures

These mechanisms support deployment and maintenance, but do not eliminate the need for security reviews, dependency updates, or operational monitoring.

## 9. Scope of This Project Overview

This document describes architectural decisions reflected in the private project repository.

It intentionally omits production credentials, infrastructure addresses, internal configuration values, private source code, and operational data.
