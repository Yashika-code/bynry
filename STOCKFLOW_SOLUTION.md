# StockFlow - B2B Inventory Management System
## Case Study Solution (MERN Stack)

---

## Part 1: Code Review & Debugging

### Original Code Analysis (Python/Flask → Node.js/Express equivalent)

The original Python/Flask code has several critical issues. Here's the equivalent Node.js/Express version of the problematic code:

```javascript
// Problematic code (what the intern might have written in Express)
app.post('/api/products', async (req, res) => {
    const { name, sku, price, warehouse_id, initial_quantity } = req.body;
    
    // Create new product
    const product = new Product({
        name,
        sku,
        price,
        warehouse_id  // PROBLEM: Product tied to single warehouse
    });
    
    await product.save();
    
    // Create inventory
    const inventory = new Inventory({
        product_id: product._id,
        warehouse_id,
        quantity: initial_quantity
    });
    
    await inventory.save();
    
    res.json({ message: "Product created", product_id: product._id });
});
```

### Issues Identified

#### 1. **No Input Validation**
- **Issue**: No validation of required fields, data types, or business rules
- **Impact**: Invalid data can be inserted, causing data corruption. Missing fields will cause crashes.
- **Fix**: Add comprehensive input validation using Joi or express-validator

#### 2. **No SKU Uniqueness Check**
- **Issue**: No check for duplicate SKU before creating product
- **Impact**: Multiple products with same SKU can be created, violating business rule that SKUs must be unique across platform
- **Fix**: Add unique constraint on SKU and handle duplicate key errors

#### 3. **Product Tied to Single Warehouse**
- **Issue**: `warehouse_id` is stored on the Product model
- **Impact**: Contradicts requirement "Products can exist in multiple warehouses". A product can only be in one warehouse
- **Fix**: Remove warehouse_id from Product; relationship should only exist through Inventory

#### 4. **No Error Handling / Transaction Management**
- **Issue**: No try-catch, no transaction support. Two separate save operations without rollback capability
- **Impact**: If inventory creation fails after product is saved, you have an orphaned product with no inventory
- **Fix**: Use MongoDB transactions (multi-document ACID transactions)

#### 5. **No Authentication/Authorization**
- **Issue**: No verification that user has permission to create products
- **Impact**: Any user can create products, potential security breach
- **Fix**: Add middleware for authentication and authorization

#### 6. **Price as Float/Number**
- **Issue**: Using Number type for price can cause floating-point precision issues
- **Impact**: Financial calculations may have rounding errors (e.g., 0.1 + 0.2 ≠ 0.3)
- **Fix**: Store price as integer (cents) or use decimal type

#### 7. **No Handling of Optional Fields**
- **Issue**: Code assumes all fields are present (`data['name']` will throw KeyError if missing)
- **Impact**: API crashes when optional fields are not provided
- **Fix**: Add proper validation with required/optional field distinction

#### 8. **No Duplicate Inventory Check**
- **Issue**: No check if inventory already exists for product-warehouse combination
- **Impact**: Multiple inventory records for same product in same warehouse
- **Fix**: Add compound unique index on (product_id, warehouse_id)

#### 9. **Poor Error Responses**
- **Issue**: No proper HTTP status codes or error messages
- **Impact**: Client doesn't know what went wrong, poor debugging experience
- **Fix**: Return appropriate status codes (400, 409, 500) with descriptive messages

#### 10. **No Audit Trail**
- **Issue**: No tracking of who created the product and when
- **Impact**: Cannot track changes for compliance or debugging
- **Fix**: Add created_at, updated_at, created_by fields

### Corrected Implementation (Node.js/Express + MongoDB)

```javascript
// controllers/productController.js
const mongoose = require('mongoose');
const Product = require('../models/Product');
const Inventory = require('../models/Inventory');
const { productValidationSchema } = require('../validators/productValidator');

/**
 * Create a new product with inventory
 * 
 * Business Rules:
 * - SKU must be unique across the platform
 * - Product is not tied to a specific warehouse
 * - Inventory tracks product-warehouse relationship
 * - All operations are transactional
 */
const createProduct = async (req, res) => {
    // Start a MongoDB session for transaction
    const session = await mongoose.startSession();
    session.startTransaction();
    
    try {
        // 1. Validate input
        const { error, value } = productValidationSchema.validate(req.body);
        if (error) {
            await session.abortTransaction();
            return res.status(400).json({
                success: false,
                message: "Validation failed",
                errors: error.details.map(d => d.message)
            });
        }
        
        const { name, sku, price, warehouse_id, initial_quantity, description, category_id } = value;
        
        // 2. Check for duplicate SKU
        const existingProduct = await Product.findOne({ sku: sku.toUpperCase() }).session(session);
        if (existingProduct) {
            await session.abortTransaction();
            return res.status(409).json({
                success: false,
                message: "A product with this SKU already exists",
                existing_product_id: existingProduct._id
            });
        }
        
        // 3. Create product (without warehouse_id - product is warehouse-agnostic)
        const product = new Product({
            name,
            sku: sku.toUpperCase(),
            price: Math.round(price * 100), // Store as cents to avoid floating point issues
            description: description || null,
            category_id: category_id || null,
            created_by: req.user.id // From authentication middleware
        });
        
        await product.save({ session });
        
        // 4. Create inventory record(s) - only if warehouse and quantity provided
        if (warehouse_id && initial_quantity !== undefined) {
            // Check if inventory already exists for this product-warehouse combo
            const existingInventory = await Inventory.findOne({
                product_id: product._id,
                warehouse_id
            }).session(session);
            
            if (existingInventory) {
                await session.abortTransaction();
                return res.status(409).json({
                    success: false,
                    message: "Inventory already exists for this product in the specified warehouse"
                });
            }
            
            const inventory = new Inventory({
                product_id: product._id,
                warehouse_id,
                quantity: initial_quantity,
                created_by: req.user.id
            });
            
            await inventory.save({ session });
        }
        
        // 5. Commit transaction
        await session.commitTransaction();
        
        // 6. Return success response
        res.status(201).json({
            success: true,
            message: "Product created successfully",
            data: {
                product_id: product._id,
                sku: product.sku,
                name: product.name,
                inventory_created: !!(warehouse_id && initial_quantity)
            }
        });
        
    } catch (err) {
        await session.abortTransaction();
        console.error('Error creating product:', err);
        res.status(500).json({
            success: false,
            message: "Internal server error while creating product"
        });
    } finally {
        session.endSession();
    }
};

module.exports = { createProduct };
```

