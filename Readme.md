# E-commerce Multi-Tenant Microservices Architecture Documentation

## Overview

This document outlines the architecture of a **multi-tenant e-commerce platform** built using a microservices approach, designed for scalability, modularity, and flexibility. The platform enables multiple clients (tenants) to operate their own **dedicated admin panels** and **websites**, fetching data via a centralized **API Gateway**. Each tenant’s data is logically separated using a `tenant_id`, supporting **dynamic product fields** for diverse product types. The system leverages **Node.js with TypeScript**, **MongoDB** (separate database per service with tenant-specific collections), **RabbitMQ** for event-driven communication, **REST APIs** for inter-service and client communication, **Elasticsearch** for search, and **AWS S3** for storing product images. The platform consists of **10 microservices** and an **API Gateway**, deployed using **Docker** containers orchestrated by **Kubernetes**.

### Tech Stack

- **Backend**: Node.js with TypeScript, Express.js for REST APIs.
- **Databases**: MongoDB (separate database per service, tenant-specific collections), Elasticsearch for Search Service.
- **Messaging**: RabbitMQ for event-driven communication.
- **Storage**: AWS S3 for product images.
- **API Gateway**: Node.js with Express or Kong.
- **Frontend**: Client-specific admin panels and websites (framework-agnostic, e.g., React, Next.js).
- **Deployment**: Docker for containerization, Kubernetes for orchestration.
- **Security**: JWT for authentication, HTTPS for APIs, MongoDB role-based access, AWS IAM for S3.

### Architecture Principles

- **Multi-Tenancy**: Logical separation of tenant data using `tenant_id` in all databases, ensuring isolation between clients.
- **Database per Service**: Each microservice has its own MongoDB database, with tenant-specific collections or filters.
- **Dynamic Fields**: MongoDB’s flexible schema and Elasticsearch’s dynamic mappings support tenant-specific product attributes.
- **Event-Driven Communication**: RabbitMQ enables asynchronous events (e.g., order creation triggers inventory updates).
- **REST APIs**: Services expose REST endpoints for synchronous communication, accessible to client websites and admin panels.
- **Single Entry Point**: The API Gateway routes client requests, enforces tenancy, and handles authentication.
- **Image Storage**: Product images are stored in AWS S3, with tenant-specific prefixes for isolation.

## Multi-Tenant Model

- **Tenants**: Each client (shop owner) is a tenant, identified by a unique `tenant_id` (e.g., UUID or shop domain like `example-shop`).
- **Admin Panel**: Each tenant has a dedicated admin panel (e.g., `admin.example-shop.com`) for managing products, orders, etc., built with a frontend framework (e.g., React) and fetching data via the API Gateway.
- **Website**: Each tenant has a dedicated website (e.g., `www.example-shop.com`) for customers, rendering products and handling checkout via API calls.
- **Data Isolation**: All MongoDB collections include a `tenant_id` field to filter tenant-specific data. Elasticsearch indexes are tenant-specific or filtered by `tenant_id`.
- **Dynamic Fields**: Tenant-specific product attributes (e.g., `size` for clothing, `voltage` for electronics) are stored in a `metafields` object in MongoDB and indexed in Elasticsearch.

## Microservices Overview

The platform comprises 10 microservices and 1 API Gateway, updated to support multi-tenancy, dynamic fields, and S3 image storage. Below is a detailed description of each component.

### 1. API Gateway

- **Purpose**: Acts as the single entry point for all client requests (admin panels, websites), routing them to microservices, enforcing tenancy, and handling authentication.
- **Functionalities**:
  - Route requests to microservices based on URL paths (e.g., `/api/{tenant_id}/products`).
  - Authenticate requests using JWT tokens, validated via User Service.
  - Enforce tenancy by extracting `tenant_id` from URLs or JWT claims.
  - Rate limiting to prevent abuse.
  - Logging and monitoring.
- **Database**: None (stateless).
- **APIs**:
  - Proxy endpoints: `GET /api/{tenant_id}/products`, `POST /api/{tenant_id}/orders`.
  - Example: `GET /api/example-shop/products` routes to Product Catalog Service with `tenant_id = example-shop`.
- **Events**: None.
- **Interactions**:
  - Receives requests from tenant admin panels and websites.
  - Forwards requests to services with `tenant_id` in headers (e.g., `X-Tenant-Id: example-shop`).
  - Validates JWT tokens via User Service (`/api/users/validate-token`).
