# Darija Market — Technical Specification
# Version: 1.0 | Status: Pre-development

---

## IMPORTANT — READ FIRST

This document is the single source of truth for the Darija Market platform.
Do not invent types, field names, event names, or routes not defined here.
Do not add dependencies not listed in the stack section.
All shared types live in `packages/types`. Import from there, never redefine locally.
When in doubt: follow the schema, follow the state machine, follow the API contract.

---

## 1. SYSTEM OVERVIEW

Three-sided marketplace: Sellers, Delivery Providers, Customers.
Cash on delivery only. No payment gateway. No card processing.
Deposit-based trust system for delivery providers.
1% commission on completed transaction value, charged to seller.

### 1.1 Applications

| App | Platform | Package |
|-----|----------|---------|
| Seller App | Android (native) | `apps/seller` |
| Delivery Provider App | Android (native) | `apps/delivery` |
| Customer App Android | Android (native) | `apps/customer` |
| Customer PWA | Web (Next.js PWA) | `apps/customer-web` |
| API Server | Node.js | `apps/api` |
| Security Core | C (existing) | `apps/core-c` |

### 1.2 Architecture Diagram

```
[Seller App] ─────────────────────────────────┐
[Delivery App] ──── HTTPS + WSS ──────────────┤
[Customer App] ───────────────────────────────┤
[Customer PWA] ───────────────────────────────┤
                                              ▼
                                    [Fastify API Server]
                                    apps/api — TypeScript
                                          │
                      ┌───────────────────┼───────────────────┐
                      ▼                   ▼                   ▼
              [PostgreSQL DB]      [Redis Pub/Sub]    [C Security Core]
              apps/api/prisma      Session cache      apps/core-c
              Persistent data      Real-time events   POST validation
                                                      Session encoding
```

### 1.3 C Security Core Interface

The existing C backend runs as a separate process on the same machine.
The Fastify API server communicates with it via Unix socket at `/tmp/core.sock`.
The C core handles: session creation, per-session encoding scheme, POST body validation.
The Fastify layer handles: all business logic, database, file storage, WebSocket.

Every POST request from a client app MUST be proxied through the C core for validation
before the Fastify business logic layer processes the body.

C core endpoints (internal only, Unix socket):
- `VALIDATE_SESSION <session_id> <body_hex>` → `OK` or `REJECT`
- `CREATE_SESSION <user_id>` → `<session_token>`
- `DESTROY_SESSION <session_id>` → `OK`

---

## 2. MONOREPO STRUCTURE

```
darija-market/
├── apps/
│   ├── seller/          # React Native + Expo (Android)
│   ├── delivery/        # React Native + Expo (Android)
│   ├── customer/        # React Native + Expo (Android)
│   ├── customer-web/    # Next.js 14 PWA
│   ├── api/             # Fastify + TypeScript
│   └── core-c/          # Existing C security backend
├── packages/
│   ├── types/           # Shared TypeScript types (source of truth)
│   ├── utils/           # Shared utility functions
│   └── constants/       # Shared constants (order states, event names, etc.)
├── turbo.json
├── package.json         # Root workspace config
└── SPEC.md              # This file
```

---

## 3. TECHNOLOGY STACK

### 3.1 Stack Versions (pin these exactly)

```json
{
  "runtime": "Node.js 20 LTS",
  "typescript": "5.4.x",
  "expo": "51.x (SDK 51)",
  "react-native": "0.74.x",
  "next": "14.2.x",
  "fastify": "4.x",
  "prisma": "5.x",
  "socket.io": "4.7.x",
  "redis": "4.x (ioredis)",
  "livekit-server-sdk": "2.x",
  "@livekit/react-native": "2.x",
  "zustand": "4.x",
  "@tanstack/react-query": "5.x",
  "zod": "3.x",
  "expo-router": "3.x",
  "expo-location": "17.x",
  "expo-camera": "15.x",
  "expo-notifications": "0.28.x",
  "react-native-maps": "1.14.x"
}
```

### 3.2 Package Manager

Use `pnpm` workspaces. Do not use npm or yarn.

```
pnpm-workspace.yaml:
  packages:
    - 'apps/*'
    - 'packages/*'
```

### 3.3 Build System

Turborepo. `turbo.json` defines pipeline for `build`, `dev`, `lint`, `type-check`.

---

## 4. DATABASE SCHEMA (PostgreSQL via Prisma)

File location: `apps/api/prisma/schema.prisma`
This is the authoritative data model. All TypeScript types are generated from this.

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// ─── ENUMS ───────────────────────────────────────────────────────────────────

enum UserRole {
  SELLER
  DELIVERY
  CUSTOMER
}

enum OrderStatus {
  PENDING           // Order placed, awaiting seller acceptance
  ACCEPTED          // Seller accepted, awaiting provider assignment
  PROVIDER_ASSIGNED // Seller chose provider, awaiting provider acceptance
  PROVIDER_ACCEPTED // Provider accepted, en route to seller
  AT_SELLER         // Provider arrived at seller location
  PICKED_UP         // Provider confirmed pickup (photo taken)
  IN_TRANSIT        // Provider en route to customer
  AT_CUSTOMER       // Provider arrived at customer location
  DELIVERED         // Provider confirmed delivery (photo taken, cash collected)
  CONFIRMED         // Customer confirmed receipt
  DISPUTED          // Customer or seller raised dispute
  RESOLVED          // Dispute resolved
  CANCELLED         // Order cancelled before pickup
  REJECTED          // Seller or provider rejected order
}

enum DisputeStatus {
  OPEN
  EVIDENCE_REVIEW
  RESOLVED_FOR_CUSTOMER
  RESOLVED_FOR_SELLER
  RESOLVED_FOR_PROVIDER
  ESCALATED_TO_COURT
  CLOSED
}

enum DepositTier {
  BASIC     // 200 MAD
  STANDARD  // 500 MAD
  PREMIUM   // 1000+ MAD
}

enum DepositStatus {
  ACTIVE
  FROZEN    // Dispute in progress
  RELEASED
  PENDING_DEPOSIT
}

enum VerticalCategory {
  FOOD
  CLOTHING
  TECH
  HOME_IMPROVEMENT
  TRANSPORT
  GENERAL
}

enum ServiceType {
  PRODUCT
  SERVICE
  PRODUCT_AND_SERVICE
}

enum SettlementStatus {
  PENDING
  COMPLETED
  DISPUTED
  FAILED
}

// ─── USERS ───────────────────────────────────────────────────────────────────

model User {
  id                String    @id @default(cuid())
  phone             String    @unique
  phoneVerified     Boolean   @default(false)
  role              UserRole
  name              String?
  avatarUrl         String?
  createdAt         DateTime  @default(now())
  updatedAt         DateTime  @updatedAt
  lastActiveAt      DateTime?
  isBlocked         Boolean   @default(false)
  blockedReason     String?

  // Relations by role
  sellerProfile     SellerProfile?
  deliveryProfile   DeliveryProfile?
  customerProfile   CustomerProfile?

  // Shared
  sentMessages      Message[]         @relation("MessageSender")
  notifications     Notification[]
  sessions          UserSession[]
}

model UserSession {
  id          String    @id @default(cuid())
  userId      String
  user        User      @relation(fields: [userId], references: [id])
  cSessionId  String    @unique  // Session ID in C core
  deviceId    String
  fcmToken    String?
  createdAt   DateTime  @default(now())
  expiresAt   DateTime
  isActive    Boolean   @default(true)

  @@index([userId])
  @@index([cSessionId])
}

// ─── SELLER ──────────────────────────────────────────────────────────────────

model SellerProfile {
  id              String    @id @default(cuid())
  userId          String    @unique
  user            User      @relation(fields: [userId], references: [id])
  storeName       String
  description     String?
  coverImageUrl   String?
  isOpen          Boolean   @default(false)
  openFromHour    Int?      // 0-23
  openToHour      Int?      // 0-23
  minOrderAmount  Float     @default(0)
  prepTimeMinutes Int       @default(15)
  
  // Location
  lat             Float
  lng             Float
  address         String    // Text description
  deliveryRadius  Float     @default(3000)  // meters
  
  // Registration (optional for informal sellers)
  iceNumber       String?   // Moroccan tax ID
  rcNumber        String?   // Registre de Commerce
  
  // Scores
  ratingAvg       Float     @default(5.0)
  completionRate  Float     @default(1.0)
  disputeRate     Float     @default(0.0)
  totalOrders     Int       @default(0)

  products        Product[]
  orders          Order[]   @relation("SellerOrders")
  preferredProviders SellerPreferredProvider[]
  erpInventory    ErpInventoryItem[]
  erpCustomers    ErpCustomer[]
  invoices        Invoice[]
  
  createdAt       DateTime  @default(now())
  updatedAt       DateTime  @updatedAt
}

model SellerPreferredProvider {
  id              String          @id @default(cuid())
  sellerProfileId String
  seller          SellerProfile   @relation(fields: [sellerProfileId], references: [id])
  deliveryProfileId String
  delivery        DeliveryProfile @relation(fields: [deliveryProfileId], references: [id])
  createdAt       DateTime        @default(now())

  @@unique([sellerProfileId, deliveryProfileId])
}

// ─── DELIVERY PROVIDER ───────────────────────────────────────────────────────

model DeliveryProfile {
  id                  String        @id @default(cuid())
  userId              String        @unique
  user                User          @relation(fields: [userId], references: [id])
  
  // Status
  isActive            Boolean       @default(false)  // Online/offline toggle
  currentLat          Float?
  currentLng          Float?
  activeRadiusMeters  Float         @default(3000)
  
  // Vehicle
  vehicleType         String        // motorcycle, car, bicycle, foot
  vehicleDescription  String?
  
  // Deposit
  depositTier         DepositTier   @default(BASIC)
  depositAmountMad    Float         @default(0)
  depositStatus       DepositStatus @default(PENDING_DEPOSIT)
  depositFrozenReason String?
  
  // Behavioral scores
  ratingAvg           Float         @default(5.0)
  acceptanceRate      Float         @default(1.0)
  completionRate      Float         @default(1.0)
  onTimeRate          Float         @default(1.0)
  disputeRate         Float         @default(0.0)
  totalDeliveries     Int           @default(0)
  
  orders              Order[]       @relation("ProviderOrders")
  preferredBySellers  SellerPreferredProvider[]
  depositTransactions DepositTransaction[]
  
  createdAt           DateTime      @default(now())
  updatedAt           DateTime      @updatedAt
}

model DepositTransaction {
  id                String          @id @default(cuid())
  deliveryProfileId String
  delivery          DeliveryProfile @relation(fields: [deliveryProfileId], references: [id])
  amountMad         Float
  type              String          // DEPOSIT, RELEASE, PARTIAL_RELEASE, FREEZE, UNFREEZE
  reason            String?
  agentReference    String?         // CashPlus / Barid Cash agent receipt number
  confirmedByAdmin  Boolean         @default(false)
  createdAt         DateTime        @default(now())
}

// ─── CUSTOMER ────────────────────────────────────────────────────────────────

model CustomerProfile {
  id                  String    @id @default(cuid())
  userId              String    @unique
  user                User      @relation(fields: [userId], references: [id])
  
  // Behavioral scores
  completionRate      Float     @default(1.0)  // Was home when delivery arrived
  cashAvailabilityRate Float    @default(1.0)  // Had correct cash amount
  disputeRate         Float     @default(0.0)
  cancelRate          Float     @default(0.0)
  totalOrders         Int       @default(0)
  
  savedAddresses      SavedAddress[]
  orders              Order[]   @relation("CustomerOrders")
  
  createdAt           DateTime  @default(now())
  updatedAt           DateTime  @updatedAt
}

model SavedAddress {
  id                String          @id @default(cuid())
  customerProfileId String
  customer          CustomerProfile @relation(fields: [customerProfileId], references: [id])
  label             String          // "Home", "Work", custom label
  lat               Float
  lng               Float
  description       String          // Free text address description
  landmarkPhotoUrl  String?
  isDefault         Boolean         @default(false)
  createdAt         DateTime        @default(now())
}

