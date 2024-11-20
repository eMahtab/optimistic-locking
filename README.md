# Optimistic Locking


## Step 1 : Create product_inventory table under inventory database in MySQL
```sql
CREATE TABLE `inventory`.`product_inventory` (
  `product_id` INT NOT NULL,
  `product_name` VARCHAR(500) NULL,
  `quantity` INT NOT NULL,
  `version` BIGINT NOT NULL DEFAULT 0,
  PRIMARY KEY (`product_id`));
```
## Step 2 : Insert a product record in the product_inventory table
```sql
INSERT INTO product_inventory(product_id,product_name,quantity)
VALUES (1,'Some Popular Product',5);
```

### Step 3 : Write the code, allowing multiple threads to try to update the same record e.g. same product_id
```java
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;

public class Test {

    private static final String DB_URL = "jdbc:mysql://localhost:3306/inventory?useSSL=false";
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
        System.out.println(Thread.currentThread().getName() + " entered updateProductInventory() execution: " + quantityChange);

        try (Connection connection = DriverManager.getConnection(DB_URL, DB_USER, DB_PASSWORD)) {
            connection.setAutoCommit(false); // Begin transaction

            // Select the row with version
            String selectQuery = "SELECT quantity, version FROM " + PRODUCT_INVENTORY_TABLE + " WHERE product_id = ?";
            try (PreparedStatement selectStmt = connection.prepareStatement(selectQuery)) {
                selectStmt.setInt(1, productId);
                ResultSet rs = selectStmt.executeQuery();

                if (rs.next()) {
                    int currentQuantity = rs.getInt("quantity");
                    long version = rs.getLong("version"); // BIGINT maps to long in Java
                    int newQuantity = currentQuantity + quantityChange;

                    // Ensure quantity does not go below zero
                    if (newQuantity < 0) {
                        throw new InsufficientProductInventoryException(
                                Thread.currentThread().getName() + " Insufficient inventory for product ID " + productId + ". " +
                                "Current quantity: " + currentQuantity + ", attempted change: " + quantityChange);
                    }

                    // Try to update the row with optimistic lock
                    String updateQuery = "UPDATE " + PRODUCT_INVENTORY_TABLE + " SET quantity = ?, version = version + 1 " +
                                         "WHERE product_id = ? AND version = ?";
                    try (PreparedStatement updateStmt = connection.prepareStatement(updateQuery)) {
                        updateStmt.setInt(1, newQuantity);
                        updateStmt.setInt(2, productId);
                        updateStmt.setLong(3, version); // Use long for BIGINT column

                        int rowsAffected = updateStmt.executeUpdate();
                        if (rowsAffected == 0) {
                            throw new OptimisticLockException(
                                    Thread.currentThread().getName() + " failed to update product ID " + productId + 
                                    " due to version mismatch caused by concurrent modification.");
                        }

                        System.out.println(Thread.currentThread().getName() + " updated product " + productId +
                                           " to quantity: " + newQuantity);
                    }
                } else {
                    System.out.println("Product with ID " + productId + " not found.");
                }

                connection.commit(); // Commit transaction
            } catch (InsufficientProductInventoryException | OptimisticLockException e) {
                connection.rollback(); // Rollback transaction
                System.err.println(e.getMessage());
            } catch (SQLException e) {
                connection.rollback(); // Rollback transaction on SQLException
                System.err.println(Thread.currentThread().getName() + " encountered a database error: " + e.getMessage());
            } catch (Exception e) {
                connection.rollback(); // Rollback transaction on generic errors
                System.err.println(Thread.currentThread().getName() + " encountered an error: " + e.getMessage());
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}

```

## Code Execution Outputs :
**Note that at the start of every run, I manually set the quanity of product_id 1 to 5 with an update statement**
```sql
UPDATE product_inventory set quantity= 5 where product_id = 1
```

## Output 1 :
```
Thread-1 entered updateProductInventory() execution: -2
Thread-0 entered updateProductInventory() execution: -1
Thread-3 entered updateProductInventory() execution: 2
Thread-2 entered updateProductInventory() execution: -2
Thread-4 entered updateProductInventory() execution: -1
Thread-3 updated product 1 to quantity: 7
Thread-1 failed to update product ID 1 due to version mismatch caused by concurrent modification.
Thread-2 failed to update product ID 1 due to version mismatch caused by concurrent modification.
Thread-0 failed to update product ID 1 due to version mismatch caused by concurrent modification.
Thread-4 failed to update product ID 1 due to version mismatch caused by concurrent modification.
```
## Output 2 :
```
Thread-2 entered updateProductInventory() execution: -2
Thread-0 entered updateProductInventory() execution: -1
Thread-4 entered updateProductInventory() execution: -1
Thread-3 entered updateProductInventory() execution: 2
Thread-1 entered updateProductInventory() execution: -2
Thread-0 updated product 1 to quantity: 4
Thread-4 failed to update product ID 1 due to version mismatch caused by concurrent modification.
Thread-3 failed to update product ID 1 due to version mismatch caused by concurrent modification.
Thread-2 failed to update product ID 1 due to version mismatch caused by concurrent modification.
Thread-1 failed to update product ID 1 due to version mismatch caused by concurrent modification.
```
From the above two execution outputs, we see only one thread was successfully able to update the quantity for product_id 1. All other four threads failed to update the quantity, due to version mismatch,
which was caused by the updation of version in the database table by another thread. We can't guarantee how many threads would be able to successfully update the quantity, that depends on the order of execution of threads and execution delay at runtime.

But we can see that many threads might fail updating the quantity in database table due to version mismatch, which is caused by the updation of version in the database table by another thread.
