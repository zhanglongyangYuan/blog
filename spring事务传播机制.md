Spring框架中的事务传播机制（Transaction Propagation）定义了当一个带有事务配置的方法（我们称之为外部方法）调用另一个同样可能带有事务配置的方法（我们称之为内部方法）时，事务是如何在这些方法间传播或如何创建和管理的。这对于构建复杂的多层应用服务至关重要。

Spring 主要通过 `@Transactional` 注解的 `propagation` 属性来指定传播行为。以下是Spring支持的事务传播机制，以及相应的说明和举例：

**1. `Propagation.REQUIRED` (默认值)**

* **说明**: 如果当前存在一个事务，则加入该事务；如果当前没有事务，则创建一个新的事务。这是最常用的传播行为。
* **举例场景**:
  一个服务 `OrderService` 的 `placeOrder` 方法需要开启事务。它内部调用了 `ProductService` 的 `decreaseStock` 方法和 `AccountService` 的 `deductBalance` 方法。我们希望这两个内部方法都在 `placeOrder` 的同一个事务中执行。如果 `deductBalance` 失败，`decreaseStock` 的操作也应该回滚。

    ```java
    // OrderService.java
    import org.springframework.beans.factory.annotation.Autowired;
    import org.springframework.stereotype.Service;
    import org.springframework.transaction.annotation.Propagation;
    import org.springframework.transaction.annotation.Transactional;

    @Service
    public class OrderService {

        @Autowired
        private ProductService productService;
        @Autowired
        private AccountService accountService;

        // 使用默认的 REQUIRED
        @Transactional // 等同于 @Transactional(propagation = Propagation.REQUIRED)
        public void placeOrder(String productId, String userId, int quantity, double amount) {
            System.out.println("OrderService.placeOrder: Starting transaction (or joining existing).");
            productService.decreaseStock(productId, quantity); // 加入 placeOrder 的事务
            accountService.deductBalance(userId, amount);     // 加入 placeOrder 的事务
            System.out.println("OrderService.placeOrder: Order placed successfully.");
            // 如果 deductBalance 抛出运行时异常，decreaseStock 的操作也会回滚
        }
    }

    // ProductService.java
    @Service
    public class ProductService {
        // 即使这里也声明了 @Transactional，它也会加入外部事务
        @Transactional(propagation = Propagation.REQUIRED)
        public void decreaseStock(String productId, int quantity) {
            System.out.println("ProductService.decreaseStock: Decreasing stock for product " + productId + ". In transaction? " + org.springframework.transaction.support.TransactionSynchronizationManager.isActualTransactionActive());
            // 数据库操作：减少库存
            if (quantity > 100) { // 模拟库存不足的场景
                 System.out.println("ProductService.decreaseStock: Not enough stock for " + productId);
                 throw new RuntimeException("Stock not enough for product: " + productId);
            }
            System.out.println("ProductService.decreaseStock: Stock decreased.");
        }
    }

    // AccountService.java
    @Service
    public class AccountService {
        @Transactional(propagation = Propagation.REQUIRED)
        public void deductBalance(String userId, double amount) {
            System.out.println("AccountService.deductBalance: Deducting balance for user " + userId + ". In transaction? " + org.springframework.transaction.support.TransactionSynchronizationManager.isActualTransactionActive());
            // 数据库操作：扣减余额
            if (amount > 500) { // 模拟余额不足
                System.out.println("AccountService.deductBalance: Insufficient balance for user " + userId);
                throw new RuntimeException("Insufficient balance for user: " + userId);
            }
            System.out.println("AccountService.deductBalance: Balance deducted.");
        }
    }

    // 调用方
    // orderService.placeOrder("P123", "U456", 10, 200.0); // 正常流程
    // orderService.placeOrder("P789", "U111", 5, 600.0);  // 触发 AccountService 异常，整个事务回滚
    ```
  在这个例子中，如果 `deductBalance` 抛出异常，由于它们都在同一个事务中，`decreaseStock` 的操作也会被回滚。如果 `placeOrder` 本身没有被一个更大的事务调用，它会自己创建一个新事务。

