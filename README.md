# Pessimistic Locking


## Step 1 : Create product_inventory table under test database in MySQL
```sql
CREATE TABLE `test`.`product_inventory` (
  `product_id` INT NOT NULL,
  `product_name` VARCHAR(500) NULL,
  `quantity` INT NULL,
  PRIMARY KEY (`product_id`));
```
## Step 2 : Insert a product record in the product_inventory table
We insert a product with product_id as 1 and quantity set to 5
```sql
INSERT INTO product_inventory(product_id,product_name,quantity)
VALUES (1,'Some Popular Product',5);
```

### Step 3 : Write the code, making sure no two threads update the same product at the same time
In the below Java program we simulate 5 different threads trying to update the quanity for the product whose product_id is 1.

Note the Prepared statement `selectForUpdate`, by adding `FOR UPDATE` at the end of SELECT statement, we are trying to get the lock for the update on the specific row (in this case product with product_id 1).
```java
String selectForUpdate = "SELECT quantity FROM "+ PRODUCT_INVENTORY_TABLE +" WHERE product_id = ? FOR UPDATE";
```

```java
package net.mahtabalam;

import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.PreparedStatement;
import java.sql.ResultSet;

public class Test {

    private static final String DB_URL = "jdbc:mysql://localhost:3306/test?useSSL=false";
    private static final String DB_USER = "root";
    private static final String DB_PASSWORD = "YOUR_DB_USER_PASSWORD";
    private static final String PRODUCT_INVENTORY_TABLE = "product_inventory";

    public static void main(String[] args) {
        // Simulating 5 threads trying to update the same product
        Thread t1 = new Thread(() -> updateProductInventory(1, -1));
        Thread t2 = new Thread(() -> updateProductInventory(1, -2));
        Thread t3 = new Thread(() -> updateProductInventory(1, -2));
        Thread t4 = new Thread(() -> updateProductInventory(1, 2));
        Thread t5 = new Thread(() -> updateProductInventory(1, -1));

        t1.start(); t2.start(); t3.start(); t4.start(); t5.start();
    }

    private static void updateProductInventory(int productId, int quantityChange) {
    	System.out.println(Thread.currentThread().getName()+" entered updateProductInventory() execution :"+quantityChange);
    	
        try (Connection connection = DriverManager.getConnection(DB_URL, DB_USER, DB_PASSWORD)) {
            connection.setAutoCommit(false); // Begin transaction

            // Lock the specific row
            String selectForUpdate = "SELECT quantity FROM "+ PRODUCT_INVENTORY_TABLE +" WHERE product_id = ? FOR UPDATE";
            try (PreparedStatement selectStmt = connection.prepareStatement(selectForUpdate)) {
                selectStmt.setInt(1, productId);
                ResultSet rs = selectStmt.executeQuery();

                if (rs.next()) {
                    int currentQuantity = rs.getInt("quantity");
                    int newQuantity = currentQuantity + quantityChange;
                    // Ensure quantity does not go below zero
                    if (newQuantity < 0) {
                        throw new InsufficientProductInventoryException(
                        		Thread.currentThread().getName()+" Insufficient inventory for product ID " + productId + ". " +
                                "Current quantity: " + currentQuantity + ", attempted change: " + quantityChange);
                    }

                    // Update the row with new quantity
                    String updateQuery = "UPDATE "+ PRODUCT_INVENTORY_TABLE +" SET quantity = ? WHERE product_id = ?";
                    try (PreparedStatement updateStmt = connection.prepareStatement(updateQuery)) {
                        updateStmt.setInt(1, newQuantity);
                        updateStmt.setInt(2, productId);
                        updateStmt.executeUpdate();
                    }

                    System.out.println(Thread.currentThread().getName() + " updated product " + productId + 
                                       " to quantity: " + newQuantity);
                } else {
                    System.out.println("Product with ID " + productId + " not found.");
                }

                connection.commit(); // Commit transaction
            } catch (InsufficientProductInventoryException e) {
                connection.rollback(); // Rollback transaction
                throw e; // Rethrow the exception to notify the caller
            } catch (Exception e) {
                connection.rollback(); // Rollback transaction in case of an error
                System.err.println(Thread.currentThread().getName() + " encountered an error: " + e.getMessage());
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}

```

### Code Execution Outputs :
Note that at the start of every run the quanity of product_id 1 is reset to 5 with an update statement 
```sql
UPDATE product_inventory set quantity= 5 where product_id = 1
```
#### Output 1:
```
Thread-4 entered updateProductInventory() execution :-1
Thread-0 entered updateProductInventory() execution :-1
Thread-1 entered updateProductInventory() execution :-2
Thread-3 entered updateProductInventory() execution :2
Thread-2 entered updateProductInventory() execution :-2
Thread-1 updated product 1 to quantity: 3
Thread-2 updated product 1 to quantity: 1
Thread-3 updated product 1 to quantity: 3
Thread-0 updated product 1 to quantity: 2
Thread-4 updated product 1 to quantity: 1
```
#### Output 2:
```
Thread-4 entered updateProductInventory() execution :-1
Thread-1 entered updateProductInventory() execution :-2
Thread-2 entered updateProductInventory() execution :-2
Thread-0 entered updateProductInventory() execution :-1
Thread-3 entered updateProductInventory() execution :2
Thread-1 updated product 1 to quantity: 3
Thread-3 updated product 1 to quantity: 5
Thread-0 updated product 1 to quantity: 4
Thread-4 updated product 1 to quantity: 3
Thread-2 updated product 1 to quantity: 1
```

#### Output 3:
```
Thread-4 entered updateProductInventory() execution :-1
Thread-3 entered updateProductInventory() execution :2
Thread-1 entered updateProductInventory() execution :-2
Thread-2 entered updateProductInventory() execution :-2
Thread-0 entered updateProductInventory() execution :-1
Thread-0 updated product 1 to quantity: 4
Thread-2 updated product 1 to quantity: 2
Thread-4 updated product 1 to quantity: 1
net.mahtabalam.InsufficientProductInventoryException: Thread-1 Insufficient inventory for product ID 1. Current quantity: 1, attempted change: -2
	at net.mahtabalam.Test.updateProductInventory(Test.java:44)
	at net.mahtabalam.Test.lambda$1(Test.java:18)
	at java.base/java.lang.Thread.run(Thread.java:1575)
Thread-3 updated product 1 to quantity: 3
```