// ─── PRODUCTS ────────────────────────────────────────────────────────────────

model Product {
  id                String          @id @default(cuid())
  sellerProfileId   String
  seller            SellerProfile   @relation(fields: [sellerProfileId], references: [id])
  
  name              String
  description       String?
  category          VerticalCategory
  serviceType       ServiceType     @default(PRODUCT)
  
  priceMad          Float
  isAvailable       Boolean         @default(true)
  prepTimeMinutes   Int?            // Override seller default
  
  // Inventory link (optional)
  erpInventoryItemId String?        @unique
  erpInventoryItem  ErpInventoryItem? @relation(fields: [erpInventoryItemId], references: [id])
  
  images            ProductImage[]
  attributes        ProductAttribute[]
  orderItems        OrderItem[]
  
  // Clothing-specific
  sizes             ProductSize[]
  
  createdAt         DateTime        @default(now())
  updatedAt         DateTime        @updatedAt
  
  @@index([sellerProfileId, category])
  @@index([sellerProfileId, isAvailable])
}

model ProductImage {
  id        String  @id @default(cuid())
  productId String
  product   Product @relation(fields: [productId], references: [id], onDelete: Cascade)
  url       String
  order     Int     @default(0)
}

model ProductAttribute {
  id        String  @id @default(cuid())
  productId String
  product   Product @relation(fields: [productId], references: [id], onDelete: Cascade)
  key       String
  value     String
}

model ProductSize {
  id          String  @id @default(cuid())
  productId   String
  product     Product @relation(fields: [productId], references: [id], onDelete: Cascade)
  label       String  // S, M, L, XL, 38, 40, etc.
  isAvailable Boolean @default(true)
  stockQty    Int?
}

// ─── ORDERS ──────────────────────────────────────────────────────────────────

model Order {
  id                  String          @id @default(cuid())
  
  // Parties
  sellerProfileId     String
  seller              SellerProfile   @relation("SellerOrders", fields: [sellerProfileId], references: [id])
  deliveryProfileId   String?
  provider            DeliveryProfile? @relation("ProviderOrders", fields: [deliveryProfileId], references: [id])
  customerProfileId   String
  customer            CustomerProfile @relation("CustomerOrders", fields: [customerProfileId], references: [id])
  
  // Status
  status              OrderStatus     @default(PENDING)
  statusHistory       OrderStatusEvent[]
  
  // Financials
  subtotalMad         Float           // Sum of order items
  deliveryFeeMad      Float
  commissionMad       Float           // 1% of subtotalMad
  totalMad            Float           // subtotalMad + deliveryFeeMad (customer pays this in cash)
  
  // Delivery address
  deliveryLat         Float
  deliveryLng         Float
  deliveryDescription String
  deliveryLandmarkPhotoUrl String?
  
  // Items
  items               OrderItem[]
  
  // Evidence (populated during delivery flow)
  pickupPhotoUrl      String?
  pickupLat           Float?
  pickupLng           Float?
  pickupConfirmedAt   DateTime?
  deliveryPhotoUrl    String?
  deliveryLat         Float?
  deliveryLng         Float?
  deliveryConfirmedAt DateTime?
  
  // Dispute
  dispute             Dispute?
  
  // Settlement
  settlement          Settlement?
  
  // Notes
  customerNote        String?
  
  messages            Message[]
  
  createdAt           DateTime        @default(now())
  updatedAt           DateTime        @updatedAt
  
  @@index([sellerProfileId, status])
  @@index([deliveryProfileId, status])
  @@index([customerProfileId, status])
}

model OrderItem {
  id          String    @id @default(cuid())
  orderId     String
  order       Order     @relation(fields: [orderId], references: [id])
  productId   String
  product     Product   @relation(fields: [productId], references: [id])
  quantity    Int
  unitPrice   Float     // Price at time of order (snapshot)
  selectedSize String?  // For clothing
  note        String?
}

model OrderStatusEvent {
  id        String      @id @default(cuid())
  orderId   String
  order     Order       @relation(fields: [orderId], references: [id])
  status    OrderStatus
  lat       Float?
  lng       Float?
  note      String?
  createdAt DateTime    @default(now())

  @@index([orderId])
}

// ─── DISPUTES ────────────────────────────────────────────────────────────────

model Dispute {
  id            String        @id @default(cuid())
  orderId       String        @unique
  order         Order         @relation(fields: [orderId], references: [id])
  
  status        DisputeStatus @default(OPEN)
  initiatedBy   String        // userId
  reason        String
  
  evidence      DisputeEvidence[]
  resolution    String?
  resolvedAt    DateTime?
  courtOrderRef String?
  
  createdAt     DateTime      @default(now())
  updatedAt     DateTime      @updatedAt
}

model DisputeEvidence {
  id          String   @id @default(cuid())
  disputeId   String
  dispute     Dispute  @relation(fields: [disputeId], references: [id])
  submittedBy String   // userId
  type        String   // PHOTO, TEXT, CHAT_LOG, GPS_LOG
  content     String   // URL for photos, text content, JSON for logs
  createdAt   DateTime @default(now())
}

// ─── SETTLEMENT ──────────────────────────────────────────────────────────────

model Settlement {
  id                  String          @id @default(cuid())
  orderId             String          @unique
  order               Order           @relation(fields: [orderId], references: [id])
  
  status              SettlementStatus @default(PENDING)
  subtotalMad         Float
  deliveryFeeMad      Float
  commissionMad       Float
  sellerReceivesMad   Float           // subtotalMad - commissionMad
  providerReceivesMad Float           // deliveryFeeMad
  
  settledAt           DateTime?
  notes               String?
  
  createdAt           DateTime        @default(now())
}

// ─── MESSAGING ───────────────────────────────────────────────────────────────

model Message {
  id          String    @id @default(cuid())
  orderId     String
  order       Order     @relation(fields: [orderId], references: [id])
  senderId    String
  sender      User      @relation("MessageSender", fields: [senderId], references: [id])
  recipientId String    // Direct recipient userId
  content     String
  type        String    @default("TEXT")  // TEXT, IMAGE, SYSTEM
  readAt      DateTime?
  createdAt   DateTime  @default(now())

  @@index([orderId])
}

// ─── NOTIFICATIONS ───────────────────────────────────────────────────────────

model Notification {
  id        String    @id @default(cuid())
  userId    String
  user      User      @relation(fields: [userId], references: [id])
  title     String
  body      String
  data      Json?
  readAt    DateTime?
  createdAt DateTime  @default(now())

  @@index([userId, readAt])
}

// ─── ERP MODELS ──────────────────────────────────────────────────────────────

model ErpInventoryItem {
  id                String        @id @default(cuid())
  sellerProfileId   String
  seller            SellerProfile @relation(fields: [sellerProfileId], references: [id])
  name              String
  sku               String?
  category          String?
  costPriceMad      Float?
  sellPriceMad      Float?
  stockQty          Int           @default(0)
  lowStockThreshold Int           @default(5)
  unit              String        @default("piece")
  product           Product?      // Link to marketplace listing
  createdAt         DateTime      @default(now())
  updatedAt         DateTime      @updatedAt

  @@index([sellerProfileId])
}

model ErpCustomer {
  id                String        @id @default(cuid())
  sellerProfileId   String
  seller            SellerProfile @relation(fields: [sellerProfileId], references: [id])
  name              String
  phone             String?
  address           String?
  notes             String?
  tags              String[]
  createdAt         DateTime      @default(now())
  updatedAt         DateTime      @updatedAt

  @@index([sellerProfileId])
}

model Invoice {
  id                String        @id @default(cuid())
  sellerProfileId   String
  seller            SellerProfile @relation(fields: [sellerProfileId], references: [id])
  invoiceNumber     String        @unique
  clientName        String
  clientAddress     String?
  clientIce         String?
  items             Json          // Array of {description, qty, unitPrice, total}
  subtotalMad       Float
  tvaMad            Float
  totalMad          Float
  tvaRate           Float         @default(0.20)
  issuedAt          DateTime      @default(now())
  pdfUrl            String?

  @@index([sellerProfileId])
}
```

---

## 5. SHARED TYPES PACKAGE

File: `packages/types/src/index.ts`
Generated from Prisma + additional API-layer types.
Never define these types locally in any app. Always import from `@darija-market/types`.

```typescript
// Re-export all Prisma-generated types
export type {
  User, UserRole, UserSession,
  SellerProfile, DeliveryProfile, CustomerProfile,
  Product, ProductImage, VerticalCategory, ServiceType,
  Order, OrderStatus, OrderItem, OrderStatusEvent,
  Dispute, DisputeStatus, DisputeEvidence,
  Settlement, SettlementStatus,
  DeliveryProfile, DepositTier, DepositStatus,
  Message, Notification,
  ErpInventoryItem, ErpCustomer, Invoice,
} from '@prisma/client'

// ─── API RESPONSE ENVELOPE ───────────────────────────────────────────────────

export interface ApiResponse<T> {
  success: boolean
  data?: T
  error?: {
    code: string
    message: string
    details?: unknown
  }
  meta?: {
    page?: number
    pageSize?: number
    total?: number
  }
}

// ─── AUTH ─────────────────────────────────────────────────────────────────────

export interface AuthPayload {
  userId: string
  role: UserRole
  sessionId: string
}

export interface LoginResponse {
  accessToken: string
  refreshToken: string
  user: UserPublic
  profile: SellerProfilePublic | DeliveryProfilePublic | CustomerProfilePublic
}

// ─── PUBLIC PROFILES (safe to send to other parties) ─────────────────────────

export interface UserPublic {
  id: string
  name: string | null
  avatarUrl: string | null
  role: UserRole
}

export interface SellerProfilePublic {
  id: string
  storeName: string
  description: string | null
  coverImageUrl: string | null
  isOpen: boolean
  lat: number
  lng: number
  address: string
  deliveryRadius: number
  ratingAvg: number
  completionRate: number
  disputeRate: number
  totalOrders: number
  category: VerticalCategory[]
}

export interface DeliveryProfilePublic {
  id: string
  depositTier: DepositTier
  depositAmountMad: number
  vehicleType: string
  ratingAvg: number
  acceptanceRate: number
  completionRate: number
  onTimeRate: number
  disputeRate: number
  totalDeliveries: number
}

export interface CustomerProfilePublic {
  id: string
  completionRate: number
  cashAvailabilityRate: number
  disputeRate: number
  totalOrders: number
}

// ─── ORDER FLOW TYPES ─────────────────────────────────────────────────────────

export interface OrderWithDetails extends Order {
  seller: SellerProfile & { user: UserPublic }
  provider: (DeliveryProfile & { user: UserPublic }) | null
  customer: CustomerProfile & { user: UserPublic }
  items: (OrderItem & { product: Product })[]
  statusHistory: OrderStatusEvent[]
  dispute: Dispute | null
}

export interface NearbyProvider {
  profile: DeliveryProfilePublic
  user: UserPublic
  distanceMeters: number
  estimatedArrivalMinutes: number
  currentLat: number
  currentLng: number
}

// ─── SOCKET EVENT PAYLOADS ────────────────────────────────────────────────────
// These must match exactly in server and all clients.
// See Section 6 for full event definitions.

export interface LocationUpdatePayload {
  userId: string
  lat: number
  lng: number
  timestamp: number
}

export interface OrderStatusUpdatePayload {
  orderId: string
  status: OrderStatus
  lat?: number
  lng?: number
  note?: string
  timestamp: number
}

export interface NewMessagePayload {
  messageId: string
  orderId: string
  senderId: string
  content: string
  type: string
  createdAt: string
}

export interface CallSignalPayload {
  orderId: string
  callerId: string
  livekitRoomName: string
  livekitToken: string
}
```

---

## 6. API CONTRACT

Base URL: `https://api.darijamarket.ma/v1`
All endpoints require `Authorization: Bearer <accessToken>` unless marked PUBLIC.
All request bodies must pass C core validation before reaching Fastify handlers.
All responses follow the `ApiResponse<T>` envelope.