**2. `Propagation.REQUIRES_NEW`**

* **说明**: 总是创建一个新的事务。如果当前存在一个事务，则将当前事务挂起（suspend），执行完新事务后再恢复（resume）外部事务。
* **举例场景**:
  假设有一个日志记录服务 `AuditLogService` 的 `logAction` 方法。无论外部方法是否在事务中，或者外部事务是否成功，我们都希望日志记录操作能够独立提交。

    ```java
    // AuditLogService.java
    import org.springframework.stereotype.Service;
    import org.springframework.transaction.annotation.Propagation;
    import org.springframework.transaction.annotation.Transactional;

    @Service
    public class AuditLogService {

        @Transactional(propagation = Propagation.REQUIRES_NEW)
        public void logAction(String actionDetails) {
            System.out.println("AuditLogService.logAction: Starting NEW transaction for logging.");
            // 数据库操作：插入日志记录
            System.out.println("AuditLogService.logAction: Logging action: " + actionDetails + ". In transaction? " + org.springframework.transaction.support.TransactionSynchronizationManager.isActualTransactionActive());
            // 即使外部事务回滚，这条日志也会被提交（除非logAction本身抛出异常）
            if (actionDetails.contains("CRITICAL_ERROR_LOG")) {
                 System.out.println("AuditLogService.logAction: Simulating error during logging critical error.");
                 throw new RuntimeException("Failed to log critical error details."); // 这个新事务会回滚
            }
            System.out.println("AuditLogService.logAction: Log action committed (or will be upon method completion).");
        }
    }

    // MainService.java
    @Service
    public class MainService {
        @Autowired
        private AuditLogService auditLogService;

        @Transactional(propagation = Propagation.REQUIRED)
        public void performMainOperation(String data) {
            System.out.println("MainService.performMainOperation: Starting main operation. In transaction? " + org.springframework.transaction.support.TransactionSynchronizationManager.isActualTransactionActive());
            // ... 主要业务逻辑 ...
            System.out.println("MainService.performMainOperation: Main operation processing data: " + data);

            try {
                auditLogService.logAction("Main operation started for: " + data); // 开启新事务

                if ("trigger_error".equals(data)) {
                    System.out.println("MainService.performMainOperation: Simulating error in main operation.");
                    throw new RuntimeException("Error in main operation!");
                }

                auditLogService.logAction("Main operation completed for: " + data); // 开启另一个新事务
            } catch (Exception e) {
                System.err.println("MainService.performMainOperation: Caught exception: " + e.getMessage());
                auditLogService.logAction("Main operation FAILED for: " + data + ", Error: " + e.getMessage().substring(0, Math.min(e.getMessage().length(), 20))); // 仍然会尝试开启新事务记录失败
                throw e; // 重新抛出，让主事务回滚
            }
            System.out.println("MainService.performMainOperation: Main operation finished.");
        }
    }

    // 调用方
    // mainService.performMainOperation("test_data"); // 外部事务成功，日志事务也成功
    // mainService.performMainOperation("trigger_error"); // 外部事务回滚，但成功的日志事务已提交，失败日志的事务也可能提交（如果它不抛异常）
    ```
  即使 `performMainOperation` 因为 `trigger_error` 而回滚，`auditLogService.logAction("Main operation started for: trigger_error")` 已经在一个独立的事务中成功提交了。如果 `logAction` 自身发生异常，则它自己的新事务会回滚，但不影响外部 `performMainOperation` 的事务。

**3. `Propagation.SUPPORTS`**

