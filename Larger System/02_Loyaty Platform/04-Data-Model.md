# Loyalty Platform - Data Model

## Overview
Entity-Relationship model and database schema for the Loyalty Platform.

## Core Entities

### 1. User
```
User
├── id (PK)
├── email (UNIQUE)
├── first_name
├── last_name
├── phone
├── date_of_birth
├── registration_date
├── last_login_date
├── status (active/inactive/suspended)
├── created_at
└── updated_at
```

### 2. Loyalty Account
```
LoyaltyAccount
├── id (PK)
├── user_id (FK)
├── total_points_earned
├── current_points_balance
├── points_redeemed
├── tier_level (BRONZE/SILVER/GOLD/PLATINUM)
├── created_at
└── updated_at
```

### 3. Points Transaction
```
PointsTransaction
├── id (PK)
├── loyalty_account_id (FK)
├── transaction_type (earn/redeem/expire)
├── points_amount
├── description
├── reference_id
├── transaction_date
├── expiration_date (nullable)
└── created_at
```

### 4. Reward
```
Reward
├── id (PK)
├── name
├── description
├── points_required
├── category
├── image_url
├── status (active/inactive)
├── created_at
└── updated_at
```

### 5. Reward Inventory
```
RewardInventory
├── id (PK)
├── reward_id (FK)
├── total_quantity
├── available_quantity
├── reserved_quantity
├── created_at
└── updated_at
```

### 6. Redemption
```
Redemption
├── id (PK)
├── loyalty_account_id (FK)
├── reward_id (FK)
├── redemption_date
├── status (pending/completed/cancelled)
├── delivery_address
├── tracking_number (nullable)
└── created_at
```

### 7. Notification
```
Notification
├── id (PK)
├── user_id (FK)
├── type (email/sms/push)
├── subject
├── message
├── status (sent/failed/pending)
├── sent_date
└── created_at
```

## Relationships

- **User** (1) → (1) **LoyaltyAccount**
- **LoyaltyAccount** (1) → (N) **PointsTransaction**
- **LoyaltyAccount** (1) → (N) **Redemption**
- **Reward** (1) → (1) **RewardInventory**
- **Reward** (1) → (N) **Redemption**
- **User** (1) → (N) **Notification**

## Indexes

- User.email
- LoyaltyAccount.user_id
- PointsTransaction.loyalty_account_id, transaction_date
- Redemption.loyalty_account_id, status
- Notification.user_id, status

## Database Constraints

- Foreign key constraints on all FK relationships
- Unique constraint on User.email
- Check constraint on Points amounts (> 0)
- Check constraint on Reward cost (> 0)
