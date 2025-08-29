# Restaurant Management System (RestaurantOS)

A comprehensive restaurant management system with role-based access control for Administrators, Cashiers, and Kitchen Staff. Built with React frontend and Node.js/Express.js backend with Supabase database.

## 🏗️ Project Overview

This system manages the complete restaurant workflow from order placement to food preparation and delivery, with real-time inventory tracking and role-based permissions.

### Core Roles & Responsibilities

#### 1. **Administrator / Management**
- **Goal**: Oversee operations, manage data, and configure the system
- **Key Features**:
  - Account & access management with role-based permissions
  - Dashboard with analytics and key metrics
  - Inventory management with real-time stock monitoring
  - Menu management with ingredient dependencies
  - Order history and financial reports
  - Employee duty tracking and time management
  - System settings and security configuration

#### 2. **Kitchen Staff**
- **Goal**: Prepare orders based on real-time requests
- **Key Features**:
  - Real-time order ticket management
  - Order status updates (pending → preparing → ready)
  - Stock level awareness and alerts
  - Ingredient requirement validation
  - Order priority management

#### 3. **Cashier / Service Staff**
- **Goal**: Take and process customer orders and payments
- **Key Features**:
  - Order processing with menu selection
  - Payment handling (cash, GCash, card)
  - Real-time order status tracking
  - Customer management
  - Receipt generation and printing

## 🚀 Server Architecture Plan

### Technology Stack
- **Backend**: Node.js + Express.js
- **Database**: Supabase (PostgreSQL)
- **Authentication**: Supabase Auth + JWT
- **Real-time**: Supabase Realtime + WebSocket
- **File Storage**: Supabase Storage
- **API Documentation**: Swagger/OpenAPI

### Server Structure

```
server/
├── src/
│   ├── config/
│   │   ├── database.js          # Supabase configuration
│   │   ├── auth.js              # Authentication middleware
│   │   └── cors.js              # CORS configuration
│   ├── middleware/
│   │   ├── auth.js              # JWT verification
│   │   ├── roleCheck.js         # Role-based access control
│   │   ├── validation.js        # Request validation
│   │   └── errorHandler.js      # Global error handling
│   ├── routes/
│   │   ├── auth/
│   │   │   ├── login.js         # User authentication
│   │   │   ├── register.js      # User registration
│   │   │   └── refresh.js       # Token refresh
│   │   ├── admin/
│   │   │   ├── dashboard.js     # Admin dashboard data
│   │   │   ├── employees.js     # Employee management
│   │   │   ├── reports.js       # Sales and analytics
│   │   │   └── settings.js      # System configuration
│   │   ├── inventory/
│   │   │   ├── ingredients.js   # Ingredient CRUD
│   │   │   ├── stock.js         # Stock operations
│   │   │   └── alerts.js        # Low stock notifications
│   │   ├── menu/
│   │   │   ├── items.js         # Menu item CRUD
│   │   │   ├── categories.js    # Menu categories
│   │   │   └── availability.js  # Menu availability logic
│   │   ├── orders/
│   │   │   ├── create.js        # Order creation
│   │   │   ├── status.js        # Order status updates
│   │   │   ├── history.js       # Order history
│   │   │   └── notifications.js # Order notifications
│   │   ├── cashier/
│   │   │   ├── orders.js        # Cashier order operations
│   │   │   ├── payments.js      # Payment processing
│   │   │   └── customers.js     # Customer management
│   │   └── kitchen/
│   │       ├── orders.js        # Kitchen order management
│   │       ├── preparation.js   # Food preparation tracking
│   │       └── equipment.js     # Equipment status
│   ├── services/
│   │   ├── inventoryService.js  # Inventory business logic
│   │   ├── orderService.js      # Order processing logic
│   │   ├── notificationService.js # Real-time notifications
│   │   └── reportService.js     # Report generation
│   ├── models/
│   │   ├── User.js              # User data model
│   │   ├── Order.js             # Order data model
│   │   ├── MenuItem.js          # Menu item model
│   │   ├── Ingredient.js        # Ingredient model
│   │   └── Employee.js          # Employee model
│   ├── utils/
│   │   ├── database.js          # Database helpers
│   │   ├── validation.js        # Data validation helpers
│   │   ├── notifications.js     # Notification helpers
│   │   └── logger.js            # Logging utilities
│   └── app.js                   # Main application file
├── tests/                       # Test files
├── docs/                        # API documentation
├── package.json
├── .env.example
└── README.md
```