- **Tech Details**:
  - Framework: Node.js + Express + TypeScript or Kong.
  - Libraries: `express-gateway`, `http-proxy-middleware`, `jsonwebtoken`.
  - Port: 3000.
- **Deployment**:
  - Docker container with multiple replicas (e.g., 3).
  - Kubernetes Deployment, Service (ClusterIP), Ingress for external access.
  - Horizontal Pod Autoscaler (HPA) for scaling.
- **Security**: HTTPS, JWT validation, tenant_id enforcement, rate limiting (`express-rate-limit`).

### 2. User Service

- **Purpose**: Manages user accounts, authentication, and authorization for each tenant.
- **Functionalities**:
  - User registration, login, password reset (JWT with `tenant_id`).
  - Profile management (name, address).
  - Role-based access (customer, tenant admin).
- **Database**: MongoDB (`user_db`, collections: `users`, `sessions`).
  - `users`: `{ tenant_id, user_id, email, hashed_password, name, role }`.
  - `sessions`: `{ tenant_id, user_id, jwt_token, expires_at }` with TTL.
  - Example document:
    ```javascript
    {
      tenant_id: "example-shop",
      user_id: "u123",
      email: "admin@example-shop.com",
      hashed_password: "...",
      role: "tenant_admin"
    }
    ```
- **APIs**:
  - `POST /api/users/register`: Create user for a tenant.
  - `POST /api/users/login`: Authenticate, return JWT with `tenant_id`.
  - `GET /api/users/profile`: Retrieve profile (requires JWT).
  - `POST /api/users/validate-token`: Validate JWT for API Gateway.
- **Events**:
  - Publishes: `user.created` (`tenant_id`, `user_id`, `email`) to RabbitMQ.
- **Interactions**:
  - Called by API Gateway for authentication.
  - Publishes events to Notification Service.
- **Tech Details**:
  - Framework: Node.js + Express + TypeScript.
  - Libraries: `jsonwebtoken`, `bcrypt`, `mongoose`.
  - Port: 3001.
- **Deployment**: Docker container, Kubernetes StatefulSet for `user_db`.
- **Security**: Encrypt passwords, tenant_id filtering.

### 3. Product Catalog Service

- **Purpose**: Manages product listings, categories, and dynamic fields for each tenant.
- **Functionalities**:
  - CRUD for products (title, price, images, metafields).
  - Category management.
  - Store images in AWS S3 with tenant-specific prefixes.
- **Database**: MongoDB (`product_db`, collections: `products`, `categories`).
  - `products`: `{ tenant_id, product_id, title, price, image_urls, metafields, categories }`.
  - `metafields`: Stores dynamic fields (e.g., `size`, `voltage`).
  - `image_urls`: References S3 paths (e.g., `s3://ecommerce-images/example-shop/products/123.jpg`).
  - Example document:
    ```javascript
    {
      tenant_id: "example-shop",
      product_id: "p123",
      title: "T-Shirt",
      price: 19.99,
      image_urls: ["s3://ecommerce-images/example-shop/products/p123.jpg"],
      metafields: { size: "Large", color: "Blue" },
      categories: ["clothing"]
    }
    ```
- **S3 Storage**:
  - Images uploaded to S3 bucket `ecommerce-images` with prefix `{tenant_id}/products/`.
  - Example upload (Node.js with AWS SDK):
    ```javascript
    const { S3 } = require("aws-sdk");
    const s3 = new S3();
    await s3
      .upload({
        Bucket: "ecommerce-images",
        Key: `${tenant_id}/products/${product_id}.jpg`,
        Body: imageBuffer,
        ContentType: "image/jpeg",
        ACL: "public-read",
      })
      .promise();
    ```
- **APIs**:
  - `GET /api/{tenant_id}/products`: List products (paginated).
  - `GET /api/{tenant_id}/products/{id}`: Get product details.
  - `POST /api/{tenant_id}/products`: Create product with image upload.
  - `PUT /api/{tenant_id}/products/{id}`: Update product, including metafields.
- **Events**:
  - Publishes: `product.created`, `product.updated`, `product.deleted` (`tenant_id`, `product_id`, details) to RabbitMQ.
- **Interactions**:
  - Called by Search Service for product data.
  - Uploads images to S3, stores URLs in MongoDB.
- **Tech Details**:
  - Framework: Node.js + Express + TypeScript.
  - Libraries: `mongoose`, `aws-sdk`.
  - Port: 3002.
- **Deployment**: Docker container, Kubernetes StatefulSet for `product_db`.
- **Security**: Tenant_id filtering, S3 IAM roles for uploads.

### 4. Inventory Service