### 6.1 Auth Routes

```
POST   /auth/request-otp          PUBLIC  { phone: string }
POST   /auth/verify-otp           PUBLIC  { phone: string, otp: string, role: UserRole }
POST   /auth/refresh              PUBLIC  { refreshToken: string }
POST   /auth/logout               AUTH    {}
```

### 6.2 Seller Routes

```
GET    /seller/profile                           SELLER  → SellerProfile
PATCH  /seller/profile                           SELLER  → SellerProfile
GET    /seller/products                          SELLER  → Product[]
POST   /seller/products                          SELLER  → Product
PATCH  /seller/products/:id                      SELLER  → Product
DELETE /seller/products/:id                      SELLER  → void
PATCH  /seller/products/:id/toggle               SELLER  → { isAvailable: boolean }

GET    /seller/orders                            SELLER  → OrderWithDetails[]  (paginated, filter by status)
GET    /seller/orders/:id                        SELLER  → OrderWithDetails
POST   /seller/orders/:id/accept                 SELLER  → OrderWithDetails
POST   /seller/orders/:id/reject                 SELLER  → OrderWithDetails  { reason: string }
POST   /seller/orders/:id/ready                  SELLER  → OrderWithDetails
POST   /seller/orders/:id/assign-provider        SELLER  → OrderWithDetails  { deliveryProfileId: string }

GET    /seller/providers/nearby                  SELLER  → NearbyProvider[]  (for order assignment)
GET    /seller/providers/preferred               SELLER  → DeliveryProfilePublic[]
POST   /seller/providers/preferred               SELLER  → void  { deliveryProfileId: string }
DELETE /seller/providers/preferred/:id           SELLER  → void

GET    /seller/erp/inventory                     SELLER  → ErpInventoryItem[]
POST   /seller/erp/inventory                     SELLER  → ErpInventoryItem
PATCH  /seller/erp/inventory/:id                 SELLER  → ErpInventoryItem
DELETE /seller/erp/inventory/:id                 SELLER  → void

GET    /seller/erp/customers                     SELLER  → ErpCustomer[]
POST   /seller/erp/customers                     SELLER  → ErpCustomer
PATCH  /seller/erp/customers/:id                 SELLER  → ErpCustomer

GET    /seller/invoices                          SELLER  → Invoice[]
POST   /seller/invoices                          SELLER  → Invoice  (generates PDF)
GET    /seller/invoices/:id/pdf                  SELLER  → PDF stream

GET    /seller/dashboard                         SELLER  → DashboardStats
```

### 6.3 Delivery Provider Routes

```
GET    /delivery/profile                         DELIVERY → DeliveryProfile
PATCH  /delivery/profile                         DELIVERY → DeliveryProfile
POST   /delivery/profile/toggle-active           DELIVERY → { isActive: boolean }
PATCH  /delivery/profile/location                DELIVERY → void  { lat, lng }

GET    /delivery/jobs/available                  DELIVERY → (Order & { distanceMeters })[]
POST   /delivery/jobs/:orderId/accept            DELIVERY → OrderWithDetails
POST   /delivery/jobs/:orderId/decline           DELIVERY → void

GET    /delivery/orders                          DELIVERY → OrderWithDetails[]  (active + history)
GET    /delivery/orders/:id                      DELIVERY → OrderWithDetails
POST   /delivery/orders/:id/arrived-seller       DELIVERY → OrderWithDetails  { lat, lng }
POST   /delivery/orders/:id/confirm-pickup       DELIVERY → OrderWithDetails  { lat, lng, photoUrl }
POST   /delivery/orders/:id/arrived-customer     DELIVERY → OrderWithDetails  { lat, lng }
POST   /delivery/orders/:id/confirm-delivery     DELIVERY → OrderWithDetails  { lat, lng, photoUrl }

GET    /delivery/deposit                         DELIVERY → { tier, amountMad, status, transactions: [] }
POST   /delivery/deposit/request                 DELIVERY → void  { agentReference, amountMad }

GET    /delivery/earnings                        DELIVERY → EarningsSummary
```

### 6.4 Customer Routes

```
GET    /customer/profile                         CUSTOMER → CustomerProfile
PATCH  /customer/profile                         CUSTOMER → CustomerProfile

GET    /customer/sellers                         CUSTOMER → SellerProfilePublic[]  
                                                           query: { lat, lng, radius, category, search }
GET    /customer/sellers/:id                     CUSTOMER → SellerProfilePublic & { products: Product[] }
GET    /customer/sellers/:id/products            CUSTOMER → Product[]  (query: category)

POST   /customer/orders                          CUSTOMER → OrderWithDetails
GET    /customer/orders                          CUSTOMER → OrderWithDetails[]
GET    /customer/orders/:id                      CUSTOMER → OrderWithDetails
POST   /customer/orders/:id/confirm              CUSTOMER → OrderWithDetails
POST   /customer/orders/:id/dispute              CUSTOMER → Dispute  { reason: string }
POST   /customer/orders/:id/cancel              CUSTOMER → OrderWithDetails

GET    /customer/addresses                       CUSTOMER → SavedAddress[]
POST   /customer/addresses                       CUSTOMER → SavedAddress
PATCH  /customer/addresses/:id                   CUSTOMER → SavedAddress
DELETE /customer/addresses/:id                   CUSTOMER → void
```

### 6.5 Shared Routes

```
POST   /upload/image                             AUTH    → { url: string }  (multipart/form-data)
GET    /verticals                                PUBLIC  → VerticalCategory[]

POST   /messages/:orderId                        AUTH    → Message  { content, type }
GET    /messages/:orderId                        AUTH    → Message[]

POST   /calls/:orderId/token                     AUTH    → { livekitToken, roomName }

POST   /disputes/:id/evidence                    AUTH    → DisputeEvidence
```

---

## 7. WEBSOCKET EVENTS (Socket.io)

Connection: `wss://api.darijamarket.ma`
Auth: Pass access token as handshake auth: `{ token: string }`

### 7.1 Client → Server Events

```typescript
// Join the room for a specific order (call after receiving order)
socket.emit('join:order', { orderId: string })

// Leave order room
socket.emit('leave:order', { orderId: string })

// Delivery provider: broadcast location (only when active delivery in progress)
socket.emit('location:update', { lat: number, lng: number })

// Send a message in an order thread
socket.emit('message:send', { orderId: string, content: string, type: 'TEXT' | 'IMAGE' })

// Typing indicator
socket.emit('message:typing', { orderId: string, recipientId: string })
```

### 7.2 Server → Client Events

```typescript
// Order status changed — all parties in the order room receive this
socket.on('order:status', (payload: OrderStatusUpdatePayload) => {})

// Delivery provider location update — customer and seller receive this during transit
socket.on('location:provider', (payload: LocationUpdatePayload) => {})

// New message in order thread
socket.on('message:new', (payload: NewMessagePayload) => {})

// Typing indicator
socket.on('message:typing', (payload: { senderId: string, orderId: string }) => {})

// Incoming call (LiveKit)
socket.on('call:incoming', (payload: CallSignalPayload) => {})

// New order available (delivery provider job feed)
socket.on('job:available', (payload: { orderId: string, distanceMeters: number }) => {})

// Order assigned to provider (after seller selects them)
socket.on('job:assigned', (payload: { orderId: string }) => {})

// Push notification fallback (for background state)
socket.on('notification:push', (payload: { title: string, body: string, data: object }) => {})
```

### 7.3 Socket Rooms

```
Room naming:
  order:{orderId}          All parties in this order
  provider:{userId}        Delivery provider's personal room (job feed)
  seller:{sellerProfileId} Seller's personal room (incoming orders)
```

---

## 8. ORDER STATE MACHINE

This is the authoritative definition of valid order state transitions.
The API server MUST reject any transition not listed here.
All transitions must be logged to `OrderStatusEvent`.

```
PENDING
  → ACCEPTED          (actor: SELLER)
  → REJECTED          (actor: SELLER)
  → CANCELLED         (actor: CUSTOMER — only before ACCEPTED)

ACCEPTED
  → PROVIDER_ASSIGNED (actor: SELLER — after selecting provider)
  → CANCELLED         (actor: SELLER — if no provider found)

PROVIDER_ASSIGNED
  → PROVIDER_ACCEPTED (actor: DELIVERY — provider accepts job)
  → ACCEPTED          (actor: DELIVERY — provider declines, returns to unassigned)

PROVIDER_ACCEPTED
  → AT_SELLER         (actor: DELIVERY — confirmed arrival at seller)

AT_SELLER
  → PICKED_UP         (actor: DELIVERY — requires pickupPhotoUrl + GPS)

PICKED_UP
  → IN_TRANSIT        (actor: SYSTEM — auto after pickup confirmed)

IN_TRANSIT
  → AT_CUSTOMER       (actor: DELIVERY — confirmed arrival at customer)

AT_CUSTOMER
  → DELIVERED         (actor: DELIVERY — requires deliveryPhotoUrl + GPS)
  → DISPUTED          (actor: DELIVERY — unable to deliver, customer not there)

DELIVERED
  → CONFIRMED         (actor: CUSTOMER — explicit confirm)
  → DISPUTED          (actor: CUSTOMER — within dispute window: 2 hours)
  → CONFIRMED         (actor: SYSTEM — auto-confirm after 24 hours if no dispute)

DISPUTED
  → RESOLVED          (actor: SYSTEM — after dispute resolution)

RESOLVED  (terminal)
CONFIRMED (terminal)
CANCELLED (terminal)
REJECTED  (terminal)
```

---

## 9. MOBILE APP SCREENS

### 9.1 Seller App — Screen List

```
Auth
  ├── PhoneEntry
  ├── OtpVerification
  └── ProfileSetup

Main (Bottom Tab Nav)
  ├── HomeTab
  │   ├── StoreStatusToggle        (open/closed switch — prominent)
  │   ├── ActiveOrdersList
  │   └── QuickStats               (today's orders, revenue)
  │
  ├── OrdersTab
  │   ├── OrdersList               (filter: active, pending, history)
  │   └── OrderDetail
  │       ├── OrderItems
  │       ├── ProviderSelector     (when status = ACCEPTED)
  │       ├── CustomerInfo
  │       ├── StatusTimeline
  │       └── OrderActions         (accept, reject, mark ready)
  │
  ├── ProductsTab
  │   ├── ProductsList
  │   ├── ProductForm              (create / edit)
  │   └── CategoryManager
  │
  ├── ErpTab
  │   ├── ErpDashboard             (revenue chart, top products)
  │   ├── InventoryList
  │   ├── InventoryItemForm
  │   ├── CustomersList            (CRM)
  │   ├── CustomerDetail
  │   ├── InvoicesList
  │   ├── InvoiceForm
  │   └── TradeWorkshop            (module builder — advanced)
  │
  └── ProfileTab
      ├── StoreProfile
      ├── PreferredProviders
      ├── PayoutSettings
      └── AppSettings
```

### 9.2 Delivery Provider App — Screen List

```
Auth
  ├── PhoneEntry
  ├── OtpVerification
  ├── ProfileSetup                 (vehicle type, area)
  └── DepositSetup                 (explain deposit tiers, guide to agent)

Main (Bottom Tab Nav)
  ├── JobsTab
  │   ├── ActiveStatus Toggle      (go online / go offline — PROMINENT)
  │   ├── JobFeed                  (list of available pickups)
  │   └── JobCard
  │       ├── SellerInfo + Rating
  │       ├── CustomerReliabilityBadge
  │       ├── DistanceAndFee
  │       ├── MapPreview
  │       └── AcceptDeclineActions
  │
  ├── ActiveDeliveryTab            (visible only when order in progress)
  │   ├── DeliveryMap              (navigation mode)
  │   ├── OrderSummary
  │   ├── ConfirmPickup            (camera + GPS)
  │   ├── ConfirmDelivery          (camera + GPS)
  │   └── ChatWithParties
  │
  ├── HistoryTab
  │   ├── DeliveryHistory
  │   └── DeliveryDetail
  │
  ├── EarningsTab
  │   ├── EarningsSummary
  │   ├── DepositStatus
  │   ├── DepositTierUpgrade
  │   └── DisputeLog
  │
  └── ProfileTab
      ├── ProviderProfile
      ├── VehicleInfo
      ├── ReputationScores
      └── AppSettings
```