### Database Schema (Supabase)

#### Core Tables

```sql
-- Users and Authentication
users (
  id: uuid PRIMARY KEY,
  email: text UNIQUE NOT NULL,
  first_name: text NOT NULL,
  last_name: text NOT NULL,
  role: enum('admin', 'cashier', 'kitchen') NOT NULL,
  is_active: boolean DEFAULT true,
  created_at: timestamp DEFAULT now(),
  updated_at: timestamp DEFAULT now()
)

-- Employee Management
employees (
  id: uuid PRIMARY KEY,
  user_id: uuid REFERENCES users(id),
  employee_code: text UNIQUE NOT NULL,
  position: text NOT NULL,
  hire_date: date NOT NULL,
  salary: decimal(10,2),
  contact_number: text,
  emergency_contact: jsonb,
  created_at: timestamp DEFAULT now()
)

-- Time Tracking
time_logs (
  id: uuid PRIMARY KEY,
  employee_id: uuid REFERENCES employees(id),
  time_in: timestamp,
  time_out: timestamp,
  date: date NOT NULL,
  total_hours: decimal(4,2),
  status: enum('present', 'absent', 'late', 'overtime')
)

-- Ingredients/Inventory
ingredients (
  id: uuid PRIMARY KEY,
  name: text NOT NULL,
  description: text,
  category: text NOT NULL,
  unit: text NOT NULL,
  current_stock: decimal(10,3) DEFAULT 0,
  min_stock: decimal(10,3) DEFAULT 0,
  max_stock: decimal(10,3),
  cost_per_unit: decimal(10,2),
  supplier: text,
  expiry_date: date,
  status: enum('sufficient', 'low', 'out') DEFAULT 'sufficient',
  created_at: timestamp DEFAULT now(),
  updated_at: timestamp DEFAULT now()
)

-- Stock Operations
stock_operations (
  id: uuid PRIMARY KEY,
  ingredient_id: uuid REFERENCES ingredients(id),
  operation_type: enum('in', 'out', 'adjustment', 'spoilage'),
  quantity: decimal(10,3) NOT NULL,
  reason: text,
  performed_by: uuid REFERENCES users(id),
  created_at: timestamp DEFAULT now()
)

-- Menu Categories
menu_categories (
  id: uuid PRIMARY KEY,
  name: text NOT NULL,
  description: text,
  sort_order: integer DEFAULT 0,
  is_active: boolean DEFAULT true
)

-- Menu Items
menu_items (
  id: uuid PRIMARY KEY,
  name: text NOT NULL,
  description: text,
  category_id: uuid REFERENCES menu_categories(id),
  price: decimal(10,2) NOT NULL,
  cost: decimal(10,2),
  image_url: text,
  prep_time: integer, -- minutes
  is_available: boolean DEFAULT true,
  is_featured: boolean DEFAULT false,
  created_at: timestamp DEFAULT now(),
  updated_at: timestamp DEFAULT now()
)

-- Menu Item Ingredients (Many-to-Many)
menu_item_ingredients (
  id: uuid PRIMARY KEY,
  menu_item_id: uuid REFERENCES menu_items(id),
  ingredient_id: uuid REFERENCES ingredients(id),
  quantity: decimal(10,3) NOT NULL,
  unit: text NOT NULL,
  is_optional: boolean DEFAULT false
)

-- Orders
orders (
  id: uuid PRIMARY KEY,
  order_number: text UNIQUE NOT NULL,
  customer_name: text NOT NULL,
  customer_phone: text,
  customer_email: text,
  order_type: enum('dine-in', 'takeout') NOT NULL,
  table_number: integer,
  status: enum('pending', 'confirmed', 'preparing', 'ready', 'completed', 'cancelled') DEFAULT 'pending',
  payment_status: enum('unpaid', 'paid', 'refunded') DEFAULT 'unpaid',
  payment_method: enum('cash', 'gcash', 'card'),
  subtotal: decimal(10,2) NOT NULL,
  discount: decimal(10,2) DEFAULT 0,
  tax: decimal(10,2) DEFAULT 0,
  total: decimal(10,2) NOT NULL,
  special_instructions: text,
  created_by: uuid REFERENCES users(id),
  assigned_to: uuid REFERENCES users(id), -- kitchen staff
  created_at: timestamp DEFAULT now(),
  updated_at: timestamp DEFAULT now(),
  completed_at: timestamp
)

-- Order Items
order_items (
  id: uuid PRIMARY KEY,
  order_id: uuid REFERENCES orders(id),
  menu_item_id: uuid REFERENCES menu_items(id),
  quantity: integer NOT NULL,
  unit_price: decimal(10,2) NOT NULL,
  total_price: decimal(10,2) NOT NULL,
  customizations: jsonb, -- array of customization strings
  special_instructions: text,
  status: enum('pending', 'preparing', 'ready') DEFAULT 'pending'
)

-- Notifications
notifications (
  id: uuid PRIMARY KEY,
  user_id: uuid REFERENCES users(id),
  title: text NOT NULL,
  message: text NOT NULL,
  type: enum('info', 'warning', 'error', 'success'),
  is_read: boolean DEFAULT false,
  related_entity: text, -- 'order', 'inventory', 'system'
  entity_id: uuid,
  created_at: timestamp DEFAULT now()
)

-- System Settings
system_settings (
  id: uuid PRIMARY KEY,
  setting_key: text UNIQUE NOT NULL,
  setting_value: jsonb NOT NULL,
  description: text,
  updated_by: uuid REFERENCES users(id),
  updated_at: timestamp DEFAULT now()
)

-- Audit Logs
audit_logs (
  id: uuid PRIMARY KEY,
  user_id: uuid REFERENCES users(id),
  action: text NOT NULL,
  entity_type: text NOT NULL,
  entity_id: uuid,
  old_values: jsonb,
  new_values: jsonb,
  ip_address: inet,
  user_agent: text,
  created_at: timestamp DEFAULT now()
)
```

