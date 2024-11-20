# Pessimistic Locking


# Step 1 : Create product_inventory table under test database in MySQL
```sql
CREATE TABLE product_inventory (
    product_id INT PRIMARY KEY,
    product_name VARCHAR(500),
    quantity INT
);
```
