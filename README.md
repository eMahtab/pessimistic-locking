# Pessimistic Locking


# Step 1 : Create product_inventory table under test database in MySQL
```sql
CREATE TABLE `test`.`product_inventory` (
  `product_id` INT NOT NULL,
  `product_name` VARCHAR(500) NULL,
  `quantity` INT NULL,
  PRIMARY KEY (`product_id`));
```