- **Purpose**: Tracks product stock levels for each tenant.
- **Functionalities**:
  - Monitor/update stock quantities.
  - Reserve stock during checkout.
- **Database**: MongoDB (`inventory_db`, collection: `inventory`).
  - `inventory`: `{ tenant_id, product_id, stock, reserved }`.
- **APIs**:
  - `GET /api/{tenant_id}/inventory/{productId}`: Get stock.
  - `POST /api/{tenant_id}/inventory/reserve`: Reserve stock.
- **Events**:
  - Consumes: `order.created` (`tenant_id`, product IDs).
  - Publishes: `inventory.low` (`tenant_id`, `product_id`).
- **Interactions**:
  - Called by Order Service.
  - Publishes events to Notification/Search Services.
- **Tech Details**:
  - Framework: Node.js + Express + TypeScript.
  - Libraries: `mongoose`, `amqplib`.
  - Port: 3003.
- **Deployment**: Docker container, Kubernetes StatefulSet for `inventory_db`.
- **Security**: Tenant_id filtering.

### 5. Order Service

- **Purpose**: Manages order creation/lifecycle for each tenant.
- **Functionalities**:
  - Create/track orders.
  - Update status (pending, shipped).
- **Database**: MongoDB (`order_db`, collection: `orders`).
  - `orders`: `{ tenant_id, order_id, user_id, products, status, total }`.
- **APIs**:
  - `POST /api/{tenant_id}/orders`: Create order.
  - `GET /api/{tenant_id}/orders/{orderId}`: Get order details.
- **Events**:
  - Publishes: `order.created` (`tenant_id`, `order_id`, products).
- **Interactions**:
  - Calls Inventory, Payment, Shipping Services.
  - Publishes events to Notification/Search Services.
- **Tech Details**:
  - Framework: Node.js + Express + TypeScript.
  - Libraries: `mongoose`, `amqplib`, `axios`.
  - Port: 3004.
- **Deployment**: Docker container, Kubernetes StatefulSet for `order_db`.
- **Security**: Tenant_id filtering.

### 6. Payment Service

- **Purpose**: Handles payment processing for each tenant.
- **Functionalities**:
  - Process payments (e.g., Stripe).
  - Manage refunds.
- **Database**: MongoDB (`payment_db`, collection: `transactions`).
  - `transactions`: `{ tenant_id, order_id, amount, status }`.
- **APIs**:
  - `POST /api/{tenant_id}/payments`: Process payment.
  - `GET /api/{tenant_id}/payments/{orderId}`: Get transaction.
- **Events**:
  - Publishes: `payment.success`, `payment.failed` (`tenant_id`, `order_id`).
- **Interactions**:
  - Called by Order Service.
  - Publishes events to Notification Service.
- **Tech Details**:
  - Framework: Node.js + Express + TypeScript.
  - Libraries: `mongoose`, `amqplib`, `stripe`.
  - Port: 3005.
- **Deployment**: Docker container, Kubernetes StatefulSet for `payment_db`.
- **Security**: Encrypt transaction data, tenant_id filtering.

### 7. Cart Service

- **Purpose**: Manages user shopping carts for each tenant.
- **Functionalities**:
  - Add/remove items.
  - Calculate totals.
- **Database**: MongoDB (`cart_db`, collection: `carts`).
  - `carts`: `{ tenant_id, user_id, products, total }`.
- **APIs**:
  - `GET /api/{tenant_id}/cart`: Get cart.
  - `POST /api/{tenant_id}/cart/add`: Add item.
- **Events**: None.
- **Interactions**:
  - Called by Order Service.
  - Queries Product Catalog Service.
- **Tech Details**:
  - Framework: Node.js + Express + TypeScript.
  - Libraries: `mongoose`, `axios`.
  - Port: 3006.
- **Deployment**: Docker container, Kubernetes StatefulSet for `cart_db`.
- **Security**: Tenant_id filtering.

### 8. Shipping Service

- **Purpose**: Manages shipping options/tracking for each tenant.
- **Functionalities**:
  - Calculate shipping costs.
  - Track shipments.
- **Database**: MongoDB (`shipping_db`, collection: `shipments`).
  - `shipments`: `{ tenant_id, order_id, provider, cost, tracking }`.
- **APIs**:
  - `POST /api/{tenant_id}/shipping/quote`: Get cost.
  - `GET /api/{tenant_id}/shipping/track/{orderId}`: Track shipment.
- **Events**:
  - Consumes: `order.created`.
  - Publishes: `shipping.updated` (`tenant_id`, `order_id`).
