# Week 3: Database Design Fundamentals

## Learning Objectives

By the end of this week, students will be able to:

1. **Apply Normalization Principles**: Implement 1NF, 2NF, 3NF, and BCNF to design efficient relational databases
2. **Design Effective Indexes**: Create clustered, non-clustered, and covering indexes for optimal query performance
3. **Understand Data Modeling**: Design conceptual, logical, and physical data models for complex business requirements
4. **Implement Data Integrity**: Use constraints, triggers, and other mechanisms to maintain data quality
5. **Optimize Database Performance**: Apply denormalization strategies and partitioning techniques when appropriate
6. **Design Scalable Schemas**: Create database designs that accommodate growth and changing requirements

## Theoretical Content

### 1. Database Design Principles and Normalization

#### 1.1 The Normalization Process

Normalization is the systematic process of organizing data in a database to eliminate redundancy and improve data integrity. The process involves decomposing tables with anomalies to produce smaller, well-structured tables.

**First Normal Form (1NF)**:
- **Requirement**: Each table cell contains only atomic (indivisible) values
- **Requirement**: Each column contains values of the same type
- **Requirement**: Each column has a unique name
- **Requirement**: The order of storing data does not affect the content

**Example: Before 1NF (Unnormalized)**
```
Customer Orders Table:
CustomerID | CustomerName | PhoneNumbers | Orders
------------------------------------------------
1001       | John Smith   | 555-1234     | (10001:Books, 10002:Pens)
            |              | 555-5678     | (10003:Pencils)
            |              | 555-9012     |
```

**Example: After 1NF**
```
CustomerOrders Table:
CustomerID | CustomerName | PhoneNumber | OrderID | Product
-----------------------------------------------------------
1001       | John Smith   | 555-1234    | 10001   | Books
1001       | John Smith   | 555-5678    | 10002   | Pens
1001       | John Smith   | 555-9012    | 10003   | Pencils
```

**Second Normal Form (2NF)**:
- **Prerequisite**: Must be in 1NF
- **Requirement**: No partial dependencies (all non-key attributes depend on the entire primary key)
- **Application**: Only relevant for tables with composite primary keys

**Example: Before 2NF**
```
OrderDetails Table (Composite Key: OrderID + ProductID):
OrderID | ProductID | ProductName | Quantity | UnitPrice | OrderDate
--------------------------------------------------------------------
10001   | P001      | Laptop      | 1        | 999.99    | 2023-01-15
10001   | P002      | Mouse       | 2        | 29.99     | 2023-01-15
10002   | P001      | Laptop      | 1        | 999.99    | 2023-01-16
```

**Problem**: ProductName and UnitPrice depend only on ProductID, not on the entire composite key.

**Example: After 2NF**
```
OrderDetails Table:
OrderID | ProductID | Quantity
---------------------------
10001   | P001      | 1
10001   | P002      | 2
10002   | P001      | 1

Products Table:
ProductID | ProductName | UnitPrice
----------------------------------
P001      | Laptop      | 999.99
P002      | Mouse       | 29.99

Orders Table:
OrderID | OrderDate
------------------
10001   | 2023-01-15
10002   | 2023-01-16
```

**Third Normal Form (3NF)**:
- **Prerequisites**: Must be in 2NF
- **Requirement**: No transitive dependencies (non-key attributes should not depend on other non-key attributes)

**Example: Before 3NF**
```
Products Table:
ProductID | ProductName | CategoryName | CategoryDescription
------------------------------------------------------------
P001      | Laptop      | Electronics  | Electronic devices
P002      | Mouse       | Electronics  | Electronic devices
P003      | Desk        | Furniture    | Office furniture
```

**Problem**: CategoryName and CategoryDescription depend on each other, not directly on the primary key.

**Example: After 3NF**
```
Products Table:
ProductID | ProductName | CategoryID
----------------------------------
P001      | Laptop      | C001
P002      | Mouse       | C001
P003      | Desk        | C002

Categories Table:
CategoryID | CategoryName   | CategoryDescription
-----------------------------------------------
C001       | Electronics    | Electronic devices
C002       | Furniture      | Office furniture
```

**Boyce-Codd Normal Form (BCNF)**:
- **Prerequisites**: Must be in 3NF
- **Requirement**: Every determinant must be a candidate key
- **Application**: Used when 3NF is insufficient to handle complex dependencies

**Higher Normal Forms (4NF, 5NF)**:
- **4NF**: Eliminates multi-valued dependencies
- **5NF**: Eliminates join dependencies (rarely applied in practice)

#### 1.2 Denormalization Strategies

While normalization reduces redundancy, sometimes strategic denormalization improves performance by reducing join operations.

**When to Denormalize**:
- **Read-Heavy Applications**: Data warehouses and reporting systems
- **Performance Critical**: Real-time applications with strict response time requirements
- **Complex Joins**: When queries involve many table joins
- **Reporting Queries**: Analytical queries that benefit from pre-computed data

**Denormalization Techniques**:

1. **Table Merging**: Combining related tables into a single table
```sql
-- Normalized Design
SELECT c.CustomerName, o.OrderDate, SUM(od.Quantity * od.UnitPrice) as OrderTotal
FROM Customers c
JOIN Orders o ON c.CustomerID = o.CustomerID
JOIN OrderDetails od ON o.OrderID = od.OrderID
GROUP BY c.CustomerName, o.OrderDate;

-- Denormalized Design (Reduces joins)
SELECT CustomerName, OrderDate, OrderTotal
FROM DenormalizedOrders
WHERE CustomerID = @CustomerID;
```

2. **Computed Columns**: Storing derived values
```sql
ALTER TABLE OrderDetails
ADD ExtendedPrice AS (Quantity * UnitPrice) PERSISTED;
```

3. **Redundant Data Storage**: Duplicating data for performance
```sql
-- Store customer information in orders table for faster reporting
CREATE TABLE Orders_Denormalized (
    OrderID INT PRIMARY KEY,
    CustomerID INT,
    CustomerName NVARCHAR(100),  -- Redundant but improves performance
    CustomerCity NVARCHAR(50),   -- Redundant but improves performance
    OrderDate DATE,
    -- Other order details
);
```

### 2. Index Design and Optimization

#### 2.1 Index Types and Characteristics

**Clustered Index**:
- **Definition**: Determines the physical order of data in the table
- **Limitation**: Only one per table
- **Use Case**: Primary keys, frequently searched ranges

**Non-Clustered Index**:
- **Definition**: Separate structure pointing to data rows
- **Limitations**: Can have up to 999 per table in SQL Server
- **Use Case**: Frequently queried columns that are not primary keys

**Unique Index**:
- **Definition**: Ensures data uniqueness within specified columns
- **Automatic**: Created with UNIQUE constraints

**Covering Index**:
- **Definition**: Includes all columns needed by a query
- **Benefits**: Eliminates table lookups (index seeks only)

#### 2.2 Index Design Principles