### API Endpoints Structure

#### Authentication Routes
```
POST   /api/auth/login          # User login
POST   /api/auth/register       # User registration
POST   /api/auth/refresh        # Refresh JWT token
POST   /api/auth/logout         # User logout
GET    /api/auth/profile        # Get user profile
PUT    /api/auth/profile        # Update user profile
```

#### Admin Routes
```
GET    /api/admin/dashboard     # Dashboard statistics
GET    /api/admin/employees     # List all employees
POST   /api/admin/employees     # Create new employee
PUT    /api/admin/employees/:id # Update employee
DELETE /api/admin/employees/:id # Delete employee
GET    /api/admin/reports       # Sales and analytics reports
GET    /api/admin/settings      # System settings
PUT    /api/admin/settings      # Update system settings
```

#### Inventory Routes
```
GET    /api/inventory/ingredients     # List all ingredients
POST   /api/inventory/ingredients     # Create new ingredient
PUT    /api/inventory/ingredients/:id # Update ingredient
DELETE /api/inventory/ingredients/:id # Delete ingredient
POST   /api/inventory/stock/in        # Stock in operation
POST   /api/inventory/stock/out       # Stock out operation
GET    /api/inventory/alerts          # Low stock alerts
GET    /api/inventory/reports         # Inventory reports
```

#### Menu Routes
```
GET    /api/menu/items               # List all menu items
POST   /api/menu/items               # Create new menu item
PUT    /api/menu/items/:id           # Update menu item
DELETE /api/menu/items/:id           # Delete menu item
GET    /api/menu/categories          # List categories
POST   /api/menu/categories          # Create category
PUT    /api/menu/availability        # Update menu availability
GET    /api/menu/ingredients/:id     # Get item ingredients
```

#### Order Routes
```
GET    /api/orders                   # List orders with filters
POST   /api/orders                   # Create new order
GET    /api/orders/:id               # Get order details
PUT    /api/orders/:id/status        # Update order status
DELETE /api/orders/:id               # Cancel order
GET    /api/orders/history           # Order history
POST   /api/orders/:id/notify        # Send order notifications
```

#### Cashier Routes
```
GET    /api/cashier/menu             # Available menu items
POST   /api/cashier/orders           # Create customer order
GET    /api/cashier/orders/active    # Active orders
PUT    /api/cashier/orders/:id/pay   # Process payment
GET    /api/cashier/customers        # Customer list
POST   /api/cashier/customers        # Add new customer
GET    /api/cashier/receipt/:id      # Generate receipt
```

#### Kitchen Routes
```
GET    /api/kitchen/orders/queue     # Orders in queue
GET    /api/kitchen/orders/active    # Currently preparing orders
PUT    /api/kitchen/orders/:id/status # Update order status
GET    /api/kitchen/inventory        # Current stock levels
POST   /api/kitchen/orders/:id/ready # Mark order ready
GET    /api/kitchen/equipment        # Equipment status
```