### 9.3 Customer App — Screen List

```
Auth (optional — phone number only for first order)
  ├── PhoneEntry
  └── OtpVerification

Main (Bottom Tab Nav)
  ├── DiscoverTab
  │   ├── VerticalSelector         (Food, Clothing, Tech, Transport, Home, General)
  │   ├── FoodUI                   (rich photo browsing, cuisine filter)
  │   ├── ClothingUI               (grid photo browsing, size/category filter)
  │   ├── TechUI                   (spec-forward, search by model)
  │   ├── TransportUI              (origin/destination map, vehicle type)
  │   ├── HomeImprovementUI        (service + product hybrid, quote request)
  │   └── GeneralUI                (generic product grid)
  │
  ├── SellerProfile
  │   ├── StoreHeader
  │   ├── ProductGrid
  │   └── ProductDetail
  │
  ├── Cart + Checkout
  │   ├── CartSummary
  │   ├── AddressSelector          (saved addresses + new pin on map)
  │   └── OrderConfirmation
  │
  ├── OrdersTab
  │   ├── ActiveOrder
  │   │   ├── LiveTrackingMap      (provider location)
  │   │   ├── OrderStatusTimeline
  │   │   ├── ChatWithProvider
  │   │   ├── ConfirmDelivery
  │   │   └── DisputeFlow
  │   └── OrderHistory
  │
  └── ProfileTab
      ├── SavedAddresses
      ├── ReliabilityScore         (own score visibility)
      └── AppSettings
```

---

## 10. KEY IMPLEMENTATION REQUIREMENTS

### 10.1 Photo Upload Flow (Delivery Confirmation)

This is mandatory and blocking — delivery flow cannot proceed without it.

```typescript
// Pseudocode for delivery photo confirmation
async function confirmPickup(orderId: string) {
  // 1. Open camera (expo-camera, front camera disabled)
  const photo = await Camera.takePictureAsync({ quality: 0.7 })
  
  // 2. Get current GPS (expo-location)
  const location = await Location.getCurrentPositionAsync({ accuracy: Location.Accuracy.High })
  
  // 3. Upload photo to server
  const { url } = await uploadImage(photo.uri)
  
  // 4. Submit to API (C core validates this POST)
  await api.post(`/delivery/orders/${orderId}/confirm-pickup`, {
    lat: location.coords.latitude,
    lng: location.coords.longitude,
    photoUrl: url,
  })
  
  // 5. Socket event triggers update on seller and customer apps
}
```

### 10.2 Background Location (Delivery Provider)

Required for live tracking to work when app is backgrounded.
Use `expo-location` with background location task.

```typescript
import * as Location from 'expo-location'
import * as TaskManager from 'expo-task-manager'

const LOCATION_TASK = 'darija-location-task'

TaskManager.defineTask(LOCATION_TASK, async ({ data, error }) => {
  if (error) return
  const { locations } = data as { locations: Location.LocationObject[] }
  const { latitude, longitude } = locations[0].coords
  // Emit to socket — socket connection must be maintained in background
  socket.emit('location:update', { lat: latitude, lng: longitude })
})

// Start when delivery begins, stop when delivered
await Location.startLocationUpdatesAsync(LOCATION_TASK, {
  accuracy: Location.Accuracy.Balanced,
  timeInterval: 5000,        // Every 5 seconds
  distanceInterval: 20,      // Or every 20 meters
  foregroundService: {
    notificationTitle: 'Delivery in progress',
    notificationBody: 'Darija Market is tracking your delivery location',
  },
})
```

### 10.3 C Core Validation Middleware

Every POST request must pass through C core validation before reaching handlers.

```typescript
// apps/api/src/middleware/core-validation.ts
import net from 'net'

const CORE_SOCKET = '/tmp/core.sock'

export async function validateWithCore(
  sessionId: string,
  bodyHex: string
): Promise<boolean> {
  return new Promise((resolve) => {
    const client = net.createConnection(CORE_SOCKET, () => {
      client.write(`VALIDATE_SESSION ${sessionId} ${bodyHex}\n`)
    })
    client.on('data', (data) => {
      resolve(data.toString().trim() === 'OK')
      client.destroy()
    })
    client.on('error', () => resolve(false))
    client.setTimeout(100, () => {
      resolve(false)
      client.destroy()
    })
  })
}

// Fastify preHandler hook — applied to all POST routes
export async function coreValidationHook(request: FastifyRequest, reply: FastifyReply) {
  const sessionId = request.user.sessionId
  const bodyHex = Buffer.from(JSON.stringify(request.body)).toString('hex')
  const isValid = await validateWithCore(sessionId, bodyHex)
  if (!isValid) {
    reply.code(403).send({ success: false, error: { code: 'INVALID_REQUEST', message: 'Request validation failed' } })
  }
}
```

### 10.4 Push Notifications

Use Expo Notifications for Android (FCM under the hood).
Send via server when socket delivery not possible (app backgrounded/killed).

```typescript
// Server: send push when unable to reach via socket
import Expo from 'expo-server-sdk'
const expo = new Expo()

async function sendPushNotification(
  fcmToken: string,
  title: string,
  body: string,
  data: object
) {
  if (!Expo.isExpoPushToken(fcmToken)) return
  await expo.sendPushNotificationsAsync([{
    to: fcmToken,
    title,
    body,
    data,
    sound: 'default',
    priority: 'high',
  }])
}
```

### 10.5 Geospatial Queries (Nearby Sellers / Providers)

Use PostGIS extension on PostgreSQL for efficient radius queries.
Add PostGIS to the database and use raw queries where Prisma does not support spatial.

```sql
-- Enable extension (run once)
CREATE EXTENSION IF NOT EXISTS postgis;

-- Add geometry columns
ALTER TABLE "SellerProfile" ADD COLUMN location GEOGRAPHY(POINT, 4326);
ALTER TABLE "DeliveryProfile" ADD COLUMN location GEOGRAPHY(POINT, 4326);

-- Update trigger to keep location column in sync with lat/lng
CREATE OR REPLACE FUNCTION sync_seller_location()
RETURNS TRIGGER AS $$
BEGIN
  NEW.location = ST_MakePoint(NEW.lng, NEW.lat)::geography;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER seller_location_sync
BEFORE INSERT OR UPDATE ON "SellerProfile"
FOR EACH ROW EXECUTE FUNCTION sync_seller_location();
```

```typescript
// Nearby sellers query
async function getNearbySellers(lat: number, lng: number, radiusMeters: number, category?: VerticalCategory) {
  return prisma.$queryRaw`
    SELECT sp.*, ST_Distance(sp.location, ST_MakePoint(${lng}, ${lat})::geography) as distance_meters
    FROM "SellerProfile" sp
    WHERE sp."isOpen" = true
    AND ST_DWithin(sp.location, ST_MakePoint(${lng}, ${lat})::geography, ${radiusMeters})
    ${category ? Prisma.sql`AND EXISTS (SELECT 1 FROM "Product" p WHERE p."sellerProfileId" = sp.id AND p.category = ${category}::"VerticalCategory" AND p."isAvailable" = true)` : Prisma.empty}
    ORDER BY distance_meters ASC
    LIMIT 50
  `
}
```

### 10.6 Delivery Fee Calculation

Flat rate by distance band. Server-calculated, never trust client.

```typescript
// packages/utils/src/delivery-fee.ts
export function calculateDeliveryFee(distanceMeters: number): number {
  if (distanceMeters <= 1000) return 20    // 1km
  if (distanceMeters <= 2000) return 25    // 2km
  if (distanceMeters <= 3000) return 30    // 3km
  if (distanceMeters <= 5000) return 40    // 5km
  if (distanceMeters <= 10000) return 55   // 10km
  return 70                                // 10km+
}
```

### 10.7 Commission Calculation

```typescript
// packages/utils/src/commission.ts
const COMMISSION_RATE = 0.01  // 1%

export function calculateCommission(subtotalMad: number): number {
  return Math.round(subtotalMad * COMMISSION_RATE * 100) / 100
}

export function calculateSettlement(subtotalMad: number, deliveryFeeMad: number) {
  const commission = calculateCommission(subtotalMad)
  return {
    sellerReceives: subtotalMad - commission,
    providerReceives: deliveryFeeMad,
    platformKeeps: commission,
  }
}
```

### 10.8 Deposit Tier Validation

```typescript
// packages/constants/src/deposit.ts
export const DEPOSIT_TIERS = {
  BASIC: {
    minDepositMad: 200,
    maxParcelValueMad: 300,
    label: 'Basic',
  },
  STANDARD: {
    minDepositMad: 500,
    maxParcelValueMad: 800,
    label: 'Standard',
  },
  PREMIUM: {
    minDepositMad: 1000,
    maxParcelValueMad: Infinity,
    label: 'Premium',
  },
} as const

export function canDeliverParcel(
  depositTier: DepositTier,
  parcelValueMad: number
): boolean {
  return parcelValueMad <= DEPOSIT_TIERS[depositTier].maxParcelValueMad
}
```

### 10.9 Dispute Window

```typescript
// packages/constants/src/dispute.ts
export const DISPUTE_WINDOW_HOURS = 2        // Customer can dispute within 2h of delivery
export const AUTO_CONFIRM_HOURS = 24         // Auto-confirm if no dispute after 24h
export const EVIDENCE_SUBMISSION_HOURS = 24  // Both parties have 24h to submit evidence
```

### 10.10 LiveKit In-App Calls

```typescript
// Server: generate LiveKit token
import { AccessToken } from 'livekit-server-sdk'

async function generateCallToken(orderId: string, userId: string): Promise<string> {
  const roomName = `order-${orderId}`
  const token = new AccessToken(
    process.env.LIVEKIT_API_KEY,
    process.env.LIVEKIT_API_SECRET,
    { identity: userId }
  )
  token.addGrant({ room: roomName, roomJoin: true, canPublish: true, canSubscribe: true })
  return token.toJwt()
}

// Client (React Native): join call
import { useRoom } from '@livekit/react-native'

const { connect, room } = useRoom()
await connect(process.env.LIVEKIT_URL, token)
```

---

## 11. ENVIRONMENT VARIABLES

### apps/api/.env

```bash
# Database
DATABASE_URL="postgresql://user:password@localhost:5432/darijamarket"

# Redis
REDIS_URL="redis://localhost:6379"

# Auth
JWT_ACCESS_SECRET="<64-char-random>"
JWT_REFRESH_SECRET="<64-char-random>"
JWT_ACCESS_EXPIRES="15m"
JWT_REFRESH_EXPIRES="30d"

# C Core
C_CORE_SOCKET="/tmp/core.sock"

# LiveKit
LIVEKIT_URL="wss://livekit.darijamarket.ma"
LIVEKIT_API_KEY="<livekit-key>"
LIVEKIT_API_SECRET="<livekit-secret>"

# Storage
STORAGE_TYPE="local"                          # or "s3"
STORAGE_LOCAL_PATH="./uploads"
STORAGE_BASE_URL="https://api.darijamarket.ma/uploads"

# SMS (OTP)
SMS_PROVIDER="twilio"                         # or any Moroccan SMS gateway
SMS_FROM="+212XXXXXXXXX"
TWILIO_ACCOUNT_SID="<sid>"
TWILIO_AUTH_TOKEN="<token>"

# Expo Push
EXPO_ACCESS_TOKEN="<expo-token>"

# Server
PORT=3000
NODE_ENV="production"
LOG_LEVEL="info"
```

### apps/seller/.env / apps/delivery/.env / apps/customer/.env

```bash
EXPO_PUBLIC_API_URL="https://api.darijamarket.ma/v1"
EXPO_PUBLIC_WS_URL="wss://api.darijamarket.ma"
EXPO_PUBLIC_LIVEKIT_URL="wss://livekit.darijamarket.ma"
EXPO_PUBLIC_GOOGLE_MAPS_API_KEY="<key>"
```