**Index Selection Criteria**:
```sql
-- Consider indexing columns that:
-- 1. Are frequently used in WHERE clauses
SELECT ColumnName, ColumnUsage
FROM sys.dm_db_index_usage_stats
WHERE object_id = OBJECT_ID('YourTable')
ORDER BY user_lookups + user_scans DESC;

-- 2. Are used in JOIN conditions
-- 3. Are used in ORDER BY clauses
-- 4. Have good selectivity (low cardinality)
-- 5. Are updated infrequently
```

**Index Design Process**:

1. **Query Analysis**
```sql
-- Identify query patterns
SELECT TOP 100
    execution_count,
    total_elapsed_time / 1000 as total_time_ms,
    SUBSTRING(st.text, (qs.statement_start_offset/2)+1,
        ((CASE qs.statement_end_offset
            WHEN -1 THEN DATALENGTH(st.text)
            ELSE qs.statement_end_offset
        END - qs.statement_start_offset)/2) + 1) AS StatementText
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
WHERE database_id = DB_ID()
ORDER BY execution_count DESC;
```

2. **Index Maintenance Strategy**
```sql
-- Monitor index fragmentation
SELECT 
    OBJECT_NAME(ips.object_id) as TableName,
    i.name as IndexName,
    ips.avg_fragmentation_in_percent,
    ips.page_count,
    CASE 
        WHEN ips.avg_fragmentation_in_percent < 10 THEN 'No Action'
        WHEN ips.avg_fragmentation_in_percent < 30 THEN 'Reorganize'
        ELSE 'Rebuild'
    END as RecommendedAction
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'DETAILED') ips
INNER JOIN sys.indexes i ON ips.object_id = i.object_id AND ips.index_id = i.index_id
WHERE ips.avg_fragmentation_in_percent > 5 AND ips.page_count > 100
ORDER BY ips.avg_fragmentation_in_percent DESC;
```

#### 2.3 Advanced Indexing Techniques

**Filtered Indexes**: Index specific subset of data
```sql
CREATE INDEX IX_Customers_Active
ON Customers(LastName, FirstName)
WHERE IsActive = 1;
```

**Columnstore Indexes**: Optimized for data warehousing
```sql
CREATE NONCLUSTERED COLUMNSTORE INDEX IX_Orders_Columnstore
ON Orders(OrderDate, CustomerID, TotalAmount);
```

**Index with Included Columns**: Covering indexes
```sql
CREATE INDEX IX_Orders_CustomerDate_covering
ON Orders(CustomerID, OrderDate)
INCLUDE (TotalAmount, OrderStatus, ShippingAddress);
```

**Index on Computed Columns**: Derived values
```sql
CREATE TABLE Products (
    ProductID INT PRIMARY KEY,
    UnitPrice DECIMAL(10,2),
    QuantityInStock INT,
    TotalValue AS (UnitPrice * QuantityInStock) PERSISTED
);

CREATE INDEX IX_Products_TotalValue
ON Products(TotalValue);
```

### 3. Data Modeling and Design Patterns

#### 3.1 Entity-Relationship Modeling

**Entity Types and Attributes**:
```sql
-- Entity Relationship Modeling Process
-- 1. Identify entities (nouns in business requirements)
-- 2. Identify attributes (characteristics of entities)
-- 3. Identify relationships between entities
-- 4. Identify cardinalities (1:1, 1:N, M:N)
-- 5. Identify constraints and business rules
```

**Relationship Types**:
- **One-to-One (1:1)**: One entity instance relates to exactly one instance of another entity
- **One-to-Many (1:N)**: One entity instance relates to multiple instances of another entity
- **Many-to-Many (M:N)**: Multiple entity instances relate to multiple instances of another entity

**Cardinality Notation**:
```
Customer (1) ----< Orders (N)
Orders (N) -----<> Products (M)
Employee (1) ----<> Department (1) -- Optional relationship
```

#### 3.2 Database Design Patterns

**Master-Detail Pattern**:
```sql
-- Master table (Header)
CREATE TABLE Orders (
    OrderID INT PRIMARY KEY,
    OrderDate DATE NOT NULL,
    CustomerID INT NOT NULL,
    OrderStatus NVARCHAR(20) NOT NULL,
    TotalAmount DECIMAL(10,2) NOT NULL,
    CONSTRAINT FK_Orders_Customers FOREIGN KEY (CustomerID) REFERENCES Customers(CustomerID)
);

-- Detail table (Lines)
CREATE TABLE OrderDetails (
    OrderDetailID INT PRIMARY KEY,
    OrderID INT NOT NULL,
    ProductID INT NOT NULL,
    Quantity INT NOT NULL,
    UnitPrice DECIMAL(10,2) NOT NULL,
    LineTotal AS (Quantity * UnitPrice) PERSISTED,
    CONSTRAINT FK_OrderDetails_Orders FOREIGN KEY (OrderID) REFERENCES Orders(OrderID),
    CONSTRAINT FK_OrderDetails_Products FOREIGN KEY (ProductID) REFERENCES Products(ProductID)
);
```

**Hierarchical Data Pattern**:
```sql
-- Adjacency List Model
CREATE TABLE Employees (
    EmployeeID INT PRIMARY KEY,
    EmployeeName NVARCHAR(100) NOT NULL,
    ManagerID INT NULL,
    JobTitle NVARCHAR(100),
    HireDate DATE NOT NULL,
    CONSTRAINT FK_Employees_Manager FOREIGN KEY (ManagerID) REFERENCES Employees(EmployeeID)
);

-- Path Enumeration Model (Hierarchical path stored as string)
CREATE TABLE ProductCategories (
    CategoryID INT PRIMARY KEY,
    CategoryName NVARCHAR(100) NOT NULL,
    ParentCategoryID INT NULL,
    CategoryPath AS ('/' + CAST(CategoryID AS VARCHAR(10)) + '/') PERSISTED,
    CONSTRAINT FK_Categories_Parent FOREIGN KEY (ParentCategoryID) REFERENCES ProductCategories(CategoryID)
);

-- Nested Set Model
CREATE TABLE ProductCategories_Nested (
    CategoryID INT PRIMARY KEY,
    CategoryName NVARCHAR(100) NOT NULL,
    LeftValue INT NOT NULL,
    RightValue INT NOT NULL
);
```