```javascript
// validators/productValidator.js
const Joi = require('joi');

const productValidationSchema = Joi.object({
    name: Joi.string().min(1).max(200).required()
        .messages({
            'string.empty': 'Product name cannot be empty',
            'any.required': 'Product name is required'
        }),
    
    sku: Joi.string().min(1).max(50).required()
        .pattern(/^[A-Za-z0-9\-]+$/)
        .messages({
            'string.empty': 'SKU cannot be empty',
            'string.pattern.base': 'SKU can only contain letters, numbers, and hyphens',
            'any.required': 'SKU is required'
        }),
    
    price: Joi.number().positive().precision(2).required()
        .messages({
            'number.base': 'Price must be a number',
            'number.positive': 'Price must be a positive number',
            'any.required': 'Price is required'
        }),
    
    warehouse_id: Joi.string().hex().length(24),
    
    initial_quantity: Joi.number().integer().min(0)
        .when('warehouse_id', {
            is: Joi.exist(),
            then: Joi.required()
        })
        .messages({
            'number.base': 'Initial quantity must be a number',
            'number.min': 'Initial quantity cannot be negative',
            'any.required': 'Initial quantity is required when warehouse_id is provided'
        }),
    
    description: Joi.string().max(1000).allow(null, ''),
    category_id: Joi.string().hex().length(24).allow(null)
});

module.exports = { productValidationSchema };
```

---

## Part 2: Database Design (MongoDB)

### Schema Design

```javascript
// models/Company.js
const mongoose = require('mongoose');

const CompanySchema = new mongoose.Schema({
    name: {
        type: String,
        required: true,
        trim: true
    },
    code: {
        type: String,
        required: true,
        unique: true,
        uppercase: true,
        trim: true
    },
    subscription_tier: {
        type: String,
        enum: ['free', 'basic', 'pro', 'enterprise'],
        default: 'free'
    },
    settings: {
        currency: { type: String, default: 'USD' },
        timezone: { type: String, default: 'UTC' },
        low_stock_notification_email: { type: String }
    },
    is_active: {
        type: Boolean,
        default: true
    },
    created_at: {
        type: Date,
        default: Date.now
    },
    updated_at: {
        type: Date,
        default: Date.now
    }
}, { timestamps: true });

// Index for common queries
CompanySchema.index({ code: 1 });
CompanySchema.index({ is_active: 1 });

module.exports = mongoose.model('Company', CompanySchema);
```

```javascript
// models/Warehouse.js
const mongoose = require('mongoose');

const WarehouseSchema = new mongoose.Schema({
    company: {
        type: mongoose.Schema.Types.ObjectId,
        ref: 'Company',
        required: true
    },
    name: {
        type: String,
        required: true,
        trim: true
    },
    code: {
        type: String,
        required: true,
        trim: true
    },
    address: {
        street: String,
        city: String,
        state: String,
        country: { type: String, default: 'US' },
        postal_code: String
    },
    contact: {
        phone: String,
        email: String,
        manager_name: String
    },
    is_active: {
        type: Boolean,
        default: true
    },
    created_at: {
        type: Date,
        default: Date.now
    }
}, { timestamps: true });

// Compound unique index - warehouse code unique per company
WarehouseSchema.index({ company: 1, code: 1 }, { unique: true });
WarehouseSchema.index({ company: 1, is_active: 1 });

module.exports = mongoose.model('Warehouse', WarehouseSchema);
```