---

## 12. DEVELOPMENT SEQUENCE

Build in this order. Do not skip steps. Each step must be working before the next begins.

```
Step 1 — Foundation
  [ ] Monorepo setup (Turborepo + pnpm)
  [ ] packages/types scaffolded
  [ ] packages/utils scaffolded (fee calc, commission calc)
  [ ] packages/constants scaffolded (state machine, deposit tiers, timings)
  [ ] Database schema applied (prisma migrate dev)
  [ ] Prisma client generated, types exported from packages/types

Step 2 — API Core
  [ ] Fastify server boilerplate with error handling
  [ ] Auth routes (OTP request, OTP verify, refresh, logout)
  [ ] JWT middleware
  [ ] C core validation middleware (Unix socket bridge)
  [ ] File upload endpoint
  [ ] Socket.io server setup with room management

Step 3 — Order Flow (API)
  [ ] Seller: create/update store, toggle open, manage products
  [ ] Customer: browse sellers, browse products
  [ ] Customer: create order
  [ ] Seller: view orders, accept/reject, mark ready, assign provider
  [ ] Delivery: job feed, accept/decline job
  [ ] Delivery: pickup confirm (photo + GPS)
  [ ] Delivery: delivery confirm (photo + GPS)
  [ ] Customer: confirm delivery or dispute
  [ ] Auto-confirm scheduler (24h cron)
  [ ] All state transitions emit Socket.io events

Step 4 — Seller App
  [ ] Expo project, navigation (Expo Router)
  [ ] Auth screens
  [ ] Store management
  [ ] Product management
  [ ] Orders list and order detail
  [ ] Provider selection for order assignment
  [ ] In-app chat
  [ ] Basic ERP dashboard

Step 5 — Delivery App
  [ ] Expo project, navigation
  [ ] Auth screens + deposit setup
  [ ] Online/offline toggle
  [ ] Job feed with customer reliability score visible
  [ ] Active delivery flow with camera + GPS confirms
  [ ] Background location task
  [ ] In-app chat
  [ ] Earnings screen

Step 6 — Customer App
  [ ] Expo project, navigation
  [ ] Auth (phone only, optional account)
  [ ] Food UI browse
  [ ] Order placement with map address pin
  [ ] Live tracking screen
  [ ] Delivery confirm / dispute
  [ ] Remaining vertical UIs (Clothing, Tech, Transport, Home, General)

Step 7 — Customer PWA
  [ ] Next.js project
  [ ] Feature parity with Customer App for core order flow
  [ ] Optimized for WhatsApp link sharing

Step 8 — ERP (Seller App)
  [ ] Inventory management screens
  [ ] Customer (CRM) screens
  [ ] Invoice generation (server-side PDF)
  [ ] Trade Workshop module builder

Step 9 — Hardening
  [ ] Dispute flow end-to-end
  [ ] Auto-confirm cron
  [ ] Deposit management
  [ ] Push notification delivery for all events
  [ ] Rate limiting on all routes
  [ ] Input validation (Zod schemas on all routes)
  [ ] Comprehensive error handling
```

---

## 13. CONSTRAINTS AND NON-NEGOTIABLES

```
1. Never process payment. COD only. Platform never holds customer cash.

2. Never label transport as taxi, ride-hailing, or any regulated category
   anywhere in code, UI strings, API responses, or database values.

3. The C core must validate every POST before business logic executes.
   No exceptions. No bypass in development mode.

4. Photo is required to transition PICKED_UP and DELIVERED states.
   The API must reject these transitions without a valid photoUrl.

5. GPS coordinates are required for PICKED_UP, DELIVERED state transitions.
   The API must reject these transitions without lat/lng.

6. A frozen deposit must not release without explicit dispute resolution
   or a court order reference in the DisputeTransaction record.

7. All inter-party communication goes through platform messaging.
   Never pass raw phone numbers between parties automatically.

8. Customer reliability score must be visible to delivery providers
   on the job feed card before they accept. Non-negotiable UX requirement.

9. Commission is always calculated server-side.
   Never trust a commission value from the client.

10. Order state transitions not listed in Section 8 must return 400.
    No soft-fail on invalid transitions.
```

---

## 14. TESTING REQUIREMENTS

```
packages/utils:         100% unit test coverage (pure functions)
API state machine:      Integration test every valid transition
API state machine:      Integration test every invalid transition → expect 400
Upload + confirm flow:  E2E test: photo required = photo missing → expect 400
C core bridge:          Unit test: REJECT response blocks handler execution
Socket events:          Integration test: status change emits to correct rooms only
```

---

## 15. DEV MODE

### 15.1 Overview

Dev mode is activated by a single environment variable: `DEV_MODE=true`
When active, the following mocks and shortcuts engage automatically.
Dev mode MUST be impossible to activate in production — the API server
hard-crashes on startup if `NODE_ENV=production` and `DEV_MODE=true`.

```typescript
// apps/api/src/lib/dev-mode.ts
// This file is the single place that checks DEV_MODE.
// All other dev shortcuts import from here. Never check process.env.DEV_MODE elsewhere.

export const DEV = process.env.DEV_MODE === 'true'

if (DEV && process.env.NODE_ENV === 'production') {
  console.error('FATAL: DEV_MODE=true is not allowed in production')
  process.exit(1)
}

if (DEV) {
  console.warn('⚠️  DEV MODE ACTIVE — all security bypasses enabled — never use in production')
}
```

---

### 15.2 C Core Mock

In dev mode the C core Unix socket is replaced with a mock that always returns OK.
This allows the API to run without the C binary compiled and running.

```typescript
// apps/api/src/lib/core-bridge.ts

import { DEV } from './dev-mode'
import net from 'net'

export async function validateWithCore(
  sessionId: string,
  bodyHex: string
): Promise<boolean> {
  if (DEV) {
    // Mock: always approve in dev mode
    return true
  }
  // Production: real Unix socket call to C core
  return new Promise((resolve) => {
    const client = net.createConnection(process.env.C_CORE_SOCKET!, () => {
      client.write(`VALIDATE_SESSION ${sessionId} ${bodyHex}\n`)
    })
    client.on('data', (data) => {
      resolve(data.toString().trim() === 'OK')
      client.destroy()
    })
    client.on('error', () => resolve(false))
    client.setTimeout(100, () => { resolve(false); client.destroy() })
  })
}

export async function createCoreSession(userId: string): Promise<string> {
  if (DEV) {
    // Mock: return a predictable dev session token
    return `dev-session-${userId}`
  }
  return new Promise((resolve, reject) => {
    const client = net.createConnection(process.env.C_CORE_SOCKET!, () => {
      client.write(`CREATE_SESSION ${userId}\n`)
    })
    client.on('data', (data) => {
      resolve(data.toString().trim())
      client.destroy()
    })
    client.on('error', reject)
  })
}

export async function destroyCoreSession(sessionId: string): Promise<void> {
  if (DEV) return
  return new Promise((resolve, reject) => {
    const client = net.createConnection(process.env.C_CORE_SOCKET!, () => {
      client.write(`DESTROY_SESSION ${sessionId}\n`)
    })
    client.on('data', () => { resolve(); client.destroy() })
    client.on('error', reject)
  })
}
```

---

### 15.3 OTP Bypass

In dev mode, OTP verification always succeeds with the code `000000`.
No SMS is sent. No external service is called.

```typescript
// apps/api/src/services/auth.service.ts

import { DEV } from '../lib/dev-mode'

export async function requestOtp(phone: string): Promise<void> {
  if (DEV) {
    console.log(`[DEV] OTP for ${phone} is 000000`)
    return  // Skip SMS send
  }
  await smsProvider.send(phone, generateOtp())
}

export async function verifyOtp(phone: string, code: string): Promise<boolean> {
  if (DEV) {
    return code === '000000'  // Any phone, magic code always works
  }
  return await otpStore.verify(phone, code)
}
```

---

### 15.4 GPS Mock

Mobile apps in dev mode use hardcoded GPS coordinates instead of the device GPS.
Useful for testing location-dependent features (nearby sellers, job feed, delivery tracking)
on an emulator or a device that is not physically in Morocco.

```typescript
// packages/utils/src/dev-location.ts

export const DEV_LOCATIONS = {
  SELLER_1:   { lat: 33.5731, lng: -7.5898 },   // Casablanca centre
  SELLER_2:   { lat: 33.5892, lng: -7.6031 },   // Hay Hassani
  CUSTOMER_1: { lat: 33.5765, lng: -7.5943 },   // 500m from SELLER_1
  PROVIDER_1: { lat: 33.5748, lng: -7.5920 },   // Between seller and customer
} as const

// In dev mode, replace expo-location calls with this
export function getDevLocation(role: 'SELLER_1' | 'SELLER_2' | 'CUSTOMER_1' | 'PROVIDER_1') {
  return DEV_LOCATIONS[role]
}
```

```typescript
// Usage in mobile apps — wrap every expo-location call like this
import { DEV } from '@darija-market/utils'
import { getDevLocation } from '@darija-market/utils/dev-location'
import * as Location from 'expo-location'

export async function getCurrentLocation() {
  if (DEV) {
    // Change the role key to test different proximity scenarios
    return getDevLocation('PROVIDER_1')
  }
  const loc = await Location.getCurrentPositionAsync({ accuracy: Location.Accuracy.High })
  return { lat: loc.coords.latitude, lng: loc.coords.longitude }
}
```

---

### 15.5 Photo Upload Mock

In dev mode, camera capture and file upload are replaced with a fixed placeholder image.
Allows the full delivery confirmation flow to be tested without a physical camera.

```typescript
// apps/delivery/src/lib/photo-upload.ts

import { DEV } from '@darija-market/utils'

export async function captureAndUploadPhoto(): Promise<string> {
  if (DEV) {
    // Skip camera, return a placeholder URL the server will accept
    return 'https://api.darijamarket.ma/dev/placeholder.jpg'
  }
  const photo = await Camera.takePictureAsync({ quality: 0.7 })
  const { url } = await uploadToServer(photo.uri)
  return url
}
```

The API server must accept `https://api.darijamarket.ma/dev/placeholder.jpg`
as a valid photoUrl in DEV mode, and reject it in production.

```typescript
// apps/api/src/validators/order.validators.ts

import { DEV } from '../lib/dev-mode'

export function validatePhotoUrl(url: string): boolean {
  if (DEV && url === 'https://api.darijamarket.ma/dev/placeholder.jpg') return true
  return url.startsWith(process.env.STORAGE_BASE_URL!)
}
```

---

### 15.6 Seed Data

Run `pnpm seed` from `apps/api` to populate the database with a full working scenario.
This creates all three user types, active products, and orders in various states —
so every screen in every app has something to render immediately.

