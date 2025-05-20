# E-commerce Microservices Architecture Documentation

## Overview

This document outlines the architecture of an e-commerce platform built using a microservices approach. The platform is designed to be scalable, modular, and maintainable, leveraging **Node.js with TypeScript**, **MongoDB** (separate database per service), **RabbitMQ** for event-driven communication, **REST APIs** for inter-service and client communication, and **Elasticsearch** for search functionality. The system consists of **10 microservices** and an **API Gateway**, each handling a specific domain of the e-commerce platform.

### Tech Stack

- **Backend**: Node.js with TypeScript, Express.js for REST APIs.
- **Databases**: MongoDB (separate database per service), Elasticsearch for Search Service.
- **Messaging**: RabbitMQ for event-driven communication.
- **API Gateway**: Node.js with Express or a dedicated gateway (e.g., Kong).
- **Deployment**: Docker for containerization, Kubernetes for orchestration.
- **Security**: JWT for authentication, HTTPS for APIs, MongoDB role-based access.

### Architecture Principles

- **Database per Service**: Each microservice has its own MongoDB database to ensure loose coupling and independent scalability.
- **Event-Driven Communication**: RabbitMQ enables asynchronous events (e.g., order creation triggers inventory updates).
- **REST APIs**: Services expose REST endpoints for synchronous communication.
- **Single Entry Point**: The API Gateway routes client requests to appropriate services and handles authentication.

## Microservices Overview

The platform comprises 10 microservices and 1 API Gateway, each with a specific responsibility. Below is a detailed description of each component.

### 1. API Gateway

- **Purpose**: Acts as the single entry point for all client requests, routing them to appropriate microservices and handling cross-cutting concerns like authentication and rate limiting.
- **Functionalities**:
  - Route client requests to microservices.
  - Authenticate requests using JWT tokens (verified via User Service).
  - Rate limiting to prevent abuse.
  - Logging and monitoring of requests.
- **Database**: None (stateless).
- **APIs**:
  - Proxy endpoints for all services (e.g., `/api/users/*`, `/api/products/*`, `/api/search/*`).
  - Example: `GET /api/products` routes to Product Catalog Service.
- **Events**: None (does not publish or consume events).
- **Interactions**:
  - Receives client requests (web, mobile, etc.).
  - Forwards requests to services via REST (e.g., `http://product-service/api/products`).
  - Validates JWT tokens by calling User Service (`/api/users/validate-token`).
- **Tech Details**:
  - Framework: Node.js + Express + TypeScript or Kong.
  - Libraries: `express-gateway` or `http-proxy-middleware` for routing.
  - Port: 3000.
- **Deployment**: Containerized with Docker, scaled with Kubernetes for high traffic.
- **Security**: HTTPS, JWT validation, rate limiting (e.g., using `express-rate-limit`).

### 2. User Service

- **Purpose**: Manages user accounts, authentication, and authorization.
- **Functionalities**:
  - User registration, login, password reset (JWT-based authentication).
  - Profile management (name, address, preferences).
  - Role-based access control (customer, admin).
- **Database**: MongoDB (`user_db`, collections: `users`, `sessions`).
  - `users`: Stores user details (email, hashed password, name, address, role).
  - `sessions`: Stores active JWT sessions (optional, with TTL).
- **APIs**:
  - `POST /api/users/register`: Create a new user.
  - `POST /api/users/login`: Authenticate user, return JWT.
  - `GET /api/users/profile`: Retrieve user profile (requires JWT).
  - `PUT /api/users/profile`: Update user profile.
  - `POST /api/users/validate-token`: Validate JWT for API Gateway.
- **Events**:
  - Publishes: `user.created` (user ID, email) to RabbitMQ for notifications.
- **Interactions**:
  - Called by API Gateway for authentication.
  - Publishes events to Notification Service.
- **Tech Details**:
  - Framework: Node.js + Express + TypeScript.
  - Libraries: `jsonwebtoken` for JWT, `bcrypt` for password hashing, `mongoose` for MongoDB.
  - Port: 3001.