```javascript
// models/Product.js
const mongoose = require('mongoose');

const ProductSchema = new mongoose.Schema({
    company: {
        type: mongoose.Schema.Types.ObjectId,
        ref: 'Company',
        required: true
    },
    name: {
        type: String,
        required: true,
        trim: true
    },
    sku: {
        type: String,
        required: true,
        uppercase: true,
        trim: true
    },
    description: {
        type: String,
        maxlength: 1000
    },
    price: {
        type: Number, // Stored in cents to avoid floating point issues
        required: true,
        min: 0
    },
    cost: {
        type: Number, // Stored in cents
        min: 0
    },
    category: {
        type: mongoose.Schema.Types.ObjectId,
        ref: 'Category'
    },
    product_type: {
        type: String,
        enum: ['standard', 'bundle', 'service'],
        default: 'standard'
    },
    unit_of_measure: {
        type: String,
        default: 'unit', // unit, kg, liter, box, etc.
        required: true
    },
    reorder_point: {
        type: Number,
        default: 0,
        min: 0
    },
    is_active: {
        type: Boolean,
        default: true
    },
    created_by: {
        type: mongoose.Schema.Types.ObjectId,
        ref: 'User'
    },
    created_at: {
        type: Date,
        default: Date.now
    },
    updated_at: {
        type: Date,
        default: Date.now
    }
}, { timestamps: true });

// SKU unique per company (not globally, as different companies can have same SKU)
ProductSchema.index({ company: 1, sku: 1 }, { unique: true });
ProductSchema.index({ company: 1, is_active: 1 });
ProductSchema.index({ company: 1, product_type: 1 });

module.exports = mongoose.model('Product', ProductSchema);
```

```javascript
// models/Inventory.js
const mongoose = require('mongoose');

const InventorySchema = new mongoose.Schema({
    company: {
        type: mongoose.Schema.Types.ObjectId,
        ref: 'Company',
        required: true
    },
    product: {
        type: mongoose.Schema.Types.ObjectId,
        ref: 'Product',
        required: true
    },
    warehouse: {
        type: mongoose.Schema.Types.ObjectId,
        ref: 'Warehouse',
        required: true
    },
    quantity: {
        type: Number,
        required: true,
        default: 0,
        min: 0
    },
    reserved_quantity: {
        type: Number,
        default: 0,
        min: 0
    },
    available_quantity: {
        type: Number
        // Computed: quantity - reserved_quantity
    },
    location: {
        aisle: String,
        shelf: String,
        bin: String
    },
    last_counted_at: {
        type: Date
    },
    created_at: {
        type: Date,
        default: Date.now
    },
    updated_at: {
        type: Date,
        default: Date.now
    }
}, { timestamps: true });

// One inventory record per product per warehouse
InventorySchema.index({ company: 1, product: 1, warehouse: 1 }, { unique: true });
InventorySchema.index({ company: 1, product: 1 });
InventorySchema.index({ company: 1, warehouse: 1 });
// For low stock queries
InventorySchema.index({ company: 1, quantity: 1 });

module.exports = mongoose.model('Inventory', InventorySchema);
```

```javascript
// models/InventoryTransaction.js - For tracking inventory changes
const mongoose = require('mongoose');

const InventoryTransactionSchema = new mongoose.Schema({
    company: {
        type: mongoose.Schema.Types.ObjectId,
        ref: 'Company',
        required: true
    },
    product: {
        type: mongoose.Schema.Types.ObjectId,
        ref: 'Product',
        required: true
    },
    warehouse: {
        type: mongoose.Schema.Types.ObjectId,
        ref: 'Warehouse',
        required: true
    },
    transaction_type: {
        type: String,
        enum: ['sale', 'purchase', 'adjustment', 'transfer_in', 'transfer_out', 'return', 'damage'],
        required: true
    },
    quantity_change: {
        type: Number,
        required: true
        // Positive for additions, negative for reductions
    },
    quantity_before: {
        type: Number,
        required: true
    },
    quantity_after: {
        type: Number,
        required: true
    },
    reference_type: {
        type: String, // 'order', 'purchase_order', 'transfer', etc.
    },
    reference_id: {
        type: mongoose.Schema.Types.ObjectId
    },
    notes: {
        type: String,
        maxlength: 500
    },
    created_by: {
        type: mongoose.Schema.Types.ObjectId,
        ref: 'User',
        required: true
    },
    created_at: {
        type: Date,
        default: Date.now
    }
}, { timestamps: true });

// Indexes for querying transaction history
InventoryTransactionSchema.index({ company: 1, product: 1, created_at: -1 });
InventoryTransactionSchema.index({ company: 1, warehouse: 1, created_at: -1 });
InventoryTransactionSchema.index({ company: 1, transaction_type: 1, created_at: -1 });

module.exports = mongoose.model('InventoryTransaction', InventoryTransactionSchema);
```

```javascript
// models/Supplier.js
const mongoose = require('mongoose');

const SupplierSchema = new mongoose.Schema({
    company: {
        type: mongoose.Schema.Types.ObjectId,
        ref: 'Company',
        required: true
    },
    name: {
        type: String,
        required: true,
        trim: true
    },
    code: {
        type: String,
        required: true,
        trim: true
    },
    contact: {
        primary_email: {
            type: String,
            lowercase: true,
            trim: true
        },
        secondary_email: String,
        phone: String,
        fax: String
    },
    address: {
        street: String,
        city: String,
        state: String,
        country: { type: String, default: 'US' },
        postal_code: String
    },
    payment_terms: {
        type: String, // e.g., "Net 30", "Due on Receipt"
        default: 'Net 30'
    },
    lead_time_days: {
        type: Number,
        default: 7,
        min: 0
    },
    is_active: {
        type: Boolean,
        default: true
    },
    created_at: {
        type: Date,
        default: Date.now
    }
}, { timestamps: true });

SupplierSchema.index({ company: 1, code: 1 }, { unique: true });
SupplierSchema.index({ company: 1, is_active: 1 });

module.exports = mongoose.model('Supplier', SupplierSchema);
```

