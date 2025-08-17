# Bynry Inc. - Backend Intern Case Study Submission

My submission for the backend developer case study.

---

## Part 1: Code Review & Debugging

### Issues Identified

1.  **Missing Error Handling:** The code directly accesses `data['name']` and other keys without checking if they exist. If a field is missing in the request, the application will crash with a `KeyError`.
2.  **Not a Single Database Transaction:** There are two separate `db.session.commit()` calls. If the first commit succeeds but the second one fails, the database will be left in an inconsistent state with a product record that has no corresponding inventory.
3.  **No Validation of Business Rules:** The code does not check if a product with the same SKU already exists before creating a new one. [cite_start]This violates the business rule that SKUs must be unique across the platform[cite: 36].
4.  [cite_start]**No Handling of Optional Fields:** The case study mentions that some fields might be optional[cite: 37], but the code treats all fields as mandatory, making the endpoint inflexible.

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