**Audit Trail Pattern**:
```sql
-- Audit table with versioning
CREATE TABLE Customer_Audit (
    AuditID INT PRIMARY KEY IDENTITY(1,1),
    CustomerID INT NOT NULL,
    CustomerName NVARCHAR(100),
    Email NVARCHAR(100),
    Phone NVARCHAR(20),
    ActionType NVARCHAR(20) NOT NULL, -- INSERT, UPDATE, DELETE
    ActionDate DATETIME2 NOT NULL,
    ActionUser NVARCHAR(100) NOT NULL,
    ActionHost NVARCHAR(100),
    -- Changed values
    OldValues NVARCHAR(MAX),
    NewValues NVARCHAR(MAX)
);

-- Audit trigger for Customers table
CREATE TRIGGER tr_Customers_Audit
ON Customers
AFTER INSERT, UPDATE, DELETE
AS
BEGIN
    DECLARE @Action NVARCHAR(10),
            @CustomerID INT,
            @CustomerName NVARCHAR(100),
            @Email NVARCHAR(100),
            @Phone NVARCHAR(20);
    
    -- Get operation type
    IF EXISTS(SELECT * FROM inserted) AND EXISTS(SELECT * FROM deleted)
        SET @Action = 'UPDATE';
    ELSE IF EXISTS(SELECT * FROM inserted)
        SET @Action = 'INSERT';
    ELSE
        SET @Action = 'DELETE';
    
    -- Get changed values
    SELECT @CustomerID = COALESCE(i.CustomerID, d.CustomerID),
           @CustomerName = COALESCE(i.CustomerName, d.CustomerName),
           @Email = COALESCE(i.Email, d.Email),
           @Phone = COALESCE(i.Phone, d.Phone)
    FROM inserted i
    FULL OUTER JOIN deleted d ON i.CustomerID = d.CustomerID;
    
    -- Insert audit record
    INSERT INTO Customer_Audit (
        CustomerID, CustomerName, Email, Phone,
        ActionType, ActionDate, ActionUser, ActionHost
    )
    VALUES (
        @CustomerID, @CustomerName, @Email, @Phone,
        @Action, GETDATE(), SYSTEM_USER, HOST_NAME()
    );
END;
```

#### 3.3 Data Modeling for Different Workloads

**OLTP (Online Transaction Processing)** Design:
- **Focus**: Data integrity, concurrency, performance
- **Design**: Highly normalized tables
- **Indexes**: Covering indexes for common queries
- **Constraints**: Foreign keys, check constraints, unique constraints

**OLAP (Online Analytical Processing)** Design:
- **Focus**: Query performance, data aggregation
- **Design**: Star schema, denormalized tables
- **Indexes**: Columnstore indexes, bitmap indexes
- **Structure**: Fact tables, dimension tables

**Star Schema Example**:
```sql
-- Fact table (central table)
CREATE TABLE FactSales (
    SalesID BIGINT PRIMARY KEY,
    DateKey INT NOT NULL,
    ProductKey INT NOT NULL,
    CustomerKey INT NOT NULL,
    StoreKey INT NOT NULL,
    SalesQuantity INT NOT NULL,
    SalesAmount DECIMAL(10,2) NOT NULL,
    CostAmount DECIMAL(10,2) NOT NULL,
    CONSTRAINT FK_FactSales_Date FOREIGN KEY (DateKey) REFERENCES DimDate(DateKey),
    CONSTRAINT FK_FactSales_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey),
    CONSTRAINT FK_FactSales_Customer FOREIGN KEY (CustomerKey) REFERENCES DimCustomer(CustomerKey),
    CONSTRAINT FK_FactSales_Store FOREIGN KEY (StoreKey) REFERENCES DimStore(StoreKey)
);

-- Dimension tables
CREATE TABLE DimDate (
    DateKey INT PRIMARY KEY,
    Date DATE NOT NULL,
    Year SMALLINT NOT NULL,
    Quarter TINYINT NOT NULL,
    Month TINYINT NOT NULL,
    DayOfMonth TINYINT NOT NULL,
    DayOfWeek TINYINT NOT NULL,
    IsWeekend BIT NOT NULL
);

CREATE TABLE DimProduct (
    ProductKey INT PRIMARY KEY,
    ProductID INT NOT NULL,
    ProductName NVARCHAR(200) NOT NULL,
    CategoryName NVARCHAR(100),
    SubcategoryName NVARCHAR(100),
    BrandName NVARCHAR(100),
    IsActive BIT NOT NULL
);
```

### 4. Data Integrity and Constraints

#### 4.1 Constraint Types and Implementation

**Primary Key Constraints**:
```sql
-- Single column primary key
CREATE TABLE Customers (
    CustomerID INT PRIMARY KEY,
    CustomerName NVARCHAR(100) NOT NULL,
    Email NVARCHAR(100)
);

-- Composite primary key
CREATE TABLE OrderDetails (
    OrderID INT NOT NULL,
    ProductID INT NOT NULL,
    Quantity INT NOT NULL,
    UnitPrice DECIMAL(10,2) NOT NULL,
    CONSTRAINT PK_OrderDetails PRIMARY KEY (OrderID, ProductID)
);
```

**Foreign Key Constraints**:
```sql
-- Simple foreign key
CREATE TABLE Orders (
    OrderID INT PRIMARY KEY,
    CustomerID INT NOT NULL,
    OrderDate DATE NOT NULL,
    CONSTRAINT FK_Orders_Customers 
        FOREIGN KEY (CustomerID) REFERENCES Customers(CustomerID)
        ON DELETE CASCADE
        ON UPDATE CASCADE
);

-- Self-referencing foreign key
CREATE TABLE Employees (
    EmployeeID INT PRIMARY KEY,
    EmployeeName NVARCHAR(100) NOT NULL,
    ManagerID INT NULL,
    CONSTRAINT FK_Employees_Manager 
        FOREIGN KEY (ManagerID) REFERENCES Employees(EmployeeID)
        ON DELETE SET NULL
);
```

**Check Constraints**:
```sql
-- Basic check constraint
ALTER TABLE Products
ADD CONSTRAINT CK_Products_Price CHECK (UnitPrice > 0);

-- Complex check constraint
ALTER TABLE Orders
ADD CONSTRAINT CK_Orders_Date CHECK (OrderDate <= GETDATE());

-- Using functions in check constraints
ALTER TABLE Employees
ADD CONSTRAINT CK_Employees_Email 
CHECK (Email LIKE '%@%.%');
```

**Unique Constraints**:
```sql
-- Single column unique constraint
CREATE TABLE Customers (
    CustomerID INT PRIMARY KEY,
    Email NVARCHAR(100) NOT NULL,
    CONSTRAINT UQ_Customers_Email UNIQUE (Email)
);

-- Composite unique constraint
CREATE TABLE ProductCategories (
    CategoryID INT PRIMARY KEY,
    ParentCategoryID INT NULL,
    CategoryName NVARCHAR(100) NOT NULL,
    CONSTRAINT UQ_ProductCategories_ParentName 
        UNIQUE (ParentCategoryID, CategoryName)
);
```

**Default Constraints**:
```sql
-- Simple default constraint
CREATE TABLE Orders (
    OrderID INT PRIMARY KEY,
    OrderDate DATE NOT NULL DEFAULT GETDATE(),
    OrderStatus NVARCHAR(20) NOT NULL DEFAULT 'Pending',
    CreatedBy NVARCHAR(100) NOT NULL DEFAULT SYSTEM_USER
);
```

#### 4.2 Advanced Data Integrity Techniques