- **Deployment**: Containerized with Docker, scaled with Kubernetes for high traffic, MongoDB with role-based access.

### 3. Product Catalog Service

- **Purpose**: Manages product listings and categories.
- **Functionalities**:
  - CRUD operations for products (title, description, price, images).
  - Category management (create, update, list categories).
- **Database**: MongoDB (`product_db`, collections: `products`, `categories`).
  - `products`: Stores product details (ID, title, description, price, categories, images).
  - `categories`: Stores category hierarchy (ID, name, parentId).
- **APIs**:
  - `GET /api/products`: List products with pagination.
  - `GET /api/products/{id}`: Get product details.
  - `POST /api/products`: Create product (admin only).
  - `PUT /api/products/{id}`: Update product.
  - `DELETE /api/products/{id}`: Delete product.
  - `GET /api/categories`: List categories.
- **Events**:
  - Publishes: `product.created`, `product.updated`, `product.deleted` (product ID, details) to RabbitMQ for Search Service.
- **Interactions**:
  - Called by Search Service for product details.
  - Publishes events to Search Service for indexing.
- **Tech Details**:
  - Framework: Node.js + Express + TypeScript.
  - Libraries: `mongoose` for MongoDB.
  - Port: 3002.
- **Deployment**: Containerized with Docker, scaled with Kubernetes for high traffic, MongoDB with indexing on `products.title`.

### 4. Inventory Service

- **Purpose**: Tracks product stock levels.
- **Functionalities**:
  - Monitor and update stock quantities.
  - Reserve stock during checkout.
  - Emit low-stock alerts.
- **Database**: MongoDB (`inventory_db`, collection: `inventory`).
  - `inventory`: Stores product ID, stock quantity, reserved quantity.
- **APIs**:
  - `GET /api/inventory/{productId}`: Get stock for a product.
  - `POST /api/inventory/reserve`: Reserve stock for an order.
  - `PUT /api/inventory/{productId}`: Update stock (admin or Order Service).
- **Events**:
  - Consumes: `order.created` (product IDs, quantities) to reserve stock.
  - Publishes: `inventory.low` (product ID, stock level) for notifications.
  - Publishes: `inventory.updated` for Search Service (optional).
- **Interactions**:
  - Called by Order Service for stock reservation.
  - Publishes events to Notification and Search Services.
- **Tech Details**:
  - Framework: Node.js + Express + TypeScript.
  - Libraries: `mongoose`, `amqplib`.
  - Port: 3003.
- **Deployment**: Containerized with Docker, scaled with Kubernetes for high traffic, MongoDB with indexes on `productId`.

### 5. Order Service

- **Purpose**: Manages order creation and lifecycle.
- **Functionalities**:
  - Create and track orders.
  - Update order status (pending, confirmed, shipped, delivered).
  - Coordinate with Inventory, Payment, and Shipping Services.
- **Database**: MongoDB (`order_db`, collection: `orders`).
  - `orders`: Stores order ID, user ID, products, status, total amount, shipping details.
- **APIs**:
  - `POST /api/orders`: Create an order.
  - `GET /api/orders/{orderId}`: Get order details.
  - `PUT /api/orders/{orderId}/status`: Update order status.
- **Events**:
  - Publishes: `order.created` (order ID, user ID, products) to RabbitMQ for inventory, shipping, and notifications.
- **Interactions**:
  - Calls Inventory Service to reserve stock.
  - Calls Payment Service to process payment.
  - Calls Shipping Service for shipping options.
  - Publishes events to Notification and Search Services.
- **Tech Details**:
  - Framework: Node.js + Express + TypeScript.
  - Libraries: `mongoose`, `amqplib`, `axios` for REST calls.
  - Port: 3004.
- **Deployment**: Containerized with Docker, scaled with Kubernetes for high traffic, MongoDB with indexes on `orderId`, `userId`.

### 6. Payment Service