```typescript
// apps/api/prisma/seed.ts
// Run with: pnpm --filter api exec ts-node prisma/seed.ts

import { PrismaClient, OrderStatus } from '@prisma/client'
const prisma = new PrismaClient()

async function main() {
  console.log('Seeding dev database...')

  // ── USERS ──────────────────────────────────────────────────────────────────

  const sellerUser = await prisma.user.upsert({
    where: { phone: '+212600000001' },
    update: {},
    create: {
      phone: '+212600000001',
      phoneVerified: true,
      role: 'SELLER',
      name: 'Hassan Bouazza',
    }
  })

  const deliveryUser = await prisma.user.upsert({
    where: { phone: '+212600000002' },
    update: {},
    create: {
      phone: '+212600000002',
      phoneVerified: true,
      role: 'DELIVERY',
      name: 'Youssef Alami',
    }
  })

  const customerUser = await prisma.user.upsert({
    where: { phone: '+212600000003' },
    update: {},
    create: {
      phone: '+212600000003',
      phoneVerified: true,
      role: 'CUSTOMER',
      name: 'Fatima Zahra',
    }
  })

  // ── SELLER PROFILE + PRODUCTS ─────────────────────────────────────────────

  const seller = await prisma.sellerProfile.upsert({
    where: { userId: sellerUser.id },
    update: {},
    create: {
      userId: sellerUser.id,
      storeName: 'Hassan Snack & Msemen',
      description: 'Msemen, harira, sandwichs. Livraison rapide.',
      isOpen: true,
      lat: 33.5731,
      lng: -7.5898,
      address: 'Derb Ghallef, près de la grande mosquée',
      deliveryRadius: 3000,
      ratingAvg: 4.8,
      totalOrders: 142,
    }
  })

  const product1 = await prisma.product.create({
    data: {
      sellerProfileId: seller.id,
      name: 'Msemen x6',
      description: 'Msemen fait maison, livré chaud',
      category: 'FOOD',
      serviceType: 'PRODUCT',
      priceMad: 15,
      isAvailable: true,
      prepTimeMinutes: 10,
    }
  })

  const product2 = await prisma.product.create({
    data: {
      sellerProfileId: seller.id,
      name: 'Harira + Chebakia',
      description: 'Harira maison + 3 chebakia',
      category: 'FOOD',
      serviceType: 'PRODUCT',
      priceMad: 25,
      isAvailable: true,
      prepTimeMinutes: 5,
    }
  })

  // ── DELIVERY PROFILE ──────────────────────────────────────────────────────

  const provider = await prisma.deliveryProfile.upsert({
    where: { userId: deliveryUser.id },
    update: {},
    create: {
      userId: deliveryUser.id,
      isActive: true,
      currentLat: 33.5748,
      currentLng: -7.5920,
      vehicleType: 'motorcycle',
      depositTier: 'STANDARD',
      depositAmountMad: 500,
      depositStatus: 'ACTIVE',
      ratingAvg: 4.9,
      acceptanceRate: 0.92,
      completionRate: 0.98,
      onTimeRate: 0.89,
      totalDeliveries: 287,
    }
  })

  // ── CUSTOMER PROFILE ──────────────────────────────────────────────────────

  const customer = await prisma.customerProfile.upsert({
    where: { userId: customerUser.id },
    update: {},
    create: {
      userId: customerUser.id,
      completionRate: 0.97,
      cashAvailabilityRate: 1.0,
      totalOrders: 23,
    }
  })

  await prisma.savedAddress.create({
    data: {
      customerProfileId: customer.id,
      label: 'Maison',
      lat: 33.5765,
      lng: -7.5943,
      description: 'Immeuble Atlas, 3ème étage, sonnette 7',
      isDefault: true,
    }
  })

  // ── ORDERS IN VARIOUS STATES ──────────────────────────────────────────────

  // Order 1: PENDING — shows up in seller's incoming queue
  await prisma.order.create({
    data: {
      sellerProfileId: seller.id,
      customerProfileId: customer.id,
      status: 'PENDING',
      subtotalMad: 40,
      deliveryFeeMad: 25,
      commissionMad: 0.40,
      totalMad: 65,
      deliveryLat: 33.5765,
      deliveryLng: -7.5943,
      deliveryDescription: 'Immeuble Atlas, 3ème étage, sonnette 7',
      items: {
        create: [
          { productId: product1.id, quantity: 1, unitPrice: 15 },
          { productId: product2.id, quantity: 1, unitPrice: 25 },
        ]
      },
      statusHistory: { create: [{ status: 'PENDING' }] }
    }
  })

  // Order 2: IN_TRANSIT — shows live tracking on customer app
  await prisma.order.create({
    data: {
      sellerProfileId: seller.id,
      deliveryProfileId: provider.id,
      customerProfileId: customer.id,
      status: 'IN_TRANSIT',
      subtotalMad: 15,
      deliveryFeeMad: 20,
      commissionMad: 0.15,
      totalMad: 35,
      deliveryLat: 33.5765,
      deliveryLng: -7.5943,
      deliveryDescription: 'Immeuble Atlas, 3ème étage, sonnette 7',
      pickupPhotoUrl: 'https://api.darijamarket.ma/dev/placeholder.jpg',
      pickupLat: 33.5731,
      pickupLng: -7.5898,
      pickupConfirmedAt: new Date(),
      items: {
        create: [{ productId: product1.id, quantity: 1, unitPrice: 15 }]
      },
      statusHistory: {
        create: [
          { status: 'PENDING' },
          { status: 'ACCEPTED' },
          { status: 'PROVIDER_ASSIGNED' },
          { status: 'PROVIDER_ACCEPTED' },
          { status: 'AT_SELLER' },
          { status: 'PICKED_UP' },
          { status: 'IN_TRANSIT' },
        ]
      }
    }
  })

  // Order 3: DELIVERED — awaiting customer confirmation
  const deliveredOrder = await prisma.order.create({
    data: {
      sellerProfileId: seller.id,
      deliveryProfileId: provider.id,
      customerProfileId: customer.id,
      status: 'DELIVERED',
      subtotalMad: 25,
      deliveryFeeMad: 20,
      commissionMad: 0.25,
      totalMad: 45,
      deliveryLat: 33.5765,
      deliveryLng: -7.5943,
      deliveryDescription: 'Immeuble Atlas, 3ème étage, sonnette 7',
      pickupPhotoUrl: 'https://api.darijamarket.ma/dev/placeholder.jpg',
      pickupLat: 33.5731,
      pickupLng: -7.5898,
      pickupConfirmedAt: new Date(Date.now() - 30 * 60 * 1000),
      deliveryPhotoUrl: 'https://api.darijamarket.ma/dev/placeholder.jpg',
      deliveryLat: 33.5765,
      deliveryLng: -7.5943,
      deliveryConfirmedAt: new Date(Date.now() - 5 * 60 * 1000),
      items: {
        create: [{ productId: product2.id, quantity: 1, unitPrice: 25 }]
      },
      statusHistory: {
        create: [
          { status: 'PENDING' },
          { status: 'ACCEPTED' },
          { status: 'PROVIDER_ASSIGNED' },
          { status: 'PROVIDER_ACCEPTED' },
          { status: 'AT_SELLER' },
          { status: 'PICKED_UP' },
          { status: 'IN_TRANSIT' },
          { status: 'AT_CUSTOMER' },
          { status: 'DELIVERED' },
        ]
      }
    }
  })

  // Order 4: CONFIRMED — completed, visible in history
  const confirmedOrder = await prisma.order.create({
    data: {
      sellerProfileId: seller.id,
      deliveryProfileId: provider.id,
      customerProfileId: customer.id,
      status: 'CONFIRMED',
      subtotalMad: 30,
      deliveryFeeMad: 25,
      commissionMad: 0.30,
      totalMad: 55,
      deliveryLat: 33.5765,
      deliveryLng: -7.5943,
      deliveryDescription: 'Immeuble Atlas, 3ème étage, sonnette 7',
      pickupPhotoUrl: 'https://api.darijamarket.ma/dev/placeholder.jpg',
      pickupLat: 33.5731,
      pickupLng: -7.5898,
      pickupConfirmedAt: new Date(Date.now() - 2 * 60 * 60 * 1000),
      deliveryPhotoUrl: 'https://api.darijamarket.ma/dev/placeholder.jpg',
      deliveryLat: 33.5765,
      deliveryLng: -7.5943,
      deliveryConfirmedAt: new Date(Date.now() - 90 * 60 * 1000),
      items: {
        create: [
          { productId: product1.id, quantity: 2, unitPrice: 15 },
        ]
      },
      statusHistory: {
        create: [
          { status: 'PENDING' },
          { status: 'ACCEPTED' },
          { status: 'PROVIDER_ASSIGNED' },
          { status: 'PROVIDER_ACCEPTED' },
          { status: 'AT_SELLER' },
          { status: 'PICKED_UP' },
          { status: 'IN_TRANSIT' },
          { status: 'AT_CUSTOMER' },
          { status: 'DELIVERED' },
          { status: 'CONFIRMED' },
        ]
      }
    }
  })

  await prisma.settlement.create({
    data: {
      orderId: confirmedOrder.id,
      status: 'COMPLETED',
      subtotalMad: 30,
      deliveryFeeMad: 25,
      commissionMad: 0.30,
      sellerReceivesMad: 29.70,
      providerReceivesMad: 25,
      settledAt: new Date(),
    }
  })

  // Order 5: DISPUTED — shows dispute flow
  const disputedOrder = await prisma.order.create({
    data: {
      sellerProfileId: seller.id,
      deliveryProfileId: provider.id,
      customerProfileId: customer.id,
      status: 'DISPUTED',
      subtotalMad: 50,
      deliveryFeeMad: 30,
      commissionMad: 0.50,
      totalMad: 80,
      deliveryLat: 33.5765,
      deliveryLng: -7.5943,
      deliveryDescription: 'Immeuble Atlas, 3ème étage, sonnette 7',
      pickupPhotoUrl: 'https://api.darijamarket.ma/dev/placeholder.jpg',
      pickupLat: 33.5731,
      pickupLng: -7.5898,
      pickupConfirmedAt: new Date(Date.now() - 3 * 60 * 60 * 1000),
      deliveryPhotoUrl: 'https://api.darijamarket.ma/dev/placeholder.jpg',
      deliveryLat: 33.5765,
      deliveryLng: -7.5943,
      deliveryConfirmedAt: new Date(Date.now() - 2 * 60 * 60 * 1000),
      items: {
        create: [{ productId: product2.id, quantity: 2, unitPrice: 25 }]
      },
      statusHistory: {
        create: [
          { status: 'DELIVERED' },
          { status: 'DISPUTED' },
        ]
      }
    }
  })

  await prisma.dispute.create({
    data: {
      orderId: disputedOrder.id,
      status: 'EVIDENCE_REVIEW',
      initiatedBy: customerUser.id,
      reason: 'Commande incomplète — il manquait une chebakia',
    }
  })

  console.log('✅ Seed complete')
  console.log('')
  console.log('Dev credentials (all use OTP 000000):')
  console.log('  Seller:   +212600000001')
  console.log('  Delivery: +212600000002')
  console.log('  Customer: +212600000003')
}

main()
  .catch((e) => { console.error(e); process.exit(1) })
  .finally(() => prisma.$disconnect())
```

---

### 15.7 Dev Mode Environment File

```bash
# apps/api/.env.development
# Copy this to .env to run in dev mode

DEV_MODE=true
NODE_ENV=development
DATABASE_URL="postgresql://postgres:postgres@localhost:5432/darijamarket_dev"
REDIS_URL="redis://localhost:6379"
JWT_ACCESS_SECRET="dev-access-secret-not-for-production"
JWT_REFRESH_SECRET="dev-refresh-secret-not-for-production"
JWT_ACCESS_EXPIRES="7d"       # Longer in dev so tokens don't expire during testing
JWT_REFRESH_EXPIRES="90d"
C_CORE_SOCKET="/tmp/core.sock"   # Not used in DEV_MODE but must be set
STORAGE_TYPE="local"
STORAGE_LOCAL_PATH="./uploads"
STORAGE_BASE_URL="http://localhost:3000/uploads"
PORT=3000
LOG_LEVEL="debug"
```

```bash
# Mobile apps — apps/seller/.env.development (same for delivery, customer)
EXPO_PUBLIC_API_URL="http://localhost:3000/v1"
EXPO_PUBLIC_WS_URL="ws://localhost:3000"
EXPO_PUBLIC_DEV_MODE="true"
EXPO_PUBLIC_LIVEKIT_URL="ws://localhost:7880"
```

---

### 15.8 Dev Mode Summary Table

| Feature | Dev Mode Behaviour | Production Behaviour |
|---|---|---|
| C core validation | Always returns OK (mock) | Unix socket to C binary |
| OTP code | `000000` always works | Real SMS via provider |
| GPS location | Hardcoded Casablanca coordinates | Device GPS |
| Camera + photo | Returns placeholder URL | Physical camera |
| JWT expiry | 7 days access / 90 days refresh | 15min access / 30 days refresh |
| SMS send | Logged to console only | Sent via Twilio / SMS provider |
| Placeholder photo URL | Accepted as valid | Rejected (403) |
| DEV_MODE in production | Server crashes on startup | N/A |