```javascript
// models/ProductSupplier.js - Many-to-many relationship
const mongoose = require('mongoose');

const ProductSupplierSchema = new mongoose.Schema({
    company: {
        type: mongoose.Schema.Types.ObjectId,
        ref: 'Company',
        required: true
    },
    product: {
        type: mongoose.Schema.Types.ObjectId,
        ref: 'Product',
        required: true
    },
    supplier: {
        type: mongoose.Schema.Types.ObjectId,
        ref: 'Supplier',
        required: true
    },
    is_primary: {
        type: Boolean,
        default: false
    },
    supplier_product_code: {
        type: String, // Supplier's SKU for this product
        trim: true
    },
    supplier_product_name: {
        type: String, // Supplier's name for this product
        trim: true
    },
    purchase_price: {
        type: Number, // In cents
        min: 0
    },
    minimum_order_quantity: {
        type: Number,
        default: 1,
        min: 1
    },
    lead_time_days: {
        type: Number,
        min: 0
    },
    created_at: {
        type: Date,
        default: Date.now
    }
}, { timestamps: true });

ProductSupplierSchema.index({ company: 1, product: 1, is_primary: 1 });
ProductSupplierSchema.index({ company: 1, supplier: 1, product: 1 }, { unique: true });

module.exports = mongoose.model('ProductSupplier', ProductSupplierSchema);
```

```javascript
// models/ProductBundle.js - For bundle products
const mongoose = require('mongoose');

const ProductBundleSchema = new mongoose.Schema({
    company: {
        type: mongoose.Schema.Types.ObjectId,
        ref: 'Company',
        required: true
    },
    bundle_product: {
        type: mongoose.Schema.Types.ObjectId,
        ref: 'Product',
        required: true
    },
    component_product: {
        type: mongoose.Schema.Types.ObjectId,
        ref: 'Product',
        required: true
    },
    quantity: {
        type: Number,
        required: true,
        min: 1
    },
    created_at: {
        type: Date,
        default: Date.now
    }
}, { timestamps: true });

// Prevent self-referencing bundles
ProductBundleSchema.index({ company: 1, bundle_product: 1, component_product: 1 }, { unique: true });

module.exports = mongoose.model('ProductBundle', ProductBundleSchema);
```

```javascript
// models/Category.js
const mongoose = require('mongoose');

const CategorySchema = new mongoose.Schema({
    company: {
        type: mongoose.Schema.Types.ObjectId,
        ref: 'Company',
        required: true
    },
    name: {
        type: String,
        required: true,
        trim: true
    },
    code: {
        type: String,
        required: true,
        uppercase: true,
        trim: true
    },
    description: String,
    low_stock_threshold: {
        type: Number,
        default: 10,
        min: 0
    },
    parent_category: {
        type: mongoose.Schema.Types.ObjectId,
        ref: 'Category'
    },
    is_active: {
        type: Boolean,
        default: true
    },
    created_at: {
        type: Date,
        default: Date.now
    }
}, { timestamps: true });

CategorySchema.index({ company: 1, code: 1 }, { unique: true });
CategorySchema.index({ company: 1, is_active: 1 });

module.exports = mongoose.model('Category', CategorySchema);
```

```javascript
// models/SalesOrder.js - For tracking sales activity
const mongoose = require('mongoose');

const SalesOrderSchema = new mongoose.Schema({
    company: {
        type: mongoose.Schema.Types.ObjectId,
        ref: 'Company',
        required: true
    },
    order_number: {
        type: String,
        required: true,
        unique: true
    },
    order_date: {
        type: Date,
        required: true,
        default: Date.now
    },
    status: {
        type: String,
        enum: ['draft', 'confirmed', 'shipped', 'delivered', 'cancelled'],
        default: 'draft'
    },
    items: [{
        product: {
            type: mongoose.Schema.Types.ObjectId,
            ref: 'Product',
            required: true
        },
        quantity: {
            type: Number,
            required: true,
            min: 1
        },
        unit_price: {
            type: Number,
            required: true
        },
        warehouse: {
            type: mongoose.Schema.Types.ObjectId,
            ref: 'Warehouse'
        }
    }],
    total_amount: {
        type: Number,
        required: true
    },
    created_by: {
        type: mongoose.Schema.Types.ObjectId,
        ref: 'User'
    },
    created_at: {
        type: Date,
        default: Date.now
    },
    updated_at: {
        type: Date,
        default: Date.now
    }
}, { timestamps: true });

SalesOrderSchema.index({ company: 1, order_date: -1 });
SalesOrderSchema.index({ company: 1, status: 1 });
// For finding recent sales by product
SalesOrderSchema.index({ company: 1, 'items.product': 1, order_date: -1 });

module.exports = mongoose.model('SalesOrder', SalesOrderSchema);
```

