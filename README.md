# sql-notes

### How would you find the third highest salary from a table

| DATABASE | QUERY|
|---|---|
SQL Server| SELECT TOP 1 salary FROM (SELECT TOP 3 salary from table_name ORDER By salary DESC) AS tmp ORDER BY salary ASC |
| MySQL/Postgres | SELECT salary from (SELECT salary FROM table_name ORDER BY salary DESC LIMIT 3) as tmp ORDER BY salary LIMIT 1 |
| MySQL/MariaDB | SELECT salary FROM table_name ORDER BY salary DESC LIMIT 2,1 |
| Postgres | SELECT salary FROM table_name ORDER BY salary DESC LIMIT 1 OFFSET 2 |
| All |  SELECT salary FROM <br/>  (SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC )) AS rank FROM table_name) as ranked <br> WHERE rank = 3

```sql
SELECT salary
FROM (
    SELECT salary,
           DENSE_RANK() OVER (ORDER BY salary DESC) AS rank
    FROM employees
) AS SalaryRanks
WHERE rank = 3;
```

- **RANK()** gives you the ranking within your ordered partition. Ties are assigned the same rank, with the next ranking(s) skipped. So, if you have 3 items at rank 2, the next rank listed would be ranked 5.
- **DENSE_RANK()** again gives you the ranking within your ordered partition, but the ranks are consecutive. No ranks are skipped if there are ranks with multiple items. So, if you have 3 items at rank 2, the next rank listed would be ranked 3.


### How would you write a query to calculate a cumulative sum or running total within a specific partition in SQL?

```sql
SELECT   
    column1,  
    column2,  
    SUM(column_to_sum) OVER (  
        PARTITION BY partition_column   
        ORDER BY order_column  
    ) AS running_total  
FROM table_name;
```

###  How do window functions differ from aggregate functions, and when would you use them?

| Aspect |	Aggregate Functions | Window Functions |
|---|---|---|
| Purpose |	Calculate a single result for a group of rows. | Perform calculations across a "window" of rows for each row. |
| Row Visibility | Collapse rows into a single result per group. | Retain all rows while adding a calculated value. |
| Usage | in SELECT	Typically used with GROUP BY. | Used without collapsing rows, often with the OVER() clause. |
| Scope of Operation| Operate on the entire group of rows defined by GROUP BY. | Operate on a window (subset of rows) defined by OVER().|
| Examples | SUM, COUNT, AVG, MAX, MIN | ROW_NUMBER, RANK, SUM, LAG, LEAD |
| | SELECT region, SUM(sales_amount) FROM sales GROUP BY region |  SELECT region, sales_date, SUM(sales_amount) OVER (PARTITION BY region ORDER BY sales_date) AS running_total FROM sales; |

### How do you identify and remove duplicate records in SQL without using temporary tables?

Use the ROW_NUMBER() window function to assign a unique rank to each duplicate row based on your grouping criteria.

```sql
SELECT   
     id,               -- Example primary key
    column1,  
    column2,  
    ROW_NUMBER() OVER (  
        PARTITION BY column1, column2 -- Columns to check for duplicates  
        ORDER BY id                   -- Determine which duplicate to keep   
    ) AS row_num  
FROM table_name;
```

- **PARTITION BY column1, column2**: Groups rows that are duplicates based on these columns.
- **ORDER BY id**: Assigns ROW_NUMBER() based on the id column, so the duplicate with the smallest id gets 1.


Query to Remove Duplicates:

```sql
WITH CTE AS (  
    SELECT   
        id,  
        ROW_NUMBER() OVER (  
            PARTITION BY column1, column2  
            ORDER BY id  
        ) AS row_num  
    FROM table_name  
)  

DELETE FROM table_name  
WHERE id IN (  
    SELECT id  
    FROM CTE  
    WHERE row_num > 1  
);
```
- **WITH CTE:** Creates a common table expression (CTE) to identify duplicates.
- **row_num > 1:** Identifies duplicates to be deleted, leaving only the first occurrence.
- **DELETE WHERE id IN (...):** Deletes all rows marked as duplicates.



### Find the Cumulative Sum of Sales for Each Employee

```sql
SELECT   
    employee_id,   
    sales_date,   
    sales_amount,   
    SUM(sales_amount) OVER (PARTITION BY employee_id ORDER BY sales_date) AS cumulative_sales  
FROM sales;
```

- **PARTITION BY employee_id:** Groups the data by employee. Each employee’s data is treated separately for cumulative sum calculations.


### Rank Employees Based on Their Sales Within Their Department

```sql
SELECT   
       department_id,   
       employee_id,   
       sales_amount,   
       RANK() OVER (PARTITION BY department_id ORDER BY sales_amount DESC) AS sales_rank  
FROM sales;  
```