**Triggers for Complex Business Rules**:
```sql
-- Instead of trigger to prevent invalid updates
CREATE TRIGGER tr_Orders_BeforeUpdate
ON Orders
INSTEAD OF UPDATE
AS
BEGIN
    -- Prevent changes to closed orders
    IF EXISTS (
        SELECT 1 
        FROM inserted i
        INNER JOIN deleted d ON i.OrderID = d.OrderID
        WHERE d.OrderStatus = 'Closed' 
        AND (i.OrderStatus != d.OrderStatus OR i.CustomerID != d.CustomerID)
    )
    BEGIN
        RAISERROR('Cannot modify closed orders', 16, 1);
        ROLLBACK TRANSACTION;
        RETURN;
    END
    
    -- Update order for valid changes
    UPDATE o
    SET OrderStatus = i.OrderStatus,
        ModifiedDate = GETDATE(),
        ModifiedBy = SYSTEM_USER
    FROM Orders o
    INNER JOIN inserted i ON o.OrderID = i.OrderID;
END;
```

**Computed Columns for Derived Values**:
```sql
-- Persisted computed column
CREATE TABLE OrderDetails (
    OrderDetailID INT PRIMARY KEY,
    OrderID INT NOT NULL,
    ProductID INT NOT NULL,
    Quantity INT NOT NULL,
    UnitPrice DECIMAL(10,2) NOT NULL,
    ExtendedPrice AS (Quantity * UnitPrice) PERSISTED,
    DiscountAmount AS (Quantity * UnitPrice * 0.10) PERSISTED,
    FinalPrice AS (Quantity * UnitPrice * 0.90) PERSISTED,
    CONSTRAINT FK_OrderDetails_Orders FOREIGN KEY (OrderID) REFERENCES Orders(OrderID)
);
```

### 5. Database Partitioning and Performance Optimization

#### 5.1 Table Partitioning Strategies

**Horizontal Partitioning (Range Partitioning)**:
```sql
-- Create partition function
CREATE PARTITION FUNCTION PF_SalesDate (DATETIME2)
AS RANGE RIGHT FOR VALUES (
    '2020-01-01',
    '2021-01-01',
    '2022-01-01',
    '2023-01-01'
);

-- Create partition scheme
CREATE PARTITION SCHEME PS_SalesDate
AS PARTITION PF_SalesDate
TO ([PRIMARY], [PRIMARY], [PRIMARY], [PRIMARY], [PRIMARY]);

-- Create partitioned table
CREATE TABLE Sales (
    SaleID BIGINT IDENTITY(1,1) PRIMARY KEY,
    SaleDate DATETIME2 NOT NULL,
    CustomerID INT NOT NULL,
    ProductID INT NOT NULL,
    Quantity INT NOT NULL,
    SaleAmount DECIMAL(10,2) NOT NULL,
    CONSTRAINT FK_Sales_Customers FOREIGN KEY (CustomerID) REFERENCES Customers(CustomerID),
    CONSTRAINT FK_Sales_Products FOREIGN KEY (ProductID) REFERENCES Products(ProductID)
) ON PS_SalesDate(SaleDate);

-- Partition switching for efficient data loading
ALTER TABLE SalesArchive SWITCH TO Sales PARTITION $PARTITION.PF_SalesDate('2023-12-31');
```

**Vertical Partitioning**:
```sql
-- Separate frequently accessed columns
CREATE TABLE Customers_Main (
    CustomerID INT PRIMARY KEY,
    CustomerName NVARCHAR(100) NOT NULL,
    Email NVARCHAR(100) NOT NULL,
    Phone NVARCHAR(20),
    CreatedDate DATETIME2 NOT NULL DEFAULT GETDATE()
);

CREATE TABLE Customers_Details (
    CustomerID INT PRIMARY KEY,
    AddressLine1 NVARCHAR(200),
    AddressLine2 NVARCHAR(200),
    City NVARCHAR(100),
    State NVARCHAR(50),
    PostalCode NVARCHAR(20),
    Country NVARCHAR(100),
    BirthDate DATE,
    LastPurchaseDate DATETIME2,
    TotalPurchases DECIMAL(12,2),
    CONSTRAINT FK_Customers_Details_Main FOREIGN KEY (CustomerID) REFERENCES Customers_Main(CustomerID)
);
```

#### 5.2 Performance Optimization Techniques

**Statistics Management**:
```sql
-- Automatic statistics
ALTER DATABASE [YourDatabase] SET AUTO_CREATE_STATISTICS ON;
ALTER DATABASE [YourDatabase] SET AUTO_UPDATE_STATISTICS ON;
ALTER DATABASE [YourDatabase] SET AUTO_UPDATE_STATISTICS_ASYNC ON;

-- Manual statistics update for better performance
UPDATE STATISTICS Sales WITH FULLSCAN;

-- Detailed statistics for critical tables
CREATE STATISTICS STAT_Customers_Email ON Customers(Email) WITH FULLSCAN;
```

**Query Performance Monitoring**:
```sql
-- Identify slow queries
SELECT TOP 10
    execution_count,
    total_elapsed_time / 1000 as total_time_ms,
    (total_elapsed_time / execution_count) / 1000 as avg_time_ms,
    SUBSTRING(st.text, (qs.statement_start_offset/2)+1,
        ((CASE qs.statement_end_offset
            WHEN -1 THEN DATALENGTH(st.text)
            ELSE qs.statement_end_offset
        END - qs.statement_start_offset)/2) + 1) AS StatementText
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
WHERE database_id = DB_ID()
ORDER BY avg_time_ms DESC;
```

## Practical Exercises

### Exercise 1: Normalization Practice

**Objective**: Apply normalization principles to design a normalized database for an online library system

**Scenario**: Design a database for a library management system with the following requirements:

- Books can have multiple authors
- Members can borrow multiple books
- Books can belong to multiple categories
- Track borrowing history and overdue books
- Store member preferences

**Initial Unnormalized Table**:
```
Library Management Table:
BookID | Title | Author1 | Author2 | Category1 | Category2 | Category3 | MemberID | MemberName | BorrowDate | ReturnDate | OverdueDays
-----------------------------------------------------------------------------------------------------------------------------------------
1      | SQL Guide | John Smith | Jane Doe | Technology | Programming | NULL | 101 | Alice Johnson | 2023-01-01 | 2023-01-15 | 0
2      | C# Essentials | Jane Doe | NULL | Technology | Programming | NULL | 102 | Bob Wilson | 2023-01-02 | NULL | 15
3      | Database Design | Mike Brown | John Smith | Technology | Database | NULL | 101 | Alice Johnson | 2023-01-03 | NULL | 12
```

**Tasks**:

1. **Apply 1NF**: Remove repeating groups
2. **Apply 2NF**: Remove partial dependencies
3. **Apply 3NF**: Remove transitive dependencies
4. **Design Final Schema**: Create final normalized tables with appropriate relationships