* **说明**: 如果当前存在事务，则加入该事务；如果当前没有事务，则以非事务的方式执行。
* **举例场景**:
  一个查询服务 `ReadOnlyService` 的 `getData` 方法。如果调用它的方法有事务，它就在那个事务里运行；如果没有，它也不需要自己开启事务，因为只是读取数据。

    ```java
    // ReadOnlyService.java
    import org.springframework.stereotype.Service;
    import org.springframework.transaction.annotation.Propagation;
    import org.springframework.transaction.annotation.Transactional;

    @Service
    public class ReadOnlyService {

        @Transactional(propagation = Propagation.SUPPORTS)
        public String getData(String key) {
            System.out.println("ReadOnlyService.getData: Getting data for key: " + key + ". In transaction? " + org.springframework.transaction.support.TransactionSynchronizationManager.isActualTransactionActive());
            // 数据库只读操作
            return "Data for " + key;
        }
    }

    // CallerService.java
    @Service
    public class CallerService {
        @Autowired
        private ReadOnlyService readOnlyService;

        @Transactional(propagation = Propagation.REQUIRED) // 外部有事务
        public void processWithTransaction() {
            System.out.println("CallerService.processWithTransaction: Starting with transaction.");
            String data = readOnlyService.getData("some_key_tx"); // readOnlyService.getData() 会加入当前事务
            System.out.println("CallerService.processWithTransaction: Received data: " + data);
        }

        public void processWithoutTransaction() { // 外部没有事务
            System.out.println("CallerService.processWithoutTransaction: Starting without transaction.");
            String data = readOnlyService.getData("some_key_no_tx"); // readOnlyService.getData() 会以非事务方式执行
            System.out.println("CallerService.processWithoutTransaction: Received data: " + data);
        }
    }

    // 调用方
    // callerService.processWithTransaction();
    // callerService.processWithoutTransaction();
    ```
  当 `processWithTransaction` 调用 `getData` 时，`getData` 会在 `processWithTransaction` 的事务上下文中运行。
  当 `processWithoutTransaction` 调用 `getData` 时，`getData` 会在没有事务的情况下运行。

**4. `Propagation.NOT_SUPPORTED`**

* **说明**: 总是以非事务方式执行。如果当前存在事务，则将当前事务挂起。
* **举例场景**:
  一个后台任务 `BackgroundTaskService` 的 `performNonTransactionalTask` 方法，该任务不应该参与任何调用者可能存在的事务，例如它调用一个外部的、不支持事务的API。

    ```java
    // BackgroundTaskService.java
    import org.springframework.stereotype.Service;
    import org.springframework.transaction.annotation.Propagation;
    import org.springframework.transaction.annotation.Transactional;

    @Service
    public class BackgroundTaskService {

        @Transactional(propagation = Propagation.NOT_SUPPORTED)
        public void performNonTransactionalTask(String taskName) {
            System.out.println("BackgroundTaskService.performNonTransactionalTask: Performing task: " + taskName + ". In transaction? " + org.springframework.transaction.support.TransactionSynchronizationManager.isActualTransactionActive());
            // 执行一些不需要事务的操作，比如调用一个外部HTTP服务
            System.out.println("BackgroundTaskService.performNonTransactionalTask: Task " + taskName + " completed (non-transactional).");
        }
    }

    // MainBusinessService.java
    @Service
    public class MainBusinessService {
        @Autowired
        private BackgroundTaskService backgroundTaskService;

        @Transactional(propagation = Propagation.REQUIRED)
        public void doBusinessLogicAndBackgroundTask(String businessData, String taskName) {
            System.out.println("MainBusinessService.doBusinessLogicAndBackgroundTask: Starting business logic with transaction. In transaction? " + org.springframework.transaction.support.TransactionSynchronizationManager.isActualTransactionActive());
            // ... 核心业务数据库操作 ...
            System.out.println("MainBusinessService.doBusinessLogicAndBackgroundTask: Business logic part 1 done.");

            backgroundTaskService.performNonTransactionalTask(taskName); // 当前事务会被挂起，这个方法以非事务方式执行

            // ... 核心业务数据库操作，继续在原事务中 ...
            System.out.println("MainBusinessService.doBusinessLogicAndBackgroundTask: Business logic part 2 done.");
            if (businessData.equals("fail_after_bgtask")) {
                throw new RuntimeException("Simulated failure after background task.");
            }
        }
    }

    // 调用方
    // mainBusinessService.doBusinessLogicAndBackgroundTask("data1", "task1"); // 主事务成功，非事务任务也执行
    // mainBusinessService.doBusinessLogicAndBackgroundTask("fail_after_bgtask", "task2"); // 主事务回滚，但非事务任务已执行且不受影响
    ```
  即使 `doBusinessLogicAndBackgroundTask` 事务最终回滚，`performNonTransactionalTask` 的执行（因为它以非事务方式运行）也不会受影响。