### Real-time Features

#### WebSocket Events
```javascript
// Order Events
'order:created'           // New order notification
'order:status_updated'    // Order status change
'order:ready'            // Order ready for pickup

// Inventory Events
'inventory:low_stock'     // Low stock alert
'inventory:out_of_stock'  // Out of stock alert
'inventory:updated'       // Stock level update

// System Events
'system:maintenance'      // System maintenance notification
'user:online'            // User online status
'user:offline'           // User offline status
```

### Security Features

#### Authentication & Authorization
- JWT-based authentication with refresh tokens
- Role-based access control (RBAC)
- Session management with configurable timeouts
- Password policies and encryption
- Multi-factor authentication (optional)

#### Data Protection
- Input validation and sanitization
- SQL injection prevention
- XSS protection
- CSRF protection
- Rate limiting
- Audit logging for all critical operations

### Performance & Scalability

#### Database Optimization
- Indexed queries for frequent operations
- Connection pooling
- Query optimization
- Caching strategies

#### API Optimization
- Response compression
- Pagination for large datasets
- Field selection (GraphQL-like)
- Request/response caching
- Rate limiting per user/role

### Monitoring & Logging

#### Application Monitoring
- Request/response logging
- Error tracking and alerting
- Performance metrics
- User activity monitoring
- Database query performance

#### Business Metrics
- Order processing times
- Inventory turnover rates
- Customer satisfaction metrics
- Revenue analytics
- Employee productivity tracking

## 🚀 Getting Started

### Prerequisites
- Node.js 18+ 
- npm or yarn
- Supabase account
- PostgreSQL knowledge (basic)

### Installation Steps

1. **Clone the repository**
   ```bash
   git clone <repository-url>
   cd restaurant-management-system
   ```

2. **Install dependencies**
   ```bash
   npm install
   ```

3. **Environment setup**
   ```bash
   cp .env.example .env
   # Configure your Supabase credentials
   ```

4. **Database setup**
   ```bash
   # Run database migrations
   npm run db:migrate
   # Seed initial data
   npm run db:seed
   ```

5. **Start development server**
   ```bash
   npm run dev
   ```

### Environment Variables

```env
# Server Configuration
PORT=3000
NODE_ENV=development

# Supabase Configuration
SUPABASE_URL=your_supabase_url
SUPABASE_ANON_KEY=your_supabase_anon_key
SUPABASE_SERVICE_ROLE_KEY=your_service_role_key

# JWT Configuration
JWT_SECRET=your_jwt_secret
JWT_EXPIRES_IN=24h
JWT_REFRESH_EXPIRES_IN=7d

# Security
BCRYPT_ROUNDS=12
SESSION_SECRET=your_session_secret

# External Services
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email
SMTP_PASS=your_password
```

## 📱 Frontend Integration

The React frontend components are already structured to work with this API:

- **Dashboard components** → `/api/admin/dashboard`
- **Inventory management** → `/api/inventory/*`
- **Menu management** → `/api/menu/*`
- **Order processing** → `/api/orders/*`
- **Cashier operations** → `/api/cashier/*`
- **Kitchen operations** → `/api/kitchen/*`

## 🧪 Testing

```bash
# Run all tests
npm test

# Run tests with coverage
npm run test:coverage

# Run specific test suites
npm run test:unit
npm run test:integration
npm run test:e2e
```

## 📊 API Documentation

API documentation is available at `/api/docs` when running the server, powered by Swagger/OpenAPI.

## 🔧 Development Workflow

1. **Feature Development**
   - Create feature branch from `main`
   - Implement feature with tests
   - Update API documentation
   - Create pull request

2. **Code Quality**
   - ESLint for code linting
   - Prettier for code formatting
   - Husky for pre-commit hooks
   - Conventional commits for commit messages

3. **Deployment**
   - Automated testing on push
   - Staging environment for testing
   - Production deployment with rollback capability

## 🤝 Contributing

1. Fork the repository
2. Create your feature branch
3. Commit your changes
4. Push to the branch
5. Create a Pull Request

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## 🆘 Support

For support and questions:
- Create an issue in the repository
- Contact the development team
- Check the documentation

## 🔮 Future Enhancements

- **Mobile App**: React Native mobile application
- **AI Integration**: Predictive inventory management
- **Analytics Dashboard**: Advanced business intelligence
- **Multi-location Support**: Chain restaurant management
- **Integration APIs**: Third-party service integrations
- **Offline Support**: PWA capabilities for offline operation