- **Interactions**:
  - Called by Order Service.
  - Publishes events to Notification Service.
- **Tech Details**:
  - Framework: Node.js + Express + TypeScript.
  - Libraries: `mongoose`, `amqplib`, `axios`.
  - Port: 3007.
- **Deployment**: Docker container, Kubernetes StatefulSet for `shipping_db`.
- **Security**: Tenant_id filtering.

### 9. Review Service

- **Purpose**: Manages customer reviews/ratings for each tenant.
- **Functionalities**:
  - Submit/edit reviews.
  - Calculate average ratings.
- **Database**: MongoDB (`review_db`, collection: `reviews`).
  - `reviews`: `{ tenant_id, product_id, user_id, rating, comment }`.
- **APIs**:
  - `POST /api/{tenant_id}/reviews`: Submit review.
  - `GET /api/{tenant_id}/reviews/product/{productId}`: Get reviews.
- **Events**:
  - Publishes: `review.created`, `review.updated` (`tenant_id`, `product_id`).
- **Interactions**:
  - Called by Search Service.
  - Publishes events to Search/Notification Services.
- **Tech Details**:
  - Framework: Node.js + Express + TypeScript.
  - Libraries: `mongoose`, `amqplib`.
  - Port: 3008.
- **Deployment**: Docker container, Kubernetes StatefulSet for `review_db`.
- **Security**: Tenant_id filtering.

### 10. Search Service

- **Purpose**: Provides search/filtering for products across tenants using Elasticsearch.
- **Functionalities**:
  - Full-text search (title, metafields).
  - Faceted search (category, price, metafields).
- **Database**: Elasticsearch (`products_index` per tenant or filtered by `tenant_id`).
  - Schema:
    ```json
    {
      "tenant_id": "example-shop",
      "product_id": "p123",
      "title": "T-Shirt",
      "price": 19.99,
      "metafields": { "size": "Large", "color": "Blue" },
      "categories": ["clothing"],
      "averageRating": 4.5,
      "stock": 100
    }
    ```
- **APIs**:
  - `GET /api/{tenant_id}/search?q={query}`: Full-text search.
  - `GET /api/{tenant_id}/search/filters?category={category}&size={size}`: Faceted search.
- **Events**:
  - Consumes: `product.created`, `product.updated`, `review.created` (`tenant_id`, data).
- **Interactions**:
  - Fetches data from Product Catalog, Review, Inventory Services via REST.
  - Indexes tenant-specific data in Elasticsearch.
- **Tech Details**:
  - Framework: Node.js + Express + TypeScript.
  - Libraries: `@elastic/elasticsearch`, `amqplib`, `axios`.
  - Port: 3009.
- **Deployment**: Docker container, Kubernetes StatefulSet for Elasticsearch.
- **Security**: Tenant_id filtering, Elasticsearch authentication.

### 11. Notification Service

- **Purpose**: Sends notifications (email, SMS) for each tenant.
- **Functionalities**:
  - Send order confirmations, shipping updates.
- **Database**: MongoDB (`notification_db`, collection: `notifications`).
  - `notifications`: `{ tenant_id, user_id, type, content, status }`.
- **APIs**:
  - `POST /api/{tenant_id}/notifications/send`: Trigger notification.
- **Events**:
  - Consumes: `order.created`, `payment.success`, `shipping.updated` (`tenant_id`, data).
- **Interactions**:
  - Uses `nodemailer` for email, Twilio for SMS.
- **Tech Details**:
  - Framework: Node.js + Express + TypeScript.
  - Libraries: `mongoose`, `amqplib`, `nodemailer`.
  - Port: 3010.
- **Deployment**: Docker container, Kubernetes StatefulSet for `notification_db`.
- **Security**: Tenant_id filtering, secure credentials.

## System Workflow Example (Product Display on Tenant Website)

1. **Client**: Visits `www.example-shop.com`, which sends `GET /api/example-shop/products` to API Gateway.
2. **API Gateway**: Validates JWT, extracts `tenant_id = example-shop`, routes to Product Catalog Service.
3. **Product Catalog Service**:
   - Queries `product_db`: `db.products.find({ tenant_id: "example-shop" }).limit(20)`.
   - Retrieves products with S3 image URLs and metafields (e.g., `size: "Large"`).
   - Returns JSON to API Gateway.
4. **Client Website**: Renders products using a frontend framework (e.g., React):
   ```javascript
   fetch("https://api.ecommerce.com/api/example-shop/products", {
     headers: { Authorization: "Bearer <jwt>" },
   })
     .then((res) => res.json())
     .then((products) => {
       products.forEach((p) => console.log(p.title, p.metafields.size));
     });
   ```