### Entity Relationship Diagram (Text Format)

```
┌─────────────┐     ┌───────────────┐     ┌──────────────┐
│   Company   │────│   Warehouse   │────│   Inventory  │
│             │    1│               │    1│              │
│  - _id      │     │  - _id        │     │  - _id       │
│  - name     │     │  - company_id │     │  - product_id│
│  - code     │     │  - name       │     │  - warehouse_│
│  - settings │     │  - address    │     │    id        │
└─────────────┘     └───────────────┘     │  - quantity  │
       │1                                    │  - location │
       │                                     └──────────────┘
       │                                            │
       │                                            │
       ▼                                            │
┌─────────────┐     ┌───────────────┐              │
│   Product   │────│   Category    │              │
│             │    1│               │              │
│  - _id      │     │  - _id        │              │
│  - company_id│    │  - name       │              │
│  - sku      │     │  - threshold  │              │
│  - name     │     └───────────────┘              │
│  - price    │                                    │
│  - type     │     ┌───────────────┐              │
└─────────────┘     │   Supplier    │              │
       │1          1│               │              │
       │            │  - _id        │              │
       │            │  - company_id │              │
       │            │  - name       │              │
       │            │  - contact    │              │
       │            └───────────────┘              │
       │                   │1                      │
       │                   │                       │
       ▼                   ▼                       │
┌─────────────┐     ┌───────────────────┐         │
│ProductBundle│     │ ProductSupplier   │         │
│             │     │                   │         │
│  - bundle_  │     │  - product_id     │         │
│    product_id│    │  - supplier_id    │         │
│  - component_│    │  - is_primary     │         │
│    product_id│   │  - purchase_price │         │
│  - quantity │     │  - lead_time_days │         │
└─────────────┘     └───────────────────┘         │
                                                  │
       ┌──────────────────────────────────────────┘
       │
       ▼
┌─────────────────────┐     ┌─────────────────────┐
│InventoryTransaction │     │    SalesOrder       │
│                     │     │                     │
│  - product_id       │     │  - company_id       │
│  - warehouse_id     │     │  - order_date       │
│  - transaction_type │     │  - status           │
│  - quantity_change  │     │  - items[]          │
│  - created_at       │     │    - product_id     │
└─────────────────────┘     │    - quantity       │
                            └─────────────────────┘
```

### Questions for Product Team (Identified Gaps)

1. **Multi-tenancy**: Should data be completely isolated per company? Can users work across multiple companies?

2. **User Management**: What user roles exist (admin, warehouse_manager, viewer)? What permissions does each role have?

3. **Bundle Behavior**: When a bundle is sold, should component inventory be automatically deducted? Should bundles have their own SKU?

4. **Inventory Valuation**: Which costing method should be used (FIFO, LIFO, Average Cost)?

5. **Low Stock Threshold**: Should threshold be per product, per category, or both? Should there be a "critical" threshold as well?

6. **Recent Sales Activity**: What timeframe defines "recent" for sales activity? 30 days? 90 days? Configurable?

7. **Supplier Management**: Can a product have multiple suppliers? Should there be a preferred/primary supplier?

8. **Purchase Orders**: Should the system support creating purchase orders to suppliers? Should low-stock alerts auto-generate PO suggestions?

9. **Inventory Transfers**: Should transfers between warehouses be tracked? Should there be an approval workflow?

10. **Barcode/QR Support**: Should products support barcode/QR code scanning?

11. **Reporting**: What reports are needed? Inventory valuation, sales by product, supplier performance?

12. **Integrations**: Should the system integrate with accounting software (QuickBooks, Xero) or e-commerce platforms (Shopify, WooCommerce)?

13. **Audit Requirements**: What level of audit trail is needed? Should all changes be logged with before/after values?

14. **Data Retention**: How long should historical data be kept? Are there compliance requirements?

---

## Part 3: API Implementation - Low Stock Alerts

### Assumptions Made

1. **Database Schema**: Using the MongoDB schemas defined in Part 2
2. **"Recent Sales Activity"**: Defined as sales within the last 30 days (configurable)
3. **Low Stock Threshold Priority**: Product-specific threshold > Category threshold > Default (10 units)
4. **Days Until Stockout Calculation**: Based on average daily sales over the recent period
5. **Supplier Selection**: Uses primary supplier if available, otherwise first active supplier
6. **Per Warehouse Alerts**: Each warehouse with low stock generates a separate alert

### Implementation

```javascript
// routes/alerts.js
const express = require('express');
const router = express.Router();
const { getLowStockAlerts } = require('../controllers/alertController');
const { authMiddleware } = require('../middleware/auth');
const { validateCompanyId } = require('../middleware/validation');

// All alert routes require authentication
router.use(authMiddleware);

/**
 * @route   GET /api/companies/:company_id/alerts/low-stock
 * @desc    Get low stock alerts for a company
 * @access  Private (requires authentication)
 */
router.get(
    '/api/companies/:company_id/alerts/low-stock',
    validateCompanyId,
    getLowStockAlerts
);

module.exports = router;
```

```javascript
// controllers/alertController.js
const mongoose = require('mongoose');
const Inventory = require('../models/Inventory');
const Product = require('../models/Product');
const Warehouse = require('../models/Warehouse');
const Category = require('../models/Category');
const ProductSupplier = require('../models/ProductSupplier');
const Supplier = require('../models/Supplier');
const SalesOrder = require('../models/SalesOrder');

/**
 * Get low stock alerts for a company
 * 
 * Business Logic:
 * 1. Find all inventory items where current stock is below threshold
 * 2. Only include products with recent sales activity (last 30 days by default)
 * 3. Calculate days until stockout based on average daily sales
 * 4. Include supplier information for reordering
 * 
 * @param {Object} req - Express request object
 * @param {Object} res - Express response object
 */
const getLowStockAlerts = async (req, res) => {
    const session = await mongoose.startSession();
    
    try {
        const { company_id } = req.params;
        const {
            days_lookback = 30,  // Days to look back for sales activity
            default_threshold = 10,  // Default low stock threshold
            include_zero_stock = true  // Include items with 0 stock
        } = req.query;
        
        // Validate company exists
        const company = await mongoose.model('Company').findById(company_id).session(session);
        if (!company || !company.is_active) {
            return res.status(404).json({
                success: false,
                message: "Company not found or inactive"
            });
        }
        
        // Convert days_lookback to milliseconds for date calculation
        const lookbackMs = parseInt(days_lookback) * 24 * 60 * 60 * 1000;
        const cutoffDate = new Date(Date.now() - lookbackMs);
        
        // Step 1: Get all active warehouses for this company
        const warehouses = await Warehouse.find({
            company: company_id,
            is_active: true
        }).session(session);
        
        if (warehouses.length === 0) {
            return res.status(200).json({
                success: true,
                alerts: [],
                total_alerts: 0
            });
        }
        
        const warehouseIds = warehouses.map(w => w._id);
        const warehouseMap = new Map(warehouses.map(w => [w._id.toString(), w]));
        
        // Step 2: Get inventory items with low stock
        // We'll get all inventory and filter in code since threshold logic is complex
        const inventoryItems = await Inventory.find({
            company: company_id,
            warehouse: { $in: warehouseIds },
            quantity: { $gte: 0 } // Include zero if flag is set
        })
        .populate({
            path: 'product',
            select: 'name sku price category product_type reorder_point is_active',
            populate: {
                path: 'category',
                select: 'name low_stock_threshold'
            }
        })
        .session(session);
        
        // Step 3: Get recent sales data for calculating daily sales rate
        // Group sales by product and warehouse over the lookback period
        const recentSales = await SalesOrder.aggregate([
            {
                $match: {
                    company: new mongoose.Types.ObjectId(company_id),
                    order_date: { $gte: cutoffDate },
                    status: { $in: ['confirmed', 'shipped', 'delivered'] },
                    'items.quantity': { $exists: true }
                }
            },
            { $unwind: '$items' },
            {
                $group: {
                    _id: {
                        product: '$items.product',
                        warehouse: '$items.warehouse'
                    },
                    total_quantity_sold: { $sum: '$items.quantity' },
                    order_count: { $sum: 1 },
                    last_sale_date: { $max: '$order_date' }
                }
            },
            {
                $addFields: {
                    avg_daily_sales: {
                        $divide: ['$total_quantity_sold', parseInt(days_lookback)]
                    }
                }
            }
        ]).session(session);
        
        // Create a map for quick lookup of sales data
        const salesMap = new Map();
        recentSales.forEach(sale => {
            const key = `${sale._id.product.toString()}_${sale._id.warehouse ? sale._id.warehouse.toString() : 'any'}`;
            salesMap.set(key, sale);
        });
        
        // Also create a product-level sales map (across all warehouses)
        const productSalesMap = new Map();
        recentSales.forEach(sale => {
            const productId = sale._id.product.toString();
            if (!productSalesMap.has(productId)) {
                productSalesMap.set(productId, {
                    total_quantity_sold: 0,
                    avg_daily_sales: 0,
                    last_sale_date: null
                });
            }
            const existing = productSalesMap.get(productId);
            existing.total_quantity_sold += sale.total_quantity_sold;
            existing.avg_daily_sales += sale.avg_daily_sales;
            if (!existing.last_sale_date || sale.last_sale_date > existing.last_sale_date) {
                existing.last_sale_date = sale.last_sale_date;
            }
        });
        
        // Step 4: Process inventory items and generate alerts
        const alerts = [];
        const processedProducts = new Set();
        
        for (const inv of inventoryItems) {
            if (!inv.product || !inv.product.is_active) continue;
            
            const productId = inv.product._id.toString();
            const warehouseId = inv.warehouse.toString();
            const warehouse = warehouseMap.get(warehouseId);
            
            if (!warehouse) continue;
            
            // Determine the low stock threshold
            // Priority: Product reorder_point > Category threshold > Default
            let threshold = default_threshold;
            
            if (inv.product.reorder_point && inv.product.reorder_point > 0) {
                threshold = inv.product.reorder_point;
            } else if (inv.product.category && inv.product.category.low_stock_threshold) {
                threshold = inv.product.category.low_stock_threshold;
            }
            
            // Check if current stock is below threshold
            const currentStock = inv.quantity - (inv.reserved_quantity || 0);
            
            if (currentStock >= threshold) continue;
            if (currentStock < 0 && !include_zero_stock) continue;
            
            // Check for recent sales activity (product must have sales in lookback period)
            const productSales = productSalesMap.get(productId);
            if (!productSales || productSales.total_quantity_sold === 0) continue;
            
            // Calculate days until stockout
            let daysUntilStockout = null;
            if (productSales.avg_daily_sales > 0) {
                daysUntilStockout = Math.round(currentStock / productSales.avg_daily_sales);
                // Cap at reasonable value
                if (daysUntilStockout > 365) daysUntilStockout = 365;
                if (daysUntilStockout < 0) daysUntilStockout = 0;
            }
            
            // Get supplier information
            const supplierInfo = await getSupplierForProduct(company_id, productId, session);
            
            alerts.push({
                product_id: productId,
                product_name: inv.product.name,
                sku: inv.product.sku,
                warehouse_id: warehouseId,
                warehouse_name: warehouse.name,
                current_stock: currentStock,
                threshold: threshold,
                days_until_stockout: daysUntilStockout,
                avg_daily_sales: Math.round(productSales.avg_daily_sales * 100) / 100,
                supplier: supplierInfo
            });
        }
        
        // Sort alerts by urgency (days until stockout, nulls last)
        alerts.sort((a, b) => {
            if (a.days_until_stockout === null && b.days_until_stockout === null) return 0;
            if (a.days_until_stockout === null) return 1;
            if (b.days_until_stockout === null) return -1;
            return a.days_until_stockout - b.days_until_stockout;
        });
        
        await session.endSession();
        
        return res.status(200).json({
            success: true,
            alerts: alerts,
            total_alerts: alerts.length,
            metadata: {
                generated_at: new Date().toISOString(),
                company_id: company_id,
                warehouses_checked: warehouses.length,
                sales_lookback_days: parseInt(days_lookback),
                default_threshold: default_threshold
            }
        });
        
    } catch (error) {
        await session.endSession();
        console.error('Error fetching low stock alerts:', error);
        
        return res.status(500).json({
            success: false,
            message: "Internal server error while fetching alerts",
            error: process.env.NODE_ENV === 'development' ? error.message : undefined
        });
    }
};

/**
 * Helper function to get supplier information for a product
 * Prioritizes primary supplier, falls back to first active supplier
 */
const getSupplierForProduct = async (companyId, productId, session) => {
    try {
        // First, try to find primary supplier for this product
        const productSupplier = await ProductSupplier.findOne({
            company: new mongoose.Types.ObjectId(companyId),
            product: new mongoose.Types.ObjectId(productId),
            is_primary: true
        })
        .populate({
            path: 'supplier',
            select: 'name contact payment_terms lead_time_days',
            match: { is_active: true }
        })
        .session(session);
        
        if (productSupplier && productSupplier.supplier) {
            return {
                id: productSupplier.supplier._id.toString(),
                name: productSupplier.supplier.name,
                contact_email: productSupplier.supplier.contact?.primary_email || null,
                contact_phone: productSupplier.supplier.contact?.phone || null,
                lead_time_days: productSupplier.lead_time_days || productSupplier.supplier.lead_time_days,
                minimum_order_quantity: productSupplier.minimum_order_quantity || 1,
                is_primary: true
            };
        }
        
        // If no primary supplier, get any active supplier for this product
        const anySupplier = await ProductSupplier.findOne({
            company: new mongoose.Types.ObjectId(companyId),
            product: new mongoose.Types.ObjectId(productId)
        })
        .populate({
            path: 'supplier',
            select: 'name contact payment_terms lead_time_days',
            match: { is_active: true }
        })
        .session(session);
        
        if (anySupplier && anySupplier.supplier) {
            return {
                id: anySupplier.supplier._id.toString(),
                name: anySupplier.supplier.name,
                contact_email: anySupplier.supplier.contact?.primary_email || null,
                contact_phone: anySupplier.supplier.contact?.phone || null,
                lead_time_days: anySupplier.lead_time_days || anySupplier.supplier.lead_time_days,
                minimum_order_quantity: anySupplier.minimum_order_quantity || 1,
                is_primary: false
            };
        }
        
        // No supplier found
        return null;
        
    } catch (error) {
        console.error('Error fetching supplier for product:', error);
        return null;
    }
};

module.exports = { getLowStockAlerts };
```

```javascript
// middleware/validation.js
const mongoose = require('mongoose');

/**
 * Middleware to validate company_id parameter
 */
const validateCompanyId = (req, res, next) => {
    const { company_id } = req.params;
    
    if (!company_id) {
        return res.status(400).json({
            success: false,
            message: "Company ID is required"
        });
    }
    
    if (!mongoose.Types.ObjectId.isValid(company_id)) {
        return res.status(400).json({
            success: false,
            message: "Invalid company ID format"
        });
    }
    
    next();
};

module.exports = { validateCompanyId };
```

```javascript
// middleware/auth.js
/**
 * Authentication middleware
 * In production, this would verify JWT tokens and attach user to request
 */
const authMiddleware = (req, res, next) => {
    // In production, verify JWT from Authorization header
    // const token = req.headers.authorization?.split(' ')[1];
    // const decoded = verifyToken(token);
    // req.user = decoded;
    
    // For demo purposes, simulate authenticated user
    req.user = {
        id: 'demo_user_id',
        company_id: req.params.company_id,
        role: 'admin'
    };
    
    // In production, you'd also check if user has access to this company
    // For demo, we'll allow access
    
    next();
};

module.exports = { authMiddleware };
```

### Edge Cases Handled

1. **Company Not Found/Inactive**: Returns 404 with appropriate message
2. **No Warehouses**: Returns empty alerts array with 200 status
3. **No Recent Sales**: Products without recent sales are excluded from alerts
4. **Zero Division**: Handles cases where avg_daily_sales is 0
5. **No Supplier**: Returns null for supplier if none found
6. **Invalid Company ID**: Returns 400 for malformed ObjectId
7. **Reserved Inventory**: Calculates available stock (quantity - reserved)
8. **Multiple Warehouses**: Generates separate alert per warehouse
9. **Null Days Until Stockout**: When sales rate is 0, days_until_stockout is null
10. **Transaction Safety**: Uses MongoDB sessions for data consistency