- **PARTITION BY department_id:** Ensures ranks are calculated separately for each department.
- **RANK()** gives you the ranking within your ordered partition. Ties are assigned the same rank, with the next ranking(s) skipped. So, if you have 3 items at rank 2, the next rank listed would be ranked 5.
- **DENSE_RANK()** again gives you the ranking within your ordered partition, but the ranks are consecutive. No ranks are skipped if there are ranks with multiple items.


### Calculate a Running Total of Orders by Order Date

```sql
SELECT   
    order_id,   
    order_date,   
    SUM(order_amount) OVER (ORDER BY order_date) AS  running_total  
FROM orders;
```

- **There’s no PARTITION BY here because we are calculating the total across all orders.**

### Identify the Top Three Salaries in Each Department

```sql
SELECT   
    dept_id,  
    employee_id,  
    salary  
FROM (  
    SELECT   
        dept_id, employee_id, salary  
        RANK() OVER (PARTITION BY dept_id ORDER BY salary DESC) as rank  
    FROM employees  
) ranked_salaries  
WHERE rank <=3
```

### Compute the Difference Between the Current and Previous Month's Sales

```sql
SELECT   
    employee_id,   
    sales_month,   
    sales_amount,   
    sales_amount - LAG(sales_amount, 1, 0) OVER (PARTITION BY employee_id ORDER BY sales_month) AS sales_difference
FROM monthly_sales;
```

- **LAG(sales_amount, 1, 0):**
    - Retrieves the sales amount from the previous row (same employee, previous month).
    - The 1 specifies that we’re looking one row back.
    - The 0 is the default value if there’s no previous row (e.g., the first month).
- **PARTITION BY employee_id:**
Ensures the comparison is done separately for each employee.


### SQL Query to Find Employees Reporting to a Specific Manager


To find employees who directly and indirectly report to a specific manager using a Common Table Expression (CTE), you can implement a recursive CTE. Here's an example:

```sql
WITH RECURSIVE EmployeeHierarchy AS (  
    -- Base case: Find all employees directly reporting to the given manager  
    SELECT   
        employee_id,  
        manager_id,  
        employee_name  
    FROM employees  
    WHERE manager_id = 101 -- Replace 101 with the specific manager's ID  

    UNION ALL  

    -- Recursive case: Find employees reporting to the employees found in the previous step
    SELECT 
        e.employee_id,
        e.manager_id,
        e.employee_name
    FROM employees e
    INNER JOIN EmployeeHierarchy eh
    ON e.manager_id = eh.employee_id
)
SELECT * 
FROM EmployeeHierarchy;
```

### Explanation:
1. Base Case:
    - The first SELECT retrieves employees who directly report to the given manager_id (in this case, 101).
    - These are the first-level reports.
2. Recursive Case:
    - The second SELECT retrieves employees who report to the employees found in the previous iteration of the CTE.
    - This step ensures that indirect reports (second-level, third-level, etc.) are included.
3. Final SELECT:
    - The SELECT * retrieves all rows from the CTE, which includes employees at all levels under the specified manager.


### Flatten the Hierarchical Organization Chart Using a CTE:

```sql
WITH RECURSIVE EmployeeHierarchy AS (
    -- Base case: Start with the top-level managers (those who don't have a manager)
    SELECT 
        employee_id,
        employee_name,
        manager_id,
        1 AS level -- Top-level employees will be at level 1
    FROM employees
    WHERE manager_id IS NULL -- Root of the hierarchy (top manager)

    UNION ALL

    -- Recursive case: Get employees who report to the employees found in the previous step
    SELECT 
        e.employee_id,
        e.employee_name,
        e.manager_id,
        eh.level + 1 AS level -- Increase the level by 1 for each deeper level
    FROM employees e
    INNER JOIN EmployeeHierarchy eh
        ON e.manager_id = eh.employee_id
)
SELECT 
    employee_id,
    employee_name,
    manager_id,
    level
FROM EmployeeHierarchy
ORDER BY level, manager_id, employee_id;
```

### Explanation:
1. Base Case:
    - The first SELECT starts with the top-level managers (those with manager_id IS NULL). These are the roots of your hierarchy.
    - level = 1: This signifies that these are the top-level employees (e.g., CEO or executives).
2. Recursive Case:
    - The second SELECT recursively finds employees who report to those found in the previous step (manager_id = eh.employee_id).
    - The level + 1 keeps track of the depth in the hierarchy. For example, if an employee is a direct report to a top-level manager, they are at level 2.
3. Final SELECT:
    - After the recursion, the SELECT retrieves all the flattened results.
    - The query is ordered by level, manager_id, and employee_id to make the hierarchy easier to read (from top-level to bottom-level employees).


### Find employees earning more than the average salary in their department.