- **Purpose**: Handles payment processing and transactions.
- **Functionalities**:
  - Process payments via gateways (e.g., Stripe).
  - Manage refunds and transaction records.
- **Database**: MongoDB (`payment_db`, collection: `transactions`).
  - `transactions`: Stores transaction ID, order ID, amount, status, payment method.
- **APIs**:
  - `POST /api/payments`: Process payment for an order.
  - `POST /api/payments/refund`: Process refund.
  - `GET /api/payments/{orderId}`: Get transaction details.
- **Events**:
  - Publishes: `payment.success`, `payment.failed` (order ID, status) to RabbitMQ for order updates and notifications.
- **Interactions**:
  - Called by Order Service for payment processing.
  - Publishes events to Notification Service.
- **Tech Details**:
  - Framework: Node.js + Express + TypeScript.
  - Libraries: `mongoose`, `amqplib`, `stripe` for payment integration.
  - Port: 3005.
- **Deployment**: Containerized with Docker, scaled with Kubernetes for high traffic, MongoDB with encrypted fields (e.g., payment details).

### 7. Cart Service

- **Purpose**: Manages user shopping carts.
- **Functionalities**:
  - Add/remove items to/from cart.
  - Calculate totals (including taxes, discounts).
  - Persist carts for logged-in users or sessions.
- **Database**: MongoDB (`cart_db`, collection: `carts`).
  - `carts`: Stores user ID (or session ID), products, quantities, total.
- **APIs**:
  - `GET /api/cart`: Get user’s cart.
  - `POST /api/cart/add`: Add item to cart.
  - `DELETE /api/cart/{productId}`: Remove item from cart.
  - `PUT /api/cart`: Update cart quantities.
- **Events**: None (cart updates are synchronous via REST).
- **Interactions**:
  - Called by Order Service to retrieve cart during checkout.
  - Queries Product Catalog Service for product details (e.g., price).
- **Tech Details**:
  - Framework: Node.js + Express + TypeScript.
  - Libraries: `mongoose`, `axios`.
  - Port: 3006.
- **Deployment**: Containerized with Docker, scaled with Kubernetes for high traffic, MongoDB with TTL indexes for session carts.

### 8. Shipping Service

- **Purpose**: Manages shipping options and tracking.
- **Functionalities**:
  - Calculate shipping costs based on location and weight.
  - Integrate with shipping providers (e.g., FedEx API).
  - Track shipments.
- **Database**: MongoDB (`shipping_db`, collection: `shipments`).
  - `shipments`: Stores order ID, shipping provider, cost, tracking number, status.
- **APIs**:
  - `POST /api/shipping/quote`: Get shipping cost for an order.
  - `GET /api/shipping/track/{orderId}`: Track shipment.
- **Events**:
  - Consumes: `order.created` to generate shipping options.
  - Publishes: `shipping.updated` (order ID, tracking number) for notifications.
- **Interactions**:
  - Called by Order Service for shipping quotes.
  - Publishes events to Notification Service.
- **Tech Details**:
  - Framework: Node.js + Express + TypeScript.
  - Libraries: `mongoose`, `amqplib`, `axios`.
  - Port: 3007.
- **Deployment**: Containerized with Docker, scaled with Kubernetes for high traffic, MongoDB with indexes on `orderId`.

### 9. Review Service

- **Purpose**: Manages customer reviews and ratings for products.
- **Functionalities**:
  - Submit, edit, delete reviews.
  - Calculate average product ratings.
  - Moderate reviews (admin functionality).
- **Database**: MongoDB (`review_db`, collection: `reviews`).
  - `reviews`: Stores product ID, user ID, rating, comment, createdAt.
- **APIs**:
  - `POST /api/reviews`: Submit a review.
  - `GET /api/reviews/product/{productId}`: Get reviews for a product.
  - `PUT /api/reviews/{reviewId}`: Update review (user or admin).
  - `DELETE /api/reviews/{reviewId}`: Delete review (admin only).
