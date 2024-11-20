# Pessimistic Locking
Pessimistic locking is used when we want **only one thread (out of multiple threads) at a time to update the database record (a row in a database table)**.
The thread which gets the lock over table row, gets the chance to update the record, other threads which want to update the same record waits for the, lock over table row to be released.
A thread releases the lock by commiting the transaction or by rollback.

Below is a simple example which explains the pessimistic locking, where multiple threads (5 in this case) try to update the quantity in product_inventory table for product_id 1.
Only one thread at a time update the quantity for product_id 1. Other threads wait until, the thread which have the lock releases the lock over table row with product_id 1.

Also the program doesn't allow the updateProductInventory() operation where the update would result in setting a value which is less than zero for quantity. It throws an **InsufficientProductInventoryException**.

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

**The program throws InsufficientProductInventoryException if we try to set the product quanity to a negative value (less than zero).**

!["Eclipse Project"](eclipse-project.png?raw=true)

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

public class InsufficientProductInventoryException extends Exception{
	private static final long serialVersionUID = 1L;

	public InsufficientProductInventoryException(String message) {
		super(message);
    }
}
```

### Code Execution Outputs :
**Note that at the start of every run, I manually set the quanity of product_id 1 to 5 with an update statement**
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
By the above output results, its clear that regardless of the order of threads being successful in getting the lock over table row, **no two threads are able to update the product quantity for the same product at the same time**. This ensures consistency of the product quantity, which is very important in any real world application, and guarantees that each thread will see the correct value of quantity before updating value of quantity. 

You would never want that multiple users bought the same product when the inventory had just 1 quantity of that product, or multiple users being able to book the exact same seat when it really is just one physical seat. Pessimistic locking is a reliable mechanism to enforce data integrity in high-contention scenarios.