**Solution Template**:
```sql
-- Create Books table (1NF)
CREATE TABLE Books (
    BookID INT PRIMARY KEY,
    Title NVARCHAR(200) NOT NULL,
    ISBN NVARCHAR(20) UNIQUE,
    PublishedDate DATE,
    PageCount INT,
    Description NVARCHAR(MAX)
);

-- Create Authors table (1NF, 2NF)
CREATE TABLE Authors (
    AuthorID INT PRIMARY KEY IDENTITY(1,1),
    FirstName NVARCHAR(100) NOT NULL,
    LastName NVARCHAR(100) NOT NULL,
    BirthDate DATE,
    Biography NVARCHAR(MAX)
);

-- Create BookAuthors junction table (3NF)
CREATE TABLE BookAuthors (
    BookID INT NOT NULL,
    AuthorID INT NOT NULL,
    AuthorOrder TINYINT NOT NULL,
    PRIMARY KEY (BookID, AuthorID),
    CONSTRAINT FK_BookAuthors_Books FOREIGN KEY (BookID) REFERENCES Books(BookID),
    CONSTRAINT FK_BookAuthors_Authors FOREIGN KEY (AuthorID) REFERENCES Authors(AuthorID)
);

-- Continue with other tables...
```

**Deliverable**: Complete normalized database design with all tables, constraints, and relationships

### Exercise 2: Index Design and Optimization

**Objective**: Design and implement an optimal indexing strategy for a sales database

**Scenario**: Design indexes for a sales database with the following query patterns:

**Business Requirements**:
- Query by customer (customer name, email, registration date)
- Query by product (product name, category, brand)
- Query by date range (sales reports, monthly summaries)
- Query by order status (pending, completed, cancelled)
- Join performance between orders and customers

**Table Definitions**:
```sql
-- Orders table
CREATE TABLE Orders (
    OrderID INT PRIMARY KEY IDENTITY(1,1),
    OrderDate DATETIME2 NOT NULL,
    CustomerID INT NOT NULL,
    OrderStatus NVARCHAR(20) NOT NULL,
    TotalAmount DECIMAL(10,2) NOT NULL,
    ShippingAddress NVARCHAR(500),
    Notes NVARCHAR(MAX)
);

-- Customers table
CREATE TABLE Customers (
    CustomerID INT PRIMARY KEY IDENTITY(1,1),
    FirstName NVARCHAR(100) NOT NULL,
    LastName NVARCHAR(100) NOT NULL,
    Email NVARCHAR(100) NOT NULL,
    Phone NVARCHAR(20),
    RegistrationDate DATE NOT NULL,
    IsActive BIT NOT NULL,
    CustomerTier NVARCHAR(20)
);

-- Products table
CREATE TABLE Products (
    ProductID INT PRIMARY KEY IDENTITY(1,1),
    ProductName NVARCHAR(200) NOT NULL,
    CategoryName NVARCHAR(100) NOT NULL,
    BrandName NVARCHAR(100),
    UnitPrice DECIMAL(10,2) NOT NULL,
    IsActive BIT NOT NULL
);
```

**Tasks**:

1. **Analyze Query Patterns**: Identify frequently executed queries
2. **Design Index Strategy**: Create appropriate indexes for each table
3. **Implement Covering Indexes**: Ensure index coverage for common queries
4. **Monitor Index Usage**: Write queries to monitor index effectiveness

**Sample Indexes**:
```sql
-- Customer indexes for common queries
CREATE INDEX IX_Customers_LastFirstName 
ON Customers(LastName, FirstName);

CREATE INDEX IX_Customers_Email 
ON Customers(Email) 
WHERE Email IS NOT NULL;

CREATE INDEX IX_Customers_RegistrationActive 
ON Customers(RegistrationDate, IsActive);

CREATE INDEX IX_Customers_Tier 
ON Customers(CustomerTier, IsActive) 
WHERE IsActive = 1;

-- Orders indexes for reporting queries
CREATE INDEX IX_Orders_CustomerDate 
ON Orders(CustomerID, OrderDate);

CREATE INDEX IX_Orders_DateStatus 
ON Orders(OrderDate, OrderStatus);

CREATE INDEX IX_Orders_CustomerStatus 
ON Orders(CustomerID, OrderStatus) 
WHERE OrderStatus != 'Cancelled';

-- Products indexes for search queries
CREATE INDEX IX_Products_CategoryBrand 
ON Products(CategoryName, BrandName);

CREATE INDEX IX_Products_Name 
ON Products(ProductName);

CREATE INDEX IX_Products_ActivePrice 
ON Products(IsActive, UnitPrice) 
WHERE IsActive = 1;

-- Covering index for order summary queries
CREATE INDEX IX_Orders_Summary_covering 
ON Orders(OrderDate, CustomerID, OrderStatus)
INCLUDE (TotalAmount);
```

**Deliverable**: Complete index strategy with monitoring queries to track effectiveness

### Exercise 3: Data Model Design

**Objective**: Create a complete data model for an e-commerce platform

**Scenario**: Design a comprehensive data model for an e-commerce platform supporting:

- Multi-vendor marketplace
- Customer reviews and ratings
- Inventory management across multiple warehouses
- Shopping carts and wishlists
- Order fulfillment with multiple shipping providers
- Customer loyalty program
- Real-time inventory tracking

**Business Rules**:
- Vendors can have multiple products
- Customers can leave multiple reviews
- Products can be stored in multiple warehouses
- Orders can be partially fulfilled
- Loyalty points are earned on purchases
- Reviews require verified purchase

**Tasks**:

1. **Identify Entities**: List all entities and their attributes
2. **Define Relationships**: Establish relationships and cardinalities
3. **Create Logical Model**: Design tables with appropriate constraints
4. **Implement Physical Model**: Create actual SQL DDL statements

**Expected Entities**:
- Users/Customers
- Vendors/Sellers
- Products
- Categories
- Reviews/Ratings
- Warehouses
- Inventory
- Orders
- OrderItems
- ShippingProviders
- Shipments
- ShoppingCarts
- Wishlists
- LoyaltyProgram
- CustomerPoints

**Solution Framework**:
```sql
-- Core entity tables
CREATE TABLE Customers (...);
CREATE TABLE Vendors (...);
CREATE TABLE Products (...);
CREATE TABLE Categories (...);

-- Relationship tables
CREATE TABLE CustomerReviews (...);
CREATE TABLE ProductCategories (...);
CREATE TABLE VendorProducts (...);

-- Transaction tables
CREATE TABLE Orders (...);
CREATE TABLE OrderItems (...);
CREATE TABLE ShoppingCarts (...);

-- Inventory and fulfillment
CREATE TABLE Warehouses (...);
CREATE TABLE Inventory (...);
CREATE TABLE Shipments (...);

-- Loyalty and engagement
CREATE TABLE LoyaltyPrograms (...);
CREATE TABLE CustomerPoints (...);
CREATE TABLE Wishlists (...);
```

**Deliverable**: Complete e-commerce database schema with all tables, constraints, indexes, and sample data

### Exercise 4: Partitioning Implementation

**Objective**: Implement table partitioning for a large sales history table

**Scenario**: Implement range partitioning for a sales history table with 500 million rows

**Requirements**:
- Partition by year for optimal query performance
- Support range queries by date
- Enable efficient archival and maintenance
- Maintain application transparency