- **Events**:
  - Publishes: `review.created`, `review.updated`, `review.deleted` (product ID, rating) to RabbitMQ for Search Service and notifications.
- **Interactions**:
  - Called by Search Service for review data.
  - Publishes events to Search and Notification Services.
- **Tech Details**:
  - Framework: Node.js + Express + TypeScript.
  - Libraries: `mongoose`, `amqplib`.
  - Port: 3008.
- **Deployment**: Containerized with Docker, scaled with Kubernetes for high traffic, MongoDB with indexes on `productId`.

### 10. Search Service

- **Purpose**: Provides advanced search and filtering for products using Elasticsearch.
- **Functionalities**:
  - Full-text search across products (title, description, keywords).
  - Faceted search (by category, price, rating, stock).
- **Database**: Elasticsearch (`products_index`).

  - Index schema:

    ```json
    {
      "productId": "string",
      "title": "text",
      "description": "text",
      "categories": ["keyword"],
      "price": "float",
      "averageRating": "float",
      "stock": "integer",
      "keywords": ["text"]
    }
    ```

- **APIs**:
  - `GET /api/search?q={query}`: Full-text search.
  - `GET /api/search/filters?category={category}&minPrice={price}&minRating={rating}`: Faceted search.
- **Events**:
  - Consumes: `product.created`, `product.updated`, `product.deleted`, `review.created`, `review.updated` from RabbitMQ to update index.
- **Interactions**:
  - Fetches data from Product Catalog (`/api/products/{id}`), Review (`/api/reviews/product/{id}`), and Inventory (`/api/inventory/{id}`) Services via REST.
  - Updates Elasticsearch index with denormalized data.
- **Tech Details**:
  - Framework: Node.js + Express + TypeScript.
  - Libraries: `@elastic/elasticsearch`, `amqplib`, `axios`.
  - Port: 3009.
- **Deployment**: Containerized with Docker, scaled with Kubernetes for high traffic.

### 11. Notification Service

- **Purpose**: Sends notifications to users (email, SMS, push).
- **Functionalities**:
  - Send order confirmations, shipping updates, review notifications.
  - Log notification history.
- **Database**: MongoDB (`notification_db`, collection: `notifications`).
  - `notifications`: Stores user ID, type (email/SMS), content, status, createdAt.
- **APIs**:
  - `POST /api/notifications/send`: Manually trigger a notification (optional).
- **Events**:
  - Consumes: `order.created`, `payment.success`, `shipping.updated`, `review.created` from RabbitMQ to send notifications.
- **Interactions**:
  - Uses `nodemailer` for email, Twilio for SMS (optional).
- **Tech Details**:
  - Framework: Node.js + Express + TypeScript.
  - Libraries: `mongoose`, `amqplib`, `nodemailer`, `twilio` (optional).
  - Port: 3010.
- **Deployment**: Containerized with Docker, scaled with Kubernetes for high traffic, MongoDB with indexes on `userId`.

## System Workflow Example (Order Placement)

1. **Client**: Sends `POST /api/orders` to API Gateway with JWT.
2. **API Gateway**: Validates JWT via User Service, routes request to Order Service.
3. **Order Service**:
   - Retrieves cart from Cart Service (`GET /api/cart`).
   - Reserves stock via Inventory Service (`POST /api/inventory/reserve`).
   - Processes payment via Payment Service (`POST /api/payments`).
   - Creates order in `order_db`, publishes `order.created` to RabbitMQ.
4. **Inventory Service**: Consumes `order.created`, updates stock in `inventory_db`.
5. **Shipping Service**: Consumes `order.created`, generates shipping options in `shipping_db`, publishes `shipping.updated`.
6. **Notification Service**: Consumes `order.created`, `payment.success`, `shipping.updated`, sends email/SMS.
7. **Search Service**: Consumes `product.updated` or `review.created` (if related), updates Elasticsearch index.
8. **Client**: Searches products via `GET /api/search?q=laptop`, routed to Search Service, which queries Elasticsearch.

