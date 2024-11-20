# Pessimistic Locking


# Step 1 : Create product_inventory table under test database in MySQL
```sql
CREATE TABLE `test`.`product_inventory` (
  `product_id` INT NOT NULL,
  `product_name` VARCHAR(500) NULL,
  `quantity` INT NULL,
  PRIMARY KEY (`product_id`));
```
# Step 2 : Insert a product record in the product_inventory table
```sql
INSERT INTO product_inventory(product_id,product_name,quantity)
VALUES (1,'Some Popular Product',5);
```