**Table Structure**:
```sql
-- Sales history table with 500 million rows
CREATE TABLE SalesHistory (
    SaleID BIGINT IDENTITY(1,1) PRIMARY KEY,
    SaleDate DATETIME2 NOT NULL,
    CustomerID INT NOT NULL,
    ProductID INT NOT NULL,
    SalesRepID INT NOT NULL,
    Quantity INT NOT NULL,
    UnitPrice DECIMAL(10,2) NOT NULL,
    TotalAmount AS (Quantity * UnitPrice) PERSISTED,
    RegionID INT NOT NULL,
    ChannelID INT NOT NULL,
    DiscountPercent DECIMAL(5,2) DEFAULT 0,
    Comments NVARCHAR(500)
);
```

**Tasks**:

1. **Design Partition Strategy**: Plan partition boundaries
2. **Create Partition Function**: Implement date-based partitioning
3. **Create Partition Scheme**: Define filegroup distribution
4. **Implement Partitioned Table**: Create table on partition scheme
5. **Test Query Performance**: Verify partition elimination

**Implementation Steps**:
```sql
-- Create partition function for yearly partitioning
CREATE PARTITION FUNCTION PF_SalesHistory (DATETIME2)
AS RANGE RIGHT FOR VALUES (
    '2018-01-01', '2019-01-01', '2020-01-01', 
    '2021-01-01', '2022-01-01', '2023-01-01',
    '2024-01-01'
);

-- Create partition scheme
CREATE PARTITION SCHEME PS_SalesHistory
AS PARTITION PF_SalesHistory
TO ([PRIMARY], [PRIMARY], [PRIMARY], [PRIMARY], 
    [PRIMARY], [PRIMARY], [PRIMARY], [PRIMARY]);

-- Create partitioned table
CREATE TABLE SalesHistory_Partitioned (
    SaleID BIGINT IDENTITY(1,1) NOT NULL,
    SaleDate DATETIME2 NOT NULL,
    CustomerID INT NOT NULL,
    ProductID INT NOT NULL,
    SalesRepID INT NOT NULL,
    Quantity INT NOT NULL,
    UnitPrice DECIMAL(10,2) NOT NULL,
    TotalAmount AS (Quantity * UnitPrice) PERSISTED,
    RegionID INT NOT NULL,
    ChannelID INT NOT NULL,
    DiscountPercent DECIMAL(5,2) DEFAULT 0,
    Comments NVARCHAR(500)
) ON PS_SalesHistory(SaleDate);
```

**Deliverable**: Working partitioned table with performance testing results

## Real-World Scenarios

### Scenario 1: E-commerce Platform Database Design

**Background**: ShopEasy is redesigning their e-commerce database to handle 10x traffic growth while maintaining sub-second response times for product searches and cart operations.

**Current Challenges**:
- Database response times exceeding 3 seconds during peak hours
- Complex JOIN operations slowing down product catalog queries
- No effective caching strategy
- Poor inventory tracking causing stock-outs
- Difficulty handling promotional campaigns with complex rules

**Current Database Issues**:
```sql
-- Single large products table with all information
CREATE TABLE Products (
    ProductID INT PRIMARY KEY,
    ProductName NVARCHAR(200),
    Description NVARCHAR(MAX),
    Category NVARCHAR(100),
    Subcategory NVARCHAR(100),
    Brand NVARCHAR(100),
    Price DECIMAL(10,2),
    Cost DECIMAL(10,2),
    InventoryCount INT,
    Weight DECIMAL(8,2),
    Dimensions NVARCHAR(100),
    -- Many more columns...
);

-- Complex query taking too long
SELECT p.ProductName, p.Price, p.InventoryCount, 
       c.CategoryName, b.BrandName
FROM Products p
JOIN Categories c ON p.Category = c.CategoryName
JOIN Brands b ON p.Brand = b.BrandName
WHERE p.Category = 'Electronics'
AND p.Brand = 'Apple'
AND p.Price BETWEEN 500 AND 1500
AND p.InventoryCount > 0
ORDER BY p.Price;
```

**DBA Solution**:

1. **Normalized Design with Performance Optimization**
```sql
-- Separate related entities
CREATE TABLE Products (
    ProductID INT PRIMARY KEY,
    ProductName NVARCHAR(200) NOT NULL,
    Description NVARCHAR(MAX),
    Price DECIMAL(10,2) NOT NULL,
    Cost DECIMAL(10,2),
    Weight DECIMAL(8,2),
    Dimensions NVARCHAR(100),
    CreatedDate DATETIME2 NOT NULL DEFAULT GETDATE(),
    IsActive BIT NOT NULL DEFAULT 1
);

CREATE TABLE ProductCategories (
    CategoryID INT PRIMARY KEY IDENTITY(1,1),
    CategoryName NVARCHAR(100) NOT NULL,
    ParentCategoryID INT NULL,
    CategoryDescription NVARCHAR(500),
    CONSTRAINT FK_Categories_Parent FOREIGN KEY (ParentCategoryID) REFERENCES ProductCategories(CategoryID)
);

CREATE TABLE ProductBrands (
    BrandID INT PRIMARY KEY IDENTITY(1,1),
    BrandName NVARCHAR(100) NOT NULL,
    BrandDescription NVARCHAR(500),
    CountryOfOrigin NVARCHAR(50)
);

-- Junction tables for many-to-many relationships
CREATE TABLE ProductCategory_Junction (
    ProductID INT NOT NULL,
    CategoryID INT NOT NULL,
    IsPrimaryCategory BIT NOT NULL DEFAULT 0,
    PRIMARY KEY (ProductID, CategoryID),
    CONSTRAINT FK_ProductCategory_Product FOREIGN KEY (ProductID) REFERENCES Products(ProductID),
    CONSTRAINT FK_ProductCategory_Category FOREIGN KEY (CategoryID) REFERENCES ProductCategories(CategoryID)
);

CREATE TABLE ProductBrand_Junction (
    ProductID INT NOT NULL,
    BrandID INT NOT NULL,
    IsPrimaryBrand BIT NOT NULL DEFAULT 0,
    PRIMARY KEY (ProductID, BrandID),
    CONSTRAINT FK_ProductBrand_Product FOREIGN KEY (ProductID) REFERENCES Products(ProductID),
    CONSTRAINT FK_ProductBrand_Brand FOREIGN KEY (BrandID) REFERENCES ProductBrands(BrandID)
);
```

2. **Optimized Indexing Strategy**
```sql
-- Covering index for common product searches
CREATE INDEX IX_Products_Search_covering
ON Products(IsActive, ProductName)
INCLUDE (Price, ProductID);

-- Index for category browsing
CREATE INDEX IX_ProductCategory_ProductID
ON ProductCategory_Junction(CategoryID, IsPrimaryCategory, ProductID);

-- Filtered index for active products by brand
CREATE INDEX IX_Products_ActiveBrand
ON Products(BrandID, Price, ProductName)
WHERE IsActive = 1;

-- Full-text index for product search
CREATE FULLTEXT CATALOG ft_Products AS DEFAULT;
CREATE FULLTEXT INDEX ON Products(ProductName, Description)
    KEY INDEX PK_Products
    ON ft_Products
    WITH CHANGE_TRACKING AUTO;
```