**5. `Propagation.MANDATORY`**

* **说明**: 必须在一个已存在的事务中执行，否则会抛出异常。它不会自己创建事务。
* **举例场景**:
  一个核心更新操作 `CoreUpdateService` 的 `updateCriticalData` 方法，这个方法设计为必须被一个外部事务包裹，以确保数据一致性。

    ```java
    // CoreUpdateService.java
    import org.springframework.stereotype.Service;
    import org.springframework.transaction.annotation.Propagation;
    import org.springframework.transaction.annotation.Transactional;

    @Service
    public class CoreUpdateService {

        @Transactional(propagation = Propagation.MANDATORY)
        public void updateCriticalData(String dataId, String newData) {
            System.out.println("CoreUpdateService.updateCriticalData: Updating critical data for " + dataId + ". In transaction? " + org.springframework.transaction.support.TransactionSynchronizationManager.isActualTransactionActive());
            // 数据库更新操作
            System.out.println("CoreUpdateService.updateCriticalData: Data " + dataId + " updated to " + newData);
        }
    }

    // OuterService.java
    @Service
    public class OuterService {
        @Autowired
        private CoreUpdateService coreUpdateService;

        @Transactional(propagation = Propagation.REQUIRED) // 提供事务
        public void processWithMandatory(String id, String data) {
            System.out.println("OuterService.processWithMandatory: Calling mandatory method within a transaction.");
            coreUpdateService.updateCriticalData(id, data); // 正常执行
            System.out.println("OuterService.processWithMandatory: Mandatory method call successful.");
        }

        public void processWithMandatoryWithoutTx(String id, String data) {
            System.out.println("OuterService.processWithMandatoryWithoutTx: Calling mandatory method WITHOUT a transaction.");
            try {
                coreUpdateService.updateCriticalData(id, data); // 将会抛出异常
            } catch (Exception e) {
                System.err.println("OuterService.processWithMandatoryWithoutTx: Error calling mandatory method: " + e.getMessage());
            }
        }
    }

    // 调用方
    // outerService.processWithMandatory("D001", "NewValueForD001");
    // outerService.processWithMandatoryWithoutTx("D002", "NewValueForD002"); // 会看到异常信息
    ```
  如果直接调用 `coreUpdateService.updateCriticalData` 或者从一个没有事务的方法中调用它，Spring会抛出 `IllegalTransactionStateException`。

**6. `Propagation.NEVER`**