5. **Search (if used)**: A search for “blue t-shirt” hits Search Service, querying Elasticsearch with `tenant_id = example-shop`.
6. **Notification**: Product updates trigger `product.updated` events, notifying tenants via Notification Service.

## Infrastructure Setup

- **Docker**: Each microservice runs in a Docker container.
  - Example `Dockerfile`:
    ```dockerfile
    FROM node:18
    WORKDIR /app
    COPY package*.json ./
    RUN npm install
    COPY . .
    CMD ["npm", "run", "start"]
    ```
- **Kubernetes**:
  - Deployments for services (2-5 replicas).
  - StatefulSets for MongoDB databases and Elasticsearch.
  - Ingress for API Gateway (e.g., `api.ecommerce.com`).
  - HPA for scaling based on CPU/memory.
- **MongoDB**: 9 databases (`user_db`, `product_db`, etc.) with tenant_id filtering.
  - Example: `docker run -p 27017:27017 mongo:latest`.
- **Elasticsearch**: Single instance or tenant-specific indexes (`docker run -p 9200:9200 elasticsearch:8.8.0`).
- **RabbitMQ**: Durable queues (`docker run -p 5672:5672 rabbitmq:3-management`).
- **AWS S3**: Bucket `ecommerce-images` with tenant prefixes (e.g., `example-shop/products/`).
  - IAM policy:
    ```json
    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Action": ["s3:PutObject", "s3:GetObject"],
          "Resource": "arn:aws:s3:::ecommerce-images/*"
        }
      ]
    }
    ```
- **Monitoring**: Prometheus, Grafana, Winston for logging.

## Security Considerations

- **Authentication**: JWT tokens include `tenant_id`, validated by API Gateway/User Service.
- **Authorization**: Role-based access (tenant admin vs. customer).
- **Data Isolation**: All queries filter by `tenant_id` (e.g., `db.products.find({ tenant_id })`).
- **Encryption**: Sensitive data (passwords, payment details) encrypted in MongoDB.
- **S3 Security**: IAM roles, bucket policies restrict access to tenant prefixes.
- **API Security**: HTTPS, API keys for service-to-service communication.

## Scalability and Performance

- **Horizontal Scaling**: Kubernetes scales services (e.g., Product Catalog, Search) via replicas.
- **Database Optimization**:
  - MongoDB: Indexes on `tenant_id`, `product_id` (e.g., `db.products.createIndex({ tenant_id: 1, product_id: 1 })`).
  - Elasticsearch: Dynamic mappings for metafields, tenant-specific indexes.
- **Caching**: Redis for product data (e.g., `tenant:example-shop:products`).
- **S3**: CDN (CloudFront) for fast image delivery.
- **Event Reliability**: RabbitMQ durable queues ensure no event loss.

## Deployment Plan

- **Local Development**: Docker Compose for services, MongoDB, Elasticsearch, RabbitMQ.
  - Example `docker-compose.yml`:
    ```yaml
    services:
      api-gateway:
        build: ./api-gateway
        ports:
          - "3000:3000"
      product-catalog:
        build: ./product-catalog
        ports:
          - "3002:3002"
      product_db:
        image: mongo:latest
        ports:
          - "27018:27017"
      elasticsearch:
        image: elasticsearch:8.8.0
        ports:
          - "9200:9200"
      rabbitmq:
        image: rabbitmq:3-management
        ports:
          - "5672:5672"
    ```
- **Production**: AWS EKS for Kubernetes, MongoDB Atlas, Elastic Cloud, S3 with CloudFront.
- **CI/CD**: GitHub Actions for testing/deployment.

## Future Enhancements

- **Recommendation Service**: Personalized product suggestions.
- **Analytics Service**: Tenant-specific sales metrics.
- **Advanced Search**: Autocomplete, fuzzy search in Elasticsearch.
- **Tenant Onboarding**: Automated setup for new tenants.

## Conclusion

This multi-tenant e-commerce platform supports dedicated admin panels and websites for each client, fetching data via a centralized API Gateway. Logical separation with `tenant_id` ensures data isolation, while MongoDB’s flexible schema and Elasticsearch’s dynamic mappings handle diverse product fields. AWS S3 stores images with tenant-specific prefixes, and Docker/Kubernetes ensure scalable deployment. The architecture extends the original microservices design, providing a robust, extensible solution for multi-tenant e-commerce.