```sql 
SELECT 
    employee_id,
    employee_name,
    salary,
    department_id
FROM employees e
WHERE salary > AVG(salary) OVER (PARTITION BY department_id)
ORDER BY department_id, salary DESC;
```

### Find departments where all employees earn above a specific threshold.

```sql
SELECT department_id
FROM employees
GROUP BY department_id
HAVING MIN(salary) > 50000;  -- Replace 50000 with the salary threshold 
```

### Find the top 3 departments where employees earn the most
```sql
SELECT department_id,
       SUM(salary) AS total_salary
FROM employees
GROUP BY department_id
ORDER BY total_salary DESC
LIMIT 3;
```

### Find the average salary in each department

```sql
SELECT department_id,
       AVG(salary) AS average_salary
FROM employees
GROUP BY department_id
ORDER BY department_id;
```


### Rank Scores 
Write a SQL query to rank scores. If there is a tie between two scores, both should have the same ranking. Note that after a tie, the next ranking number should be the next consecutive integer value. In other words, there should be no “holes” between ranks.

```sql
SELECT score, DENSE_RANK() OVER (ORDER By Score DESC) AS "Rank"
FROM Scores;
```

### Consecutive Numbers 
Write an SQL query to find all numbers that appear at least three times consecutively.

```text
Logs table:
+----+-----+
| Id | Num |
+----+-----+
| 1  | 1   |
| 2  | 1   |
| 3  | 1   |
| 4  | 2   |
| 5  | 1   |
| 6  | 2   |
| 7  | 2   |
+----+-----+

Result table:
+-----------------+
| ConsecutiveNums |
+-----------------+
| 1               |
+-----------------+
1 is the only number that appears consecutively for at least three times.
```

#### Solution
```sql
SELECT a.Num as ConsecutiveNums
FROM Logs a
JOIN Logs b
ON a.id = b.id+1 AND a.num = b.num
JOIN Logs c
ON a.id = c.id+2 AND a.num = c.num;
```

### Employees Earning More Than Their Managers
Given the Employee table, write a SQL query that finds out employees who earn more than their managers.

```sql
SELECT E.Name as "Employee"
FROM Employee E
JOIN Employee M
ON E.ManagerId = M.Id
AND E.Salary > M.Salary;
```

### Duplicate Emails
Write a SQL query to find all duplicate emails in a table named Person.

#### Solution 1
```sql
SELECT Email
FROM Person
GROUP BY Email
HAVING count(*) > 1
```

#### Solution 2

```sql
WITH CTE AS(
SELECT Email, ROW_NUMBER() OVER(PARTITION BY Email ORDER BY Email) AS RN
    FROM Person
)

SELECT Email
FROM CTE
WHERE RN > 1;
```


### Customers Who Never Order
Suppose that a website contains two tables, the Customers table and the Orders table. Write a SQL query to find all customers who never order anything.

### Solution 1

```sql
SELECT Name AS Customers
FROM Customers
LEFT JOIN Orders
ON Customers.Id = Orders.CustomerId
WHERE CustomerId IS NULL;
```

#### Solution 2
```sql
SELECT Name as Customers
FROM Customers
WHERE Id NOT IN(
    SELECT CustomerId
    FROM Orders
)
```

### Department Highest Salary
Write a SQL query to find employees who have the highest salary in each of the departments.
```sql
SELECT Department.Name AS Department, Employee.Name AS Employee, Salary
FROM Employee
JOIN Department
ON Employee.DepartmentId = Department.Id
WHERE (DepartmentId, Salary) IN(
        SELECT  DepartmentId, MAX(Salary) AS Salary
        FROM Employee
        GROUP BY DepartmentId
        );
```

### Department Top Three Salaries
Write a SQL query to find employees who earn the top three salaries in each of the department.

```sql
WITH department_ranking AS (
SELECT Name AS Employee, Salary ,DepartmentId
  ,DENSE_RANK() OVER (PARTITION BY DepartmentId ORDER BY Salary DESC) AS rnk
FROM Employee
)

SELECT d.Name AS Department, r.Employee, r.Salary
FROM department_ranking AS r
JOIN Department AS d
ON r.DepartmentId = d.Id
WHERE r.rnk <= 3
ORDER BY d.Name ASC, r.Salary DESC;
```

### Delete Duplicate Emails
Write a SQL query to delete all duplicate email entries in a table named Person, keeping only unique emails based on its smallest Id.

```sql
DELETE p2
FROM Person p1
JOIN Person p2
ON p1.Email = p2.Email
AND p1.id < p2.id
```

### Rising Temperature
Write an SQL query to find all dates’ id with higher temperature compared to its previous dates (yesterday).

```sql
SELECT t.Id
FROM Weather AS t, Weather AS y
WHERE DATEDIFF(t.RecordDate, y.RecordDate) = 1
AND t.Temperature > y.Temperature;
```