* **说明**: 总是以非事务方式执行。如果当前存在事务，则抛出异常。
* **举例场景**:
  一个工具方法 `UtilService` 的 `executeUtilityTask`，它执行的操作绝对不能在事务中运行，例如，它可能与一些不支持事务回滚的外部资源交互，如果在事务中执行可能会导致状态不一致。

    ```java
    // UtilService.java
    import org.springframework.stereotype.Service;
    import org.springframework.transaction.annotation.Propagation;
    import org.springframework.transaction.annotation.Transactional;

    @Service
    public class UtilService {

        @Transactional(propagation = Propagation.NEVER)
        public void executeUtilityTask(String taskInfo) {
            System.out.println("UtilService.executeUtilityTask: Executing utility task: " + taskInfo + ". In transaction? " + org.springframework.transaction.support.TransactionSynchronizationManager.isActualTransactionActive());
            // 执行一些绝对不能在事务中的操作
            System.out.println("UtilService.executeUtilityTask: Utility task " + taskInfo + " done (non-transactional).");
        }
    }

    // AnotherService.java
    @Service
    public class AnotherService {
        @Autowired
        private UtilService utilService;

        @Transactional(propagation = Propagation.REQUIRED) // 提供事务
        public void callNeverFromTransaction(String info) {
            System.out.println("AnotherService.callNeverFromTransaction: Calling NEVER method from within a transaction.");
            try {
                utilService.executeUtilityTask(info); // 将会抛出异常
            } catch (Exception e) {
                System.err.println("AnotherService.callNeverFromTransaction: Error calling NEVER method: " + e.getMessage());
            }
        }

        public void callNeverWithoutTransaction(String info) { // 没有事务
            System.out.println("AnotherService.callNeverWithoutTransaction: Calling NEVER method without a transaction.");
            utilService.executeUtilityTask(info); // 正常执行
            System.out.println("AnotherService.callNeverWithoutTransaction: NEVER method call successful.");
        }
    }

    // 调用方
    // anotherService.callNeverWithoutTransaction("utility_task_1");
    // anotherService.callNeverFromTransaction("utility_task_2"); // 会看到异常信息
    ```
  如果 `executeUtilityTask` 从一个有事务的方法（如 `callNeverFromTransaction`）中被调用，Spring会抛出 `IllegalTransactionStateException`。

**7. `Propagation.NESTED`**

* **说明**:
    * 如果当前存在一个事务，则创建一个嵌套事务（savepoint）。嵌套事务是外部事务的一部分，它有自己的作用范围。嵌套事务可以独立于外部事务进行提交或回滚。
    * 如果嵌套事务回滚，它会回滚到它开始执行前的保存点，外部事务可以捕获这个回滚并继续执行，或者选择也回滚。
    * 如果外部事务回滚，则嵌套事务的所有操作（即使已经提交到保存点）也会被回滚。
    * 如果当前没有事务，则其行为类似于 `REQUIRED`（即创建一个新的主事务）。