3. **Performance Testing and Validation**
```sql
-- Monitor query performance
SET STATISTICS IO ON;
SET STATISTICS TIME ON;

-- Optimized query with proper indexing
SELECT p.ProductName, p.Price, pb.BrandName, pc.CategoryName
FROM Products p
INNER JOIN ProductCategory_Junction pcj ON p.ProductID = pcj.ProductID AND pcj.IsPrimaryCategory = 1
INNER JOIN ProductCategories pc ON pcj.CategoryID = pc.CategoryID
INNER JOIN ProductBrand_Junction pbj ON p.ProductID = pbj.ProductID AND pbj.IsPrimaryBrand = 1
INNER JOIN ProductBrands pb ON pbj.BrandID = pb.BrandID
WHERE p.IsActive = 1
AND pc.CategoryName = 'Electronics'
AND pb.BrandName = 'Apple'
AND p.Price BETWEEN 500 AND 1500
ORDER BY p.Price;
```

**Learning Outcome**: Understanding database design optimization for high-traffic e-commerce applications

### Scenario 2: Healthcare Data Warehouse Design

**Background**: MedCare Health System is building a data warehouse to analyze patient outcomes, treatment effectiveness, and operational efficiency across 25 hospitals and 200 clinics.

**Requirements**:
- Patient data spanning 10+ years
- Treatment outcomes analysis
- Readmission rate tracking
- Operational efficiency metrics
- Regulatory compliance (HIPAA)
- Real-time analytics capabilities

**Design Challenges**:
- Handling semi-structured data (clinical notes)
- Managing patient consent and privacy
- Performance requirements for complex medical queries
- Integration with multiple hospital systems
- Maintaining data lineage and audit trails

**Star Schema Design**:
```sql
-- Fact table for patient encounters
CREATE TABLE FactPatientEncounters (
    EncounterKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    DateKey INT NOT NULL,
    PatientKey INT NOT NULL,
    ProviderKey INT NOT NULL,
    FacilityKey INT NOT NULL,
    DiagnosisCodeKey INT NOT NULL,
    ProcedureCodeKey INT NOT NULL,
    EncounterType NVARCHAR(20) NOT NULL,
    LengthOfStayDays INT NULL,
    TotalCost DECIMAL(10,2),
    PatientSatisfactionScore TINYINT NULL,
    OutcomeCode NVARCHAR(10) NULL,
    ReadmissionFlag BIT NOT NULL DEFAULT 0,
    CONSTRAINT FK_Encounters_Date FOREIGN KEY (DateKey) REFERENCES DimDate(DateKey),
    CONSTRAINT FK_Encounters_Patient FOREIGN KEY (PatientKey) REFERENCES DimPatient(PatientKey),
    CONSTRAINT FK_Encounters_Provider FOREIGN KEY (ProviderKey) REFERENCES DimProvider(ProviderKey),
    CONSTRAINT FK_Encounters_Facility FOREIGN KEY (FacilityKey) REFERENCES DimFacility(FacilityKey)
);

-- Dimension tables with appropriate constraints
CREATE TABLE DimPatient (
    PatientKey INT PRIMARY KEY,
    PatientID NVARCHAR(20) NOT NULL, -- Hashed for privacy
    BirthDate DATE NOT NULL,
    Gender NVARCHAR(10) NOT NULL,
    RaceEthnicity NVARCHAR(50),
    InsuranceType NVARCHAR(30),
    PrimaryLanguage NVARCHAR(30),
    ZIPCode NVARCHAR(10),
    IsActive BIT NOT NULL DEFAULT 1,
    ConsentDate DATETIME2 NULL,
    ConsentExpiresDate DATETIME2 NULL
);

CREATE TABLE DimDiagnosis (
    DiagnosisKey INT PRIMARY KEY IDENTITY(1,1),
    ICD10Code NVARCHAR(10) NOT NULL,
    DiagnosisDescription NVARCHAR(500) NOT NULL,
    PrimaryDiagnosisCategory NVARCHAR(100),
    SeverityLevel TINYINT,
    IsChronicCondition BIT NOT NULL DEFAULT 0,
    EffectiveDate DATE NOT NULL,
    ExpirationDate DATE NULL
);
```

**Performance Optimization**:
```sql
-- Columnstore index for analytical queries
CREATE NONCLUSTERED COLUMNSTORE INDEX IX_FactEncounters_Columnstore
ON FactPatientEncounters(EncounterKey, DateKey, PatientKey, ProviderKey, 
                        EncounterType, LengthOfStayDays, TotalCost, ReadmissionFlag);

-- Indexes for common analytical queries
CREATE INDEX IX_Encounters_Date_Provider 
ON FactPatientEncounters(DateKey, ProviderKey) 
INCLUDE (EncounterType, TotalCost);

CREATE INDEX IX_Encounters_Patient_Date
ON FactPatientEncounters(PatientKey, DateKey)
INCLUDE (EncounterType, OutcomeCode);

CREATE INDEX IX_Encounters_Outcome
ON FactPatientEncounters(OutcomeCode, ReadmissionFlag, DateKey)
INCLUDE (TotalCost, LengthOfStayDays);
```

**Learning Outcome**: Designing data warehouse schemas for healthcare analytics with performance and compliance considerations

### Scenario 3: Banking System Core Database

**Background**: First National Bank is redesigning their core banking database to support real-time transaction processing, customer self-service, and regulatory reporting across multiple countries.

**Critical Requirements**:
- ACID compliance for financial transactions
- Real-time balance calculations
- Multi-currency support
- Audit trail for all transactions
- Fraud detection integration
- Regulatory reporting capabilities
- 24/7 availability with < 4-hour recovery time

**Database Design Challenges**:
- High concurrency requirements (10,000+ concurrent users)
- Complex financial calculations
- Data integrity and consistency
- Performance for both OLTP and reporting
- Scalability for growth