---

## 16. FULL CODE MAP

Every file that will exist in the finished system.
Files marked `[G]` are generated (do not edit manually).
Files marked `[C]` are configuration (edit once, rarely change).
Files marked `[CORE]` are the most important logic files.
All other files are implementation files — create and fill these.

```
darija-market/
│
├── turbo.json                          [C]  Turborepo pipeline: build, dev, lint, type-check
├── package.json                        [C]  Root workspace: pnpm workspaces declaration
├── pnpm-workspace.yaml                 [C]  Workspace glob: apps/*, packages/*
├── tsconfig.base.json                  [C]  Base TypeScript config extended by all packages
├── .env.example                        [C]  Template env file — committed to repo
├── .gitignore                          [C]
├── SPEC.md                             [CORE]  This file — the source of truth
│
├── packages/
│   │
│   ├── types/
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   └── src/
│   │       └── index.ts                [CORE]  All shared types — ApiResponse, AuthPayload,
│   │                                           public profiles, socket payloads, OrderWithDetails
│   │
│   ├── utils/
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   └── src/
│   │       ├── index.ts                        Re-exports all utils
│   │       ├── delivery-fee.ts         [CORE]  calculateDeliveryFee(distanceMeters) → MAD
│   │       ├── commission.ts           [CORE]  calculateCommission, calculateSettlement
│   │       ├── distance.ts                     haversineDistance(lat1,lng1,lat2,lng2) → meters
│   │       ├── format.ts                       formatMad, formatDate, formatPhone
│   │       ├── dev-location.ts                 DEV_LOCATIONS map + getDevLocation()
│   │       └── dev-mode.ts                     export const DEV (read once, used everywhere)
│   │
│   └── constants/
│       ├── package.json
│       ├── tsconfig.json
│       └── src/
│           ├── index.ts                        Re-exports all constants
│           ├── order-states.ts         [CORE]  VALID_TRANSITIONS map (state machine definition)
│           ├── deposit.ts              [CORE]  DEPOSIT_TIERS, canDeliverParcel()
│           ├── dispute.ts                      DISPUTE_WINDOW_HOURS, AUTO_CONFIRM_HOURS
│           ├── socket-events.ts        [CORE]  All socket event name strings as typed constants
│           └── vertical-config.ts             Display names, icons, filter options per vertical
│
├── apps/
│   │
│   ├── api/                            Fastify API server
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   ├── .env                        [C]  Not committed — copy from .env.development
│   │   ├── .env.development            [C]  Dev defaults (DEV_MODE=true)
│   │   │
│   │   ├── prisma/
│   │   │   ├── schema.prisma           [CORE]  Database schema — source of truth for data model
│   │   │   ├── migrations/             [G]  Generated by prisma migrate dev
│   │   │   └── seed.ts                 [CORE]  Full seed script — see Section 15.6
│   │   │
│   │   └── src/
│   │       ├── server.ts               [CORE]  Fastify instance, register plugins, start server
│   │       ├── app.ts                          App factory (testable without listen())
│   │       │
│   │       ├── lib/
│   │       │   ├── dev-mode.ts         [CORE]  DEV flag + production guard — see Section 15.1
│   │       │   ├── core-bridge.ts      [CORE]  C core Unix socket bridge + dev mock — Section 15.2
│   │       │   ├── prisma.ts                   Prisma client singleton
│   │       │   ├── redis.ts                    ioredis client singleton
│   │       │   ├── socket.ts           [CORE]  Socket.io server setup, room management
│   │       │   ├── livekit.ts                  LiveKit server SDK wrapper, token generation
│   │       │   ├── storage.ts                  File upload: local disk or S3 (reads STORAGE_TYPE)
│   │       │   ├── sms.ts                      SMS send wrapper (Twilio) + dev console mock
│   │       │   ├── otp-store.ts                Redis-backed OTP storage (5min TTL)
│   │       │   └── scheduler.ts                Cron jobs: auto-confirm orders, score recalculation
│   │       │
│   │       ├── middleware/
│   │       │   ├── auth.ts             [CORE]  JWT verify → attach req.user (userId, role, sessionId)
│   │       │   ├── core-validation.ts  [CORE]  preHandler: proxy POST body through C core
│   │       │   ├── role-guard.ts               Verify req.user.role matches route requirement
│   │       │   └── error-handler.ts            Global Fastify error handler → ApiResponse format
│   │       │
│   │       ├── validators/
│   │       │   ├── auth.validators.ts          Zod schemas: RequestOtpBody, VerifyOtpBody
│   │       │   ├── order.validators.ts [CORE]  Zod schemas + validatePhotoUrl (dev/prod aware)
│   │       │   ├── product.validators.ts       Zod schemas: CreateProductBody, UpdateProductBody
│   │       │   ├── seller.validators.ts        Zod schemas: UpdateSellerProfileBody
│   │       │   ├── delivery.validators.ts      Zod schemas: ConfirmPickupBody, ConfirmDeliveryBody
│   │       │   └── customer.validators.ts      Zod schemas: CreateOrderBody, SaveAddressBody
│   │       │
│   │       ├── services/
│   │       │   ├── auth.service.ts     [CORE]  requestOtp, verifyOtp (dev bypass), issueTokens
│   │       │   ├── order.service.ts    [CORE]  All order state transitions — enforces state machine
│   │       │   ├── seller.service.ts           Seller profile CRUD, product CRUD, dashboard stats
│   │       │   ├── delivery.service.ts         Provider profile, job feed query, earnings calc
│   │       │   ├── customer.service.ts         Customer profile, nearby sellers query (PostGIS)
│   │       │   ├── dispute.service.ts  [CORE]  Dispute open, evidence submit, resolution
│   │       │   ├── settlement.service.ts       Create settlement records, mark complete
│   │       │   ├── score.service.ts            Recalculate behavioral scores after order events
│   │       │   ├── notification.service.ts     Push via Expo + socket fallback
│   │       │   ├── message.service.ts          Save message, emit socket event, notify recipient
│   │       │   ├── erp.service.ts              Inventory, ERP customers, invoice generation
│   │       │   └── deposit.service.ts  [CORE]  Deposit tier logic, freeze, release, validation
│   │       │
│   │       └── routes/
│   │           ├── auth.routes.ts              POST /auth/request-otp, /verify-otp, /refresh, /logout
│   │           ├── seller.routes.ts            All /seller/* routes — see Section 6.2
│   │           ├── delivery.routes.ts          All /delivery/* routes — see Section 6.3
│   │           ├── customer.routes.ts          All /customer/* routes — see Section 6.4
│   │           └── shared.routes.ts            /upload, /messages, /calls, /disputes, /verticals
│   │
│   ├── seller/                         React Native (Expo) — Seller app
│   │   ├── package.json
│   │   ├── app.json                    [C]  Expo config: appId, permissions, splash
│   │   ├── tsconfig.json
│   │   ├── .env                        [C]
│   │   │
│   │   └── src/
│   │       ├── app/                            Expo Router file-based navigation
│   │       │   ├── _layout.tsx                 Root layout: auth gate, socket init, notification setup
│   │       │   ├── (auth)/
│   │       │   │   ├── phone.tsx               Phone number entry screen
│   │       │   │   └── otp.tsx                 OTP entry screen (000000 in dev)
│   │       │   ├── (setup)/
│   │       │   │   └── profile.tsx             First-time store profile setup
│   │       │   └── (main)/
│   │       │       ├── _layout.tsx             Bottom tab navigator (Home, Orders, Products, ERP, Profile)
│   │       │       ├── index.tsx               Home: store toggle, active orders summary, quick stats
│   │       │       ├── orders/
│   │       │       │   ├── index.tsx           Orders list with status filter tabs
│   │       │       │   └── [id]/
│   │       │       │       ├── index.tsx       Order detail: items, status timeline, actions
│   │       │       │       └── assign.tsx      Provider selector screen
│   │       │       ├── products/
│   │       │       │   ├── index.tsx           Products list
│   │       │       │   └── [id]/
│   │       │       │       └── edit.tsx        Product create / edit form
│   │       │       ├── erp/
│   │       │       │   ├── index.tsx           ERP dashboard: revenue chart, top products
│   │       │       │   ├── inventory/
│   │       │       │   │   ├── index.tsx       Inventory list with stock levels
│   │       │       │   │   └── [id]/edit.tsx   Inventory item form
│   │       │       │   ├── customers/
│   │       │       │   │   ├── index.tsx       CRM customer list
│   │       │       │   │   └── [id]/index.tsx  Customer detail + order history
│   │       │       │   ├── invoices/
│   │       │       │   │   ├── index.tsx       Invoice list
│   │       │       │   │   └── new.tsx         Invoice creation form
│   │       │       │   └── workshop/
│   │       │       │       └── index.tsx       Trade Workshop graphical module builder
│   │       │       └── profile/
│   │       │           ├── index.tsx           Store profile edit
│   │       │           └── providers.tsx       Preferred providers list
│   │       │
│   │       ├── components/
│   │       │   ├── ui/                         Generic reusable components
│   │       │   │   ├── Button.tsx
│   │       │   │   ├── Card.tsx
│   │       │   │   ├── Badge.tsx               Deposit tier badge, rating badge
│   │       │   │   ├── Avatar.tsx
│   │       │   │   ├── Input.tsx
│   │       │   │   ├── Sheet.tsx               Bottom sheet wrapper
│   │       │   │   ├── StatusBar.tsx           Custom status bar
│   │       │   │   └── EmptyState.tsx
│   │       │   ├── order/
│   │       │   │   ├── OrderCard.tsx           Summary card for orders list
│   │       │   │   ├── OrderTimeline.tsx       Status history visualization
│   │       │   │   ├── OrderActions.tsx        Accept / reject / ready / assign buttons
│   │       │   │   └── ProviderCard.tsx        Provider info card with scores for selection
│   │       │   └── chat/
│   │       │       ├── ChatThread.tsx          Message list for an order
│   │       │       └── ChatInput.tsx           Message input + send
│   │       │
│   │       ├── hooks/
│   │       │   ├── useAuth.ts                  Auth state, login, logout
│   │       │   ├── useSocket.ts        [CORE]  Socket.io connection, join/leave order rooms
│   │       │   ├── useOrders.ts                React Query: fetch orders, mutations
│   │       │   ├── useProducts.ts              React Query: fetch/mutate products
│   │       │   └── useNotifications.ts         Expo push notification setup + handlers
│   │       │
│   │       ├── store/
│   │       │   ├── auth.store.ts               Zustand: user, tokens, profile
│   │       │   └── socket.store.ts             Zustand: socket instance, connection state
│   │       │
│   │       └── lib/
│   │           ├── api.ts                      Axios instance with base URL + auth header
│   │           ├── socket.ts                   Socket.io client init + reconnect logic
│   │           └── photo-upload.ts             Camera capture + upload (dev mock in DEV mode)
│   │
│   ├── delivery/                       React Native (Expo) — Delivery provider app
│   │   ├── package.json
│   │   ├── app.json                    [C]  Extra permissions: FOREGROUND_SERVICE, BACKGROUND_LOCATION
│   │   ├── tsconfig.json
│   │   ├── .env                        [C]
│   │   │
│   │   └── src/
│   │       ├── app/
│   │       │   ├── _layout.tsx                 Root layout: auth gate, socket, background location init
│   │       │   ├── (auth)/
│   │       │   │   ├── phone.tsx
│   │       │   │   └── otp.tsx
│   │       │   ├── (setup)/
│   │       │   │   ├── profile.tsx             Vehicle type, area setup
│   │       │   │   └── deposit.tsx             Explain deposit tiers, guide to CashPlus agent
│   │       │   └── (main)/
│   │       │       ├── _layout.tsx             Bottom tabs: Jobs, Active, History, Earnings, Profile
│   │       │       ├── index.tsx               Job feed: online toggle + available job list
│   │       │       ├── active/
│   │       │       │   └── index.tsx           Active delivery: map, confirm pickup, confirm delivery
│   │       │       ├── history/
│   │       │       │   ├── index.tsx           Past deliveries list
│   │       │       │   └── [id]/index.tsx      Delivery detail
│   │       │       ├── earnings/
│   │       │       │   └── index.tsx           Earnings summary, deposit status, dispute log
│   │       │       └── profile/
│   │       │           └── index.tsx           Provider profile, vehicle info, reputation scores
│   │       │
│   │       ├── components/
│   │       │   ├── ui/                         (same base components as seller app)
│   │       │   ├── jobs/
│   │       │   │   ├── JobCard.tsx     [CORE]  Job card with customer reliability score badge
│   │       │   │   └── JobFeed.tsx             Scrollable live job list
│   │       │   ├── delivery/
│   │       │   │   ├── DeliveryMap.tsx [CORE]  Map with route to current destination
│   │       │   │   ├── ConfirmPickup.tsx [CORE] Camera + GPS + submit (dev mock aware)
│   │       │   │   └── ConfirmDelivery.tsx [CORE]
│   │       │   └── chat/
│   │       │       ├── ChatThread.tsx
│   │       │       └── ChatInput.tsx
│   │       │
│   │       ├── hooks/
│   │       │   ├── useAuth.ts
│   │       │   ├── useSocket.ts
│   │       │   ├── useJobs.ts                  React Query: available jobs, accept/decline
│   │       │   ├── useActiveDelivery.ts        Active order state + actions
│   │       │   ├── useBackgroundLocation.ts    [CORE]  Start/stop background location task
│   │       │   └── useNotifications.ts
│   │       │
│   │       ├── store/
│   │       │   ├── auth.store.ts
│   │       │   ├── socket.store.ts
│   │       │   └── delivery.store.ts           Active order, online status
│   │       │
│   │       ├── tasks/
│   │       │   └── location-task.ts    [CORE]  TaskManager background location → socket emit
│   │       │
│   │       └── lib/
│   │           ├── api.ts
│   │           ├── socket.ts
│   │           └── photo-upload.ts             Same as seller, dev mock aware
│   │
│   ├── customer/                       React Native (Expo) — Customer Android app
│   │   ├── package.json
│   │   ├── app.json                    [C]
│   │   ├── tsconfig.json
│   │   ├── .env                        [C]
│   │   │
│   │   └── src/
│   │       ├── app/
│   │       │   ├── _layout.tsx
│   │       │   ├── (auth)/
│   │       │   │   ├── phone.tsx               Phone entry (optional — skip to browse allowed)
│   │       │   │   └── otp.tsx
│   │       │   └── (main)/
│   │       │       ├── _layout.tsx             Bottom tabs: Discover, Orders, Profile
│   │       │       ├── index.tsx               Vertical selector (Food, Clothing, Tech, etc.)
│   │       │       ├── food/
│   │       │       │   ├── index.tsx           Food browse UI — visual, filter by cuisine/time
│   │       │       │   └── [sellerId]/
│   │       │       │       ├── index.tsx       Seller food menu
│   │       │       │       └── [productId].tsx Product detail + add to cart
│   │       │       ├── clothing/
│   │       │       │   ├── index.tsx           Clothing browse — grid, size filter
│   │       │       │   └── [sellerId]/index.tsx
│   │       │       ├── tech/
│   │       │       │   ├── index.tsx           Tech browse — spec-forward, search
│   │       │       │   └── [sellerId]/index.tsx
│   │       │       ├── transport/
│   │       │       │   └── index.tsx           Transport booking: origin pin, dest pin, vehicle type
│   │       │       ├── home/
│   │       │       │   ├── index.tsx           Home improvement — services + products
│   │       │       │   └── [sellerId]/index.tsx
│   │       │       ├── general/
│   │       │       │   ├── index.tsx           General marketplace grid
│   │       │       │   └── [sellerId]/index.tsx
│   │       │       ├── cart/
│   │       │       │   ├── index.tsx           Cart summary
│   │       │       │   └── checkout.tsx        Address selector + order confirmation
│   │       │       ├── orders/
│   │       │       │   ├── index.tsx           Order history
│   │       │       │   └── [id]/
│   │       │       │       ├── index.tsx       Active order: live map, status, chat, confirm/dispute
│   │       │       │       └── dispute.tsx     Dispute initiation flow
│   │       │       └── profile/
│   │       │           ├── index.tsx           Profile, reliability score, settings
│   │       │           └── addresses.tsx       Saved addresses manager
│   │       │
│   │       ├── components/
│   │       │   ├── ui/
│   │       │   ├── browse/
│   │       │   │   ├── SellerCard.tsx          Seller summary card for listing screens
│   │       │   │   ├── ProductCard.tsx         Product card: photo, price, availability
│   │       │   │   ├── FoodSellerCard.tsx      Food-specific: cuisine, prep time, min order
│   │       │   │   └── ClothingProductCard.tsx Grid card with size badges
│   │       │   ├── order/
│   │       │   │   ├── LiveTrackingMap.tsx [CORE]  Map with provider live location
│   │       │   │   ├── OrderTimeline.tsx
│   │       │   │   └── DeliveryConfirm.tsx     Confirm received / open dispute
│   │       │   ├── address/
│   │       │   │   ├── AddressPicker.tsx [CORE] Map pin + description text input
│   │       │   │   └── SavedAddressList.tsx
│   │       │   └── chat/
│   │       │       ├── ChatThread.tsx
│   │       │       └── ChatInput.tsx
│   │       │
│   │       ├── hooks/
│   │       │   ├── useAuth.ts
│   │       │   ├── useSocket.ts
│   │       │   ├── useSellers.ts               React Query: nearby sellers, search
│   │       │   ├── useCart.ts                  Zustand cart state + cart mutations
│   │       │   ├── useOrders.ts
│   │       │   └── useLocation.ts              Get current location (dev mock aware)
│   │       │
│   │       ├── store/
│   │       │   ├── auth.store.ts
│   │       │   ├── socket.store.ts
│   │       │   └── cart.store.ts               Items, seller lock, clear cart
│   │       │
│   │       └── lib/
│   │           ├── api.ts
│   │           └── socket.ts
│   │
│   ├── customer-web/                   Next.js 14 PWA — Customer web app
│   │   ├── package.json
│   │   ├── next.config.js              [C]  next-pwa config, service worker, manifest
│   │   ├── tsconfig.json
│   │   ├── .env                        [C]
│   │   ├── public/
│   │   │   ├── manifest.json           [C]  PWA manifest: name, icons, theme color
│   │   │   └── icons/                       App icons for PWA install
│   │   │
│   │   └── src/
│   │       ├── app/                            Next.js App Router
│   │       │   ├── layout.tsx                  Root layout: fonts, providers
│   │       │   ├── page.tsx                    Vertical selector landing
│   │       │   ├── food/page.tsx               Food browse
│   │       │   ├── clothing/page.tsx
│   │       │   ├── tech/page.tsx
│   │       │   ├── transport/page.tsx
│   │       │   ├── home/page.tsx
│   │       │   ├── general/page.tsx
│   │       │   ├── seller/[id]/page.tsx        Seller detail + product list
│   │       │   ├── cart/page.tsx
│   │       │   ├── checkout/page.tsx
│   │       │   ├── orders/page.tsx
│   │       │   ├── orders/[id]/page.tsx        Live tracking + confirm/dispute
│   │       │   └── auth/
│   │       │       ├── phone/page.tsx
│   │       │       └── otp/page.tsx
│   │       │
│   │       ├── components/             (PWA equivalents of customer native components)
│   │       │   ├── ui/
│   │       │   ├── browse/
│   │       │   ├── order/
│   │       │   │   └── LiveTrackingMap.tsx     Leaflet.js (no Google Maps dep on web)
│   │       │   └── address/
│   │       │       └── AddressPicker.tsx       Leaflet map pin picker
│   │       │
│   │       ├── hooks/                  (same interface as native hooks, different implementation)
│   │       │   ├── useAuth.ts
│   │       │   ├── useSocket.ts
│   │       │   ├── useSellers.ts
│   │       │   ├── useCart.ts
│   │       │   └── useOrders.ts
│   │       │
│   │       └── lib/
│   │           ├── api.ts
│   │           └── socket.ts
│   │
│   └── core-c/                         Existing C security backend
│       ├── README.md                   [C]  How to build and run the C binary
│       ├── Makefile                    [C]  Build instructions
│       └── src/                        Existing C source — do not modify
│           └── (existing files)
│
└── docs/
    ├── SPEC.md                         Symlink → SPEC.md at root
    ├── adr/                            Architecture Decision Records
    │   ├── 001-monorepo.md             Why Turborepo
    │   ├── 002-react-native-expo.md    Why Expo over bare RN
    │   ├── 003-prisma.md               Why Prisma over raw SQL
    │   ├── 004-c-core-bridge.md        How Fastify talks to C binary
    │   └── 005-no-payment-gateway.md   Why we never touch money
    └── flows/
        ├── order-flow.md               Full order lifecycle walkthrough
        ├── dispute-flow.md             Dispute resolution walkthrough
        └── deposit-flow.md             Deposit collection and release walkthrough
```