* **重要**: `NESTED` 传播行为依赖于JDBC驱动和数据库对保存点（savepoints）的支持。不是所有数据库都支持。如果不支持，其行为可能会退化为 `REQUIRED`。
* **举例场景**:
  在 `OrderService` 的 `createOrder` 方法中，尝试为订单的每个商品项（`OrderItem`）单独处理库存和积分。如果某个商品项处理失败（例如库存不足），我们希望只回滚该商品项的操作，并记录下来，然后继续尝试处理其他商品项，而不是让整个订单创建失败。

    ```java
    // OrderItemService.java
    import org.springframework.stereotype.Service;
    import org.springframework.transaction.annotation.Propagation;
    import org.springframework.transaction.annotation.Transactional;

    @Service
    public class OrderItemService {

        @Transactional(propagation = Propagation.NESTED)
        public boolean processOrderItem(String itemId, int quantity) {
            System.out.println("OrderItemService.processOrderItem: Processing item " + itemId + " with NESTED transaction. In transaction? " + org.springframework.transaction.support.TransactionSynchronizationManager.isActualTransactionActive());
            // 模拟库存检查和扣减
            if ("ITEM_FAIL".equals(itemId) && quantity > 5) {
                System.out.println("OrderItemService.processOrderItem: Failed to process item " + itemId + " due to insufficient stock (simulated).");
                throw new RuntimeException("Stock error for " + itemId); // 这个嵌套事务会回滚
            }
            // 模拟积分增加
            System.out.println("OrderItemService.processOrderItem: Item " + itemId + " processed successfully (nested).");
            return true;
        }
    }

    // MainOrderService.java
    import org.springframework.beans.factory.annotation.Autowired;
    import org.springframework.stereotype.Service;
    import org.springframework.transaction.annotation.Propagation;
    import org.springframework.transaction.annotation.Transactional;
    import java.util.List;
    import java.util.ArrayList;

    @Service
    public class MainOrderService {
        @Autowired
        private OrderItemService orderItemService;

        @Transactional(propagation = Propagation.REQUIRED) // 主事务
        public void createOrderWithNestedItems(List<String> itemIds) {
            System.out.println("MainOrderService.createOrderWithNestedItems: Starting main order transaction.");
            List<String> processedItems = new ArrayList<>();
            List<String> failedItems = new ArrayList<>();

            for (String itemId : itemIds) {
                try {
                    // 每个item的处理都在一个嵌套事务中
                    orderItemService.processOrderItem(itemId, itemId.equals("ITEM_FAIL") ? 10 : 2);
                    processedItems.add(itemId);
                    System.out.println("MainOrderService: Successfully processed item " + itemId + " in nested transaction.");
                } catch (Exception e) {
                    // 捕获嵌套事务的回滚异常
                    System.err.println("MainOrderService: Failed to process item " + itemId + " in nested transaction: " + e.getMessage() + ". Main transaction continues.");
                    failedItems.add(itemId);
                    // 主事务可以选择不回滚，继续处理其他项
                }
            }

            System.out.println("MainOrderService.createOrderWithNestedItems: Order processing summary:");
            System.out.println("Successfully processed items: " + processedItems);
            System.out.println("Failed items (rolled back individually): " + failedItems);

            if (!failedItems.isEmpty() && itemIds.contains("FAIL_ENTIRE_ORDER_IF_ANY_ITEM_FAILS")) {
                System.out.println("MainOrderService: At least one item failed, and policy is to fail entire order. Rolling back main transaction.");
                throw new RuntimeException("Main order failed due to item processing errors."); // 导致主事务回滚
            }
            System.out.println("MainOrderService.createOrderWithNestedItems: Main order transaction completing.");
        }
    }

    // 调用方
    // List<String> items1 = List.of("ITEM_A", "ITEM_B", "ITEM_C");
    // mainOrderService.createOrderWithNestedItems(items1); // 所有item成功，主事务提交

    // List<String> items2 = List.of("ITEM_X", "ITEM_FAIL", "ITEM_Y");
    // mainOrderService.createOrderWithNestedItems(items2); // ITEM_FAIL 的嵌套事务回滚，ITEM_X 和 ITEM_Y 的嵌套事务提交，主事务提交

    // List<String> items3 = List.of("ITEM_X", "ITEM_FAIL", "FAIL_ENTIRE_ORDER_IF_ANY_ITEM_FAILS");
    // mainOrderService.createOrderWithNestedItems(items3); // ITEM_FAIL 的嵌套事务回滚，主事务因为检查到失败且有特定标记也回滚
    ```
  在这个例子中，如果 `processOrderItem` 对 "ITEM_FAIL" 抛出异常，只有这个商品项的数据库操作会回滚（回滚到它开始前的保存点）。`createOrderWithNestedItems` 可以捕获这个异常，记录失败，然后继续处理下一个商品项。最终，主事务 `createOrderWithNestedItems` 可以决定是提交已成功处理的部分，还是也进行回滚。

**重要提示**:

* **方法可见性**: `@Transactional` 注解通常只对 `public` 方法生效。对 `protected`, `private` 或包级可见方法的事务行为可能不会按预期工作，或者需要特定的AOP配置（如AspectJ模式）。
* **自调用问题**: 如果在同一个类中，一个 `public @Transactional` 方法调用同一个类的另一个 `public @Transactional` 方法（即 `this.otherMethod()`），事务传播行为可能不会按预期工作。这是因为Spring的事务管理是基于代理的，这种内部调用绕过了代理。解决方法通常是将内部方法移到另一个Bean中，或者使用 `AspectJ` 模式进行编织。
* **选择合适的传播机制**: 理解每种传播行为的含义和影响对于设计健壮的、可维护的应用程序至关重要。错误的选择可能导致数据不一致或意外的行为。

通过这些传播机制，Spring提供了非常灵活和强大的事务管理能力。