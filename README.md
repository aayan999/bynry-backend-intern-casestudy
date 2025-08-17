# Bynry Inc. - Backend Intern Case Study Submission

My submission for the backend developer case study.

---

## Part 1: Code Review & Debugging

### Issues Identified

1.  **Missing Error Handling:** The code directly accesses `data['name']` and other keys without checking if they exist. If a field is missing in the request, the application will crash with a `KeyError`.
2.  **Not a Single Database Transaction:** There are two separate `db.session.commit()` calls. If the first commit succeeds but the second one fails, the database will be left in an inconsistent state with a product record that has no corresponding inventory.
3. No Validation of Business Rules: The code does not check if a product with the same SKU already exists before creating a new one. This violates the business rule that SKUs must be unique across the platform[cite: 36].
4. **No Handling of Optional Fields:** The case study mentions that some fields might be optional[cite: 37], but the code treats all fields as mandatory, making the endpoint inflexible.

### Corrected Code

```python
from flask import request, jsonify

@app.route('/api/products', methods=['POST'])
def create_product():
    data = request.get_json()

    # 1. Validate required fields are present
    required_fields = ['name', 'sku', 'price', 'warehouse_id', 'initial_quantity']
    if not data or not all(field in data for field in required_fields):
        return jsonify({"error": "Missing required fields"}), 400

    name = data['name']
    sku = data['sku']
    price = data['price']
    warehouse_id = data['warehouse_id']
    initial_quantity = data['initial_quantity']

    # 2. Validate business rules (e.g., SKU uniqueness)
    if Product.query.filter_by(sku=sku).first():
        return jsonify({"error": f"Product with SKU '{sku}' already exists"}), 409 # 409 is for Conflict

    # Start a single "all-or-nothing" transaction
    try:
        # 3. Create the product
        product = Product(
            name=name,
            sku=sku,
            price=price,
            warehouse_id=warehouse_id
        )
        db.session.add(product)

        # 4. Create the initial inventory
        # We need to save the product first to get its ID, so we use flush
        db.session.flush()

        inventory = Inventory(
            product_id=product.id,
            warehouse_id=warehouse_id,
            quantity=initial_quantity
        )
        db.session.add(inventory)

        # 5. Commit the transaction
        # Both product and inventory are saved together. If anything fails, it all rolls back.
        db.session.commit()

        return jsonify({
            "message": "Product created successfully",
            "product": {
                "id": product.id,
                "name": product.name,
                "sku": product.sku
            }
        }), 201 # 201 means "Created"

    except Exception as e:
        # If any error occurs, rollback the entire transaction
        db.session.rollback()
        # It's good practice to log the error, e.g., app.logger.error(e)
        return jsonify({"error": "An unexpected error occurred."}), 500



---

## Part 2: Database Design

This section outlines the proposed database schema, identifies missing requirements, and justifies the design choices.

### 1. Database Schema Design

#### `Companies` Table
Stores information about the businesses using the software.

| Column Name | Data Type | Description |
| :--- | :--- | :--- |
| `id` | `INTEGER` | A unique number to identify each company (Primary Key). |
| `name` | `VARCHAR(255)` | The official name of the company. |
| `contact_email`| `VARCHAR(255)` | The main email address for communication. Must be unique. |
| `created_at` | `TIMESTAMP` | The date and time when the company was added. |
| `updated_at` | `TIMESTAMP` | The date and time when the company's record was last updated. |

<br>

#### `Warehouses` Table
Each warehouse must be associated with a company.

| Column Name | Data Type | Description |
| :--- | :--- | :--- |
| `id` | `INTEGER` | A unique number to identify each warehouse (Primary Key). |
| `company_id` | `INTEGER` | Links this warehouse to a specific company (Foreign Key). |
| `name` | `VARCHAR(255)` | The name of the warehouse (e.g., "Main Warehouse"). |
| `location` | `TEXT` | The physical address of the warehouse. |
| `created_at` | `TIMESTAMP` | The date and time when the warehouse was added. |
| `updated_at` | `TIMESTAMP` | The date and time when the warehouse record was last updated. |

<br>

#### `Suppliers` Table
Stores information about product suppliers.

| Column Name | Data Type | Description |
| :--- | :--- | :--- |
| `id` | `INTEGER` | A unique number to identify each supplier (Primary Key). |
| `name` | `VARCHAR(255)` | The name of the supplier. |
| `contact_email`| `VARCHAR(255)` | The supplier's contact email for placing orders. |
| `contact_phone`| `VARCHAR(20)` | The supplier's contact phone number. |
| `created_at` | `TIMESTAMP` | The date and time when the supplier was added. |
| `updated_at` | `TIMESTAMP` | The date and time when the supplier record was last updated. |

<br>

#### `Products` Table
Stores details for each unique product.

| Column Name | Data Type | Description |
| :--- | :--- | :--- |
| `id` | `INTEGER` | A unique number to identify each product (Primary Key). |
| `supplier_id`| `INTEGER` | Links this product to its supplier (Foreign Key). |
| `name` | `VARCHAR(255)` | The name of the product. |
| `sku` | `VARCHAR(100)` | The Stock Keeping Unit. Must be unique across all products. |
| `price` | `DECIMAL(10, 2)` | The price of the product. |
| `is_bundle` | `BOOLEAN` | A flag (true/false) to indicate if this product is a bundle. |
| `created_at` | `TIMESTAMP` | The date and time when the product was added. |
| `updated_at` | `TIMESTAMP` | The date and time when the product record was last updated. |

<br>

#### `Inventory` Table
A linking table that tracks the quantity of a product in a specific warehouse.

| Column Name | Data Type | Description |
| :--- | :--- | :--- |
| `id` | `INTEGER` | A unique ID for this inventory record (Primary Key). |
| `product_id` | `INTEGER` | Links to a specific product (Foreign Key). |
| `warehouse_id` | `INTEGER` | Links to a specific warehouse (Foreign Key). |
| `quantity` | `INTEGER` | The current stock quantity of the product in that warehouse. |
| `updated_at` | `TIMESTAMP` | The date and time when the quantity was last updated. |

*Note: A unique constraint should be placed on the combination of (`product_id`, `warehouse_id`).*

<br>

#### `Product_Bundles` Table
Defines the contents of a bundle.

| Column Name | Data Type | Description |
| :--- | :--- | :--- |
| `bundle_product_id` | `INTEGER` | The ID of the main bundle product (Foreign Key). |
| `contained_product_id` | `INTEGER` | The ID of the individual product included in the bundle (Foreign Key). |
| `quantity` | `INTEGER` | The quantity of the contained product within the bundle. |

<br>

#### `Inventory_Log` Table
Tracks every change to inventory for auditing purposes.

| Column Name | Data Type | Description |
| :--- | :--- | :--- |
| `id` | `INTEGER` | A unique ID for the log entry (Primary Key). |
| `inventory_id` | `INTEGER` | Links to the specific inventory record being changed (Foreign Key). |
| `quantity_change` | `INTEGER` | The change in quantity (+10 for stock in, -2 for a sale). |
| `reason` | `VARCHAR(255)` | The reason for the change (e.g., "Sale #123", "Stock Intake"). |
| `created_at` | `TIMESTAMP` | The exact time the inventory change occurred. |

---

### 2. Gaps and Questions for the Product Team

The requirements are intentionally incomplete. Here are questions I would ask to clarify the needs:

1.  **User Roles:** Do we need a concept of "Users" and "Roles" (e.g., an Admin who can see all company data vs. a Warehouse Manager who can only see their own warehouse)?
2.  **Supplier Details:** What specific information is needed for a supplier besides their name and contact info? (e.g., address, payment terms).
3.  **Product Attributes:** Do products have other attributes we need to track, such as weight, dimensions, or customs information?
4.  **Pricing:** Will prices ever change? Do we need to handle different currencies or sales tax?
5.  **Inventory Tracking:** What specific events should trigger an `Inventory_Log` entry? Just sales and stock-ins, or also internal transfers between warehouses?

---

### 3. Design Decisions and Justification

* **Normalization:** I used linking tables (`Inventory`, `Product_Bundles`) to create a "many-to-many" relationship. This avoids data duplication. For example, instead of storing product details in the inventory table, I only store its `product_id`. This is efficient and reduces errors.
* **Keys and Relationships:** I used a unique `id` (Primary Key) in every table for clear identification. **Foreign Keys** (e.g., `company_id` in the `Warehouses` table) are used to enforce relationships between tables, ensuring data integrity.
* **Data Integrity:** I've specified **unique constraints** where necessary (e.g., on `product.sku` and the combination of `product_id` and `warehouse_id` in the `Inventory` table) to prevent invalid or duplicate data.
* **Auditing and Timestamps:** The `Inventory_Log` table provides a full audit trail of stock movements, which is critical for a system like this. The `created_at` and `updated_at` timestamps in all major tables are a best practice for debugging and data analysis.


---

## Part 3: API Implementation

This section provides the implementation for the low-stock alerts endpoint, including the necessary assumptions, the code itself, and a discussion of how edge cases are handled.

### 1. Assumptions Made

To implement this feature, I've made the following assumptions based on the incomplete requirements:

* **Database Access:** The code assumes it can query the database schema designed in Part 2 using an ORM like SQLAlchemy.
* **Low Stock Threshold:** I assume a `low_stock_threshold` column of type `INTEGER` exists on the `Products` table to define the alert level for each product.
* **Recent Sales Activity:** I am defining "recent sales activity" as any sale recorded within the **last 30 days**.
* **Sales Data:** I assume the existence of a `Sales` table that records which products are sold, in what quantity, and on what date. A simplified version would look like this:
    * `Sales` Table: `id`, `inventory_id`, `quantity_sold`, `sale_date`
* **Stockout Calculation:** The `days_until_stockout` is a predictive metric. I will assume it's calculated based on the average daily sales over the last 30 days. If there is no sales history, this will be `null`.

### 2. API Implementation

Here is the Python/Flask implementation for the endpoint. The code is heavily commented to explain the logic and how edge cases are handled directly within it.

```python
from flask import jsonify
from sqlalchemy import func, and_
from datetime import datetime, timedelta

# Assuming db models are defined for Company, Warehouse, Product, Inventory, Supplier, Sales
# from .models import db, Company, Warehouse, Product, Inventory, Supplier, Sales

@app.route('/api/companies/<int:company_id>/alerts/low-stock', methods=['GET'])
def get_low_stock_alerts(company_id):
    """
    Generates a list of low-stock alerts for a given company.
    """
    # EDGE CASE HANDLING 1: Company not found
    company = Company.query.get(company_id)
    if not company:
        return jsonify({"error": "Company not found"}), 404

    thirty_days_ago = datetime.utcnow() - timedelta(days=30)

    low_stock_items = db.session.query(
        Product,
        Inventory,
        Warehouse,
        Supplier
    ).join(
        Inventory, Product.id == Inventory.product_id
    ).join(
        Warehouse, Inventory.warehouse_id == Warehouse.id
    ).join(
        Supplier, Product.supplier_id == Supplier.id
    ).filter(
        Warehouse.company_id == company_id,
        Inventory.quantity <= Product.low_stock_threshold
    ).all()

    alerts = []
    # EDGE CASE HANDLING 2: No low-stock items found
    # If `low_stock_items` is empty, this loop is skipped, and an empty list is returned.

    for product, inventory, warehouse, supplier in low_stock_items:

        recent_sales = db.session.query(
            func.sum(Sales.quantity_sold)
        ).filter(
            Sales.inventory_id == inventory.id,
            Sales.sale_date >= thirty_days_ago
        ).scalar() or 0

        # EDGE CASE HANDLING 3: No recent sales
        # This 'if' statement skips products that don't have recent sales, per the business rule.
        if recent_sales > 0:
            days_until_stockout = None
            avg_daily_sales = recent_sales / 30.0
            
            # EDGE CASE HANDLING 4: Prevents division-by-zero for stockout calculation
            if avg_daily_sales > 0:
                days_until_stockout = int(inventory.quantity / avg_daily_sales)

            alert = {
                "product_id": product.id,
                "product_name": product.name,
                "sku": product.sku,
                "warehouse_id": warehouse.id,
                "warehouse_name": warehouse.name,
                "current_stock": inventory.quantity,
                "threshold": product.low_stock_threshold,
                "days_until_stockout": days_until_stockout,
                "supplier": {
                    "id": supplier.id,
                    "name": supplier.name,
                    "contact_email": supplier.contact_email
                }
            }
            alerts.append(alert)

    return jsonify({
        "alerts": alerts,
        "total_alerts": len(alerts)
    })