## Infrastructure Setup

- **Docker**: Each microservice runs in a separate Docker container.

  - Example `Dockerfile` for a service:

    ```dockerfile
    FROM node:18
    WORKDIR /app
    COPY package*.json ./
    RUN npm install
    COPY . .
    CMD ["npm", "run", "start"]
    ```

- **MongoDB**: 9 separate databases (`user_db`, `product_db`, etc.) using Docker or MongoDB Atlas.

  - Example Docker Compose for `product_db`:

    ```yaml
    services:
      product_db:
        image: mongo:latest
        ports:
          - "27018:27017"
        volumes:
          - product_data:/data/db
    volumes:
      product_data:
    ```

- **Elasticsearch**: Single instance for Search Service (`docker run -p 9200:9200 elasticsearch:8.8.0`).
- **RabbitMQ**: Single instance with durable queues (`docker run -p 5672:5672 rabbitmq:3-management`).
- **Kubernetes**: Orchestrate containers, scale services, and manage MongoDB/Elasticsearch clusters.
- **Monitoring**: Prometheus for metrics, Grafana for visualization, Winston for logging.

## Security Considerations

- **Authentication**: JWT tokens issued by User Service, validated by API Gateway.
- **Authorization**: Role-based access (e.g., admin-only endpoints in Product Catalog Service).
- **Data Encryption**: Encrypt sensitive MongoDB fields (e.g., user passwords, payment details) using `bcrypt` or MongoDB encryption.
- **API Security**: HTTPS for all REST APIs, API keys for service-to-service communication.
- **Database Security**: Separate MongoDB credentials per service, role-based access control.
- **Elasticsearch Security**: Enable authentication (e.g., API keys) for production.

## Scalability and Performance

- **Horizontal Scaling**: Scale high-traffic services (e.g., Product Catalog, Search) using Kubernetes.
- **Database Optimization**:
  - MongoDB: Use indexes (e.g., `productId` in `inventory_db`, `title` in `product_db`) and sharding for large datasets.
  - Elasticsearch: Optimize mappings and analyzers for search performance.
- **Caching**: Use Redis for caching search results or frequently accessed data (e.g., product details).
- **Event Reliability**: RabbitMQ durable queues and retries ensure no event loss.

## Deployment Plan

- **Local Development**: Use Docker Compose to run all services, MongoDB databases, Elasticsearch, and RabbitMQ.

  - Example `docker-compose.yml` snippet:

    ```yaml
    services:
      api-gateway:
        build: ./api-gateway
        ports:
          - "3000:3000"
      user-service:
        build: ./user-service
        ports:
          - "3001:3001"
        depends_on:
          - user_db
      user_db:
        image: mongo:latest
        ports:
          - "27017:27017"
      # Add other services and databases
      elasticsearch:
        image: elasticsearch:8.8.0
        ports:
          - "9200:9200"
      rabbitmq:
        image: rabbitmq:3-management
        ports:
          - "5672:5672"
          - "15672:15672"
    ```

- **Production**: Deploy on AWS/GCP with Kubernetes, MongoDB Atlas for managed databases, and Elastic Cloud for Elasticsearch.
- **CI/CD**: Use GitHub Actions for automated testing and deployment.

## Future Enhancements

- **Recommendation Service**: Add personalized product suggestions using machine learning.
- **Analytics Service**: Track user behavior and sales metrics.
- **Advanced Search**: Implement autocomplete and fuzzy search in Elasticsearch.
- **Database Scaling**: Shard MongoDB databases for high-traffic services (e.g., `product_db`, `order_db`).

## Conclusion

This architecture provides a robust, scalable foundation for the e-commerce platform, with 10 microservices and an API Gateway covering all core functionalities (user management, product catalog, inventory, orders, payments, cart, shipping, reviews, search, notifications). The database-per-service approach ensures independence, while Elasticsearch powers advanced search, and RabbitMQ enables loose coupling. The system is designed for an MVP but is extensible for future growth.