---

### 16.1 File Count Summary

```
packages/types:       1 core file (generated + hand-extended)
packages/utils:       7 files
packages/constants:   5 files

apps/api/src:
  lib:                9 files
  middleware:         4 files
  validators:         6 files
  services:           12 files       ← most business logic lives here
  routes:             5 files

apps/seller/src:
  screens:            18 screens
  components:         11 components
  hooks:              5 hooks
  store:              2 stores
  lib:                3 files

apps/delivery/src:
  screens:            11 screens
  components:         8 components
  hooks:              6 hooks
  store:              3 stores
  tasks:              1 background task
  lib:                3 files

apps/customer/src:
  screens:            22 screens
  components:         13 components
  hooks:              6 hooks
  store:              3 stores
  lib:                2 files

apps/customer-web/src:
  pages:              14 pages
  components:         ~12 components
  hooks:              5 hooks
  lib:                2 files

Total implementation files:  ~220 files
Total generated/config:      ~30 files
```

---

### 16.2 Files To Write First

In strict dependency order — each file depends only on files above it in this list.

```
1.  packages/constants/src/order-states.ts      VALID_TRANSITIONS map
2.  packages/constants/src/deposit.ts           DEPOSIT_TIERS, canDeliverParcel
3.  packages/constants/src/dispute.ts           Time window constants
4.  packages/constants/src/socket-events.ts     All event name strings
5.  packages/constants/src/vertical-config.ts   Vertical display config
6.  packages/utils/src/dev-mode.ts              DEV flag
7.  packages/utils/src/delivery-fee.ts          Pure function
8.  packages/utils/src/commission.ts            Pure functions
9.  packages/utils/src/distance.ts              Haversine pure function
10. packages/utils/src/dev-location.ts          Dev GPS coordinates
11. apps/api/prisma/schema.prisma               Full schema
12. [run: prisma migrate dev]                   [G] migrations
13. [run: prisma generate]                      [G] Prisma client
14. packages/types/src/index.ts                 Import from Prisma + hand-written types
15. apps/api/src/lib/dev-mode.ts                DEV flag + production guard
16. apps/api/src/lib/prisma.ts                  Client singleton
17. apps/api/src/lib/redis.ts                   Client singleton
18. apps/api/src/lib/core-bridge.ts             C core bridge + dev mock
19. apps/api/src/middleware/auth.ts             JWT verify
20. apps/api/src/middleware/core-validation.ts  preHandler hook
21. apps/api/src/services/auth.service.ts       OTP + token issue
22. apps/api/src/services/order.service.ts      State machine transitions
23. apps/api/src/routes/auth.routes.ts
24. apps/api/src/lib/socket.ts                  Socket.io server
25. apps/api/src/server.ts                      Wire everything together
26. apps/api/prisma/seed.ts                     Dev seed data
```

---

*End of specification. Last updated: 2026-03-13.*
*All decisions not covered here should be raised as questions before implementation, not resolved by assumption.*