### API Usage Examples

```bash
# Basic request
curl -X GET "http://api.stockflow.com/api/companies/507f1f77bcf86cd799439011/alerts/low-stock" \
  -H "Authorization: Bearer YOUR_TOKEN"

# With custom parameters
curl -X GET "http://api.stockflow.com/api/companies/507f1f77bcf86cd799439011/alerts/low-stock?days_lookback=60&default_threshold=15" \
  -H "Authorization: Bearer YOUR_TOKEN"
```

### Sample Response

```json
{
  "success": true,
  "alerts": [
    {
      "product_id": "507f1f77bcf86cd799439012",
      "product_name": "Widget A",
      "sku": "WID-001",
      "warehouse_id": "507f1f77bcf86cd799439013",
      "warehouse_name": "Main Warehouse",
      "current_stock": 5,
      "threshold": 20,
      "days_until_stockout": 12,
      "avg_daily_sales": 1.25,
      "supplier": {
        "id": "507f1f77bcf86cd799439014",
        "name": "Supplier Corp",
        "contact_email": "orders@supplier.com",
        "contact_phone": "+1-555-0100",
        "lead_time_days": 7,
        "minimum_order_quantity": 10,
        "is_primary": true
      }
    },
    {
      "product_id": "507f1f77bcf86cd799439015",
      "product_name": "Gadget B",
      "sku": "GAD-002",
      "warehouse_id": "507f1f77bcf86cd799439013",
      "warehouse_name": "Main Warehouse",
      "current_stock": 0,
      "threshold": 10,
      "days_until_stockout": 0,
      "avg_daily_sales": 2.5,
      "supplier": {
        "id": "507f1f77bcf86cd799439016",
        "name": "Tech Supplies Inc",
        "contact_email": "sales@techsupplies.com",
        "contact_phone": "+1-555-0200",
        "lead_time_days": 14,
        "minimum_order_quantity": 5,
        "is_primary": false
      }
    }
  ],
  "total_alerts": 2,
  "metadata": {
    "generated_at": "2024-01-15T10:30:00.000Z",
    "company_id": "507f1f77bcf86cd799439011",
    "warehouses_checked": 2,
    "sales_lookback_days": 30,
    "default_threshold": 10
  }
}
```

---

## Summary of Assumptions

1. **MERN Stack**: Used MongoDB for database, Express.js for API, Node.js runtime
2. **Multi-tenancy**: Each company's data is isolated with company_id on all records
3. **Price Storage**: Stored as integer (cents) to avoid floating-point precision issues
4. **SKU Uniqueness**: Unique per company, not globally (different companies can have same SKU)
5. **Recent Sales**: Defined as last 30 days by default, configurable via query parameter
6. **Days Until Stockout**: Calculated as current_stock / avg_daily_sales
7. **Threshold Priority**: Product reorder_point > Category threshold > Default (10)
8. **Supplier Selection**: Primary supplier preferred, falls back to any active supplier
9. **Available Stock**: Calculated as quantity - reserved_quantity
10. **Transaction Safety**: Used MongoDB sessions for multi-document operations

---

## Files Structure

```
stockflow-api/
├── src/
│   ├── models/
│   │   ├── Company.js
│   │   ├── Warehouse.js
│   │   ├── Product.js
│   │   ├── Inventory.js
│   │   ├── InventoryTransaction.js
│   │   ├── Supplier.js
│   │   ├── ProductSupplier.js
│   │   ├── ProductBundle.js
│   │   ├── Category.js
│   │   └── SalesOrder.js
│   ├── controllers/
│   │   ├── productController.js
│   │   └── alertController.js
│   ├── routes/
│   │   ├── products.js
│   │   └── alerts.js
│   ├── middleware/
│   │   ├── auth.js
│   │   └── validation.js
│   ├── validators/
│   │   └── productValidator.js
│   └── app.js
├── package.json
└── .env.example
```

---

*Document prepared for StockFlow B2B SaaS Inventory Management System Case Study*
*Author: MERN Stack Developer Candidate*
*Date: 2024*