**Core Banking Schema Design**:
```sql
-- Account balances with proper constraints
CREATE TABLE Accounts (
    AccountID BIGINT PRIMARY KEY IDENTITY(1,1),
    CustomerID INT NOT NULL,
    AccountNumber NVARCHAR(20) UNIQUE NOT NULL,
    AccountType NVARCHAR(20) NOT NULL, -- CHECKING, SAVINGS, CREDIT, LOAN
    CurrencyCode NVARCHAR(3) NOT NULL,
    CurrentBalance DECIMAL(18,2) NOT NULL,
    AvailableBalance AS (CurrentBalance - ReservedAmount) PERSISTED,
    ReservedAmount DECIMAL(18,2) NOT NULL DEFAULT 0,
    Status NVARCHAR(20) NOT NULL DEFAULT 'ACTIVE', -- ACTIVE, CLOSED, FROZEN
    OpenDate DATE NOT NULL,
    LastActivityDate DATE NULL,
    InterestRate DECIMAL(5,4) NULL,
    CONSTRAINT FK_Accounts_Customers FOREIGN KEY (CustomerID) REFERENCES Customers(CustomerID),
    CONSTRAINT CK_Accounts_Balance CHECK (CurrentBalance >= 0),
    CONSTRAINT CK_Accounts_Reserved CHECK (ReservedAmount >= 0),
    CONSTRAINT CK_Accounts_Type CHECK (AccountType IN ('CHECKING', 'SAVINGS', 'CREDIT', 'LOAN'))
);

-- Transactions with complete audit trail
CREATE TABLE Transactions (
    TransactionID BIGINT PRIMARY KEY IDENTITY(1,1),
    TransactionReference NVARCHAR(50) UNIQUE NOT NULL,
    AccountID BIGINT NOT NULL,
    TransactionType NVARCHAR(30) NOT NULL, -- DEPOSIT, WITHDRAWAL, TRANSFER, PAYMENT, FEE
    Amount DECIMAL(18,2) NOT NULL,
    CurrencyCode NVARCHAR(3) NOT NULL,
    ExchangeRate DECIMAL(10,6) DEFAULT 1.0,
    Description NVARCHAR(200),
    TransactionDate DATETIME2 NOT NULL DEFAULT GETDATE(),
    PostedDate DATETIME2 NULL,
    Status NVARCHAR(20) NOT NULL DEFAULT 'PENDING', -- PENDING, POSTED, CANCELLED
    TransactionCode NVARCHAR(10) NOT NULL,
    UserID NVARCHAR(100) NOT NULL,
    SystemGenerated BIT NOT NULL DEFAULT 0,
    CONSTRAINT FK_Transactions_Accounts FOREIGN KEY (AccountID) REFERENCES Accounts(AccountID),
    CONSTRAINT CK_Transactions_Amount CHECK (Amount > 0),
    CONSTRAINT CK_Transactions_Status CHECK (Status IN ('PENDING', 'POSTED', 'CANCELLED'))
);

-- Balance snapshots for historical tracking
CREATE TABLE AccountBalances (
    BalanceID BIGINT PRIMARY KEY IDENTITY(1,1),
    AccountID BIGINT NOT NULL,
    BalanceDate DATE NOT NULL,
    CurrentBalance DECIMAL(18,2) NOT NULL,
    ReservedAmount DECIMAL(18,2) NOT NULL,
    DailyTransactionsCount INT NOT NULL,
    DailyTransactionsTotal DECIMAL(18,2) NOT NULL,
    CONSTRAINT FK_Balances_Accounts FOREIGN KEY (AccountID) REFERENCES Accounts(AccountID),
    CONSTRAINT UQ_Balances_Account_Date UNIQUE (AccountID, BalanceDate)
);
```

**High-Performance Indexing**:
```sql
-- Covering indexes for transaction queries
CREATE INDEX IX_Transactions_Account_Date 
ON Transactions(AccountID, PostedDate DESC)
INCLUDE (TransactionType, Amount, Status, Description);

CREATE INDEX IX_Transactions_Status_Date
ON Transactions(PostedDate, Status, TransactionType)
INCLUDE (AccountID, Amount);

-- Unique constraints for data integrity
CREATE UNIQUE INDEX IX_AccountNumbers_Unique
ON Accounts(AccountNumber);

CREATE INDEX IX_Transactions_Reference
ON Transactions(TransactionReference);

-- Computed column indexes
CREATE INDEX IX_Accounts_AvailableBalance
ON Accounts(AvailableBalance) 
WHERE Status = 'ACTIVE';

-- Partial index for active accounts only
CREATE INDEX IX_Accounts_Active_Currency
ON Accounts(CurrencyCode, AccountType, CurrentBalance)
WHERE Status = 'ACTIVE';
```

**Concurrency and Performance Considerations**:
```sql
-- Stored procedure for safe balance updates
CREATE PROCEDURE sp_UpdateAccountBalance
    @AccountID BIGINT,
    @TransactionAmount DECIMAL(18,2),
    @TransactionType NVARCHAR(30)
AS
BEGIN
    SET NOCOUNT ON;
    BEGIN TRANSACTION;
    
    -- Update balance based on transaction type
    IF @TransactionType IN ('DEPOSIT', 'CREDIT', 'TRANSFER_IN')
    BEGIN
        UPDATE Accounts 
        SET CurrentBalance = CurrentBalance + @TransactionAmount,
            LastActivityDate = CAST(GETDATE() AS DATE)
        WHERE AccountID = @AccountID 
        AND Status = 'ACTIVE';
    END
    ELSE IF @TransactionType IN ('WITHDRAWAL', 'DEBIT', 'TRANSFER_OUT')
    BEGIN
        UPDATE Accounts 
        SET CurrentBalance = CurrentBalance - @TransactionAmount,
            LastActivityDate = CAST(GETDATE() AS DATE)
        WHERE AccountID = @AccountID 
        AND Status = 'ACTIVE'
        AND (CurrentBalance - @TransactionAmount) >= 0; -- Check sufficient funds
    END
    
    IF @@ROWCOUNT = 0
    BEGIN
        ROLLBACK TRANSACTION;
        THROW 50000, 'Insufficient funds or invalid account', 1;
    END
    
    COMMIT TRANSACTION;
END;
```

**Learning Outcome**: Designing high-performance, mission-critical databases with strict ACID requirements and audit capabilities

## Summary and Key Takeaways

This week covered comprehensive database design fundamentals essential for SQL Server DBA success:

1. **Normalization Principles**: Understanding when and how to apply normalization for data integrity and performance
2. **Index Design**: Strategic indexing for optimal query performance and application scalability
3. **Data Modeling**: Entity-relationship modeling and pattern recognition for efficient database structures
4. **Data Integrity**: Constraints, triggers, and business rule enforcement for reliable databases
5. **Performance Optimization**: Partitioning, statistics, and query optimization techniques
6. **Real-World Application**: Practical solutions for complex business scenarios

Key database design principles:
- **Design for Growth**: Scalable schemas that accommodate business expansion
- **Balance Normalization**: Optimize for both data integrity and query performance
- **Index Strategically**: Focus on frequently executed queries and critical access patterns
- **Maintain Data Quality**: Implement comprehensive constraints and validation
- **Monitor Performance**: Continuously analyze and optimize database performance

## Additional Resources

### Database Design Resources
- Database Design for Mere Mortals (Michael J. Hernandez)
- SQL Server Performance Tuning and Optimization
- Microsoft SQL Server 2019 Performance Tuning Guide

### Tools and Utilities
- SQL Server Database Diagram Tools
- Entity Framework for application design
- Visual Studio Database Projects for schema management

### Standards and Best Practices
- ISO/IEC 11179 Data Documentation Standards
- Chen's Entity-Relationship Model
- Star Schema and Data Warehousing Design Patterns

---

**Next Week Preview**: Security and Permissions - We'll explore SQL Server security architecture, authentication methods, authorization, and comprehensive security management for enterprise environments.