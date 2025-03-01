# HadoopHiveHue
# Apache Hive Employee Data Analysis with Hadoop & Hue

## 📌 Project Overview

This project focuses on analyzing **employee data** using **Apache Hive** in a **pseudo-distributed environment** set up with **Docker Compose**. The process includes:

- Setting up **Hadoop, Hive, and Hue**.
- **Loading and partitioning** employee data into a Hive table.
- Running **analytical queries** on the dataset.
- Automating execution using **HQL scripts**.
- Saving query results and pushing them to **GitHub**.

---

## 📂 Dataset Information

### **1️⃣ employees.csv**

| Column Name  | Description                                                  |
|-------------|--------------------------------------------------------------|
| emp_id      | Unique identifier for each employee                          |
| name        | Full name of the employee                                    |
| age         | Age of the employee                                          |
| job_role    | Job title or designation                                     |
| salary      | Employee’s annual salary                                     |
| project     | Project assigned (e.g., Alpha, Beta, Gamma, Delta, Omega)   |
| join_date   | Date of joining                                              |
| department  | Department name (Used for **partitioning** in Hive)         |

### **2️⃣ departments.csv**

| Column Name      | Description                     |
|-----------------|---------------------------------|
| dept_id         | Unique ID for each department  |
| department_name | Name of the department         |
| location        | Location of the department     |

---

## 🔥 Execution Steps

### **Step 1️⃣: Start Hadoop & Hive Containers**

```sh
docker start hive-server  # Start Hive server

docker exec -it hive-server /bin/bash  # Access Hive container
hive  # Launch Hive CLI
```

---

### **Step 2️⃣: Setup HDFS & Load Data**

#### **(A) Create HDFS Directories**

```sh
docker exec -it namenode /bin/bash  
hdfs dfs -mkdir -p /data/employee_data  # Create directory in HDFS
mkdir -p /data/employee_data  # Create local directory
exit  
```

#### **(B) Copy CSV Files to Namenode**

```sh
docker cp employees.csv namenode:/data/employee_data/employees.csv 
docker cp departments.csv namenode:/data/employee_data/departments.csv
```

#### **(C) Upload Data to HDFS**

```sh
docker exec -it namenode /bin/bash  
hdfs dfs -put /data/employee_data/employees.csv /data/employee_data/  
hdfs dfs -put /data/employee_data/departments.csv /data/employee_data/  
hdfs dfs -ls /data/employee_data/  # Verify upload
exit  
```

---

### **Step 3️⃣: Create & Load Hive Tables**

#### **(A) Create Employees Table**

```sql
CREATE EXTERNAL TABLE IF NOT EXISTS employees (
    emp_id INT,
    name STRING,
    age INT,
    job_role STRING,
    salary DOUBLE,
    project STRING,
    join_date STRING
)
PARTITIONED BY (department STRING)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
STORED AS TEXTFILE
LOCATION '/data/employee_data';
```

#### **(B) Create Departments Table**

```sql
CREATE EXTERNAL TABLE IF NOT EXISTS departments (
    dept_id INT,
    department_name STRING,
    location STRING
)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
STORED AS TEXTFILE
LOCATION '/data/employee_data';
```

#### **(C) Enable Dynamic Partitioning**

```sql
SET hive.exec.dynamic.partition=true;
SET hive.exec.dynamic.partition.mode=nonstrict;
```

#### **(D) Load Data into Employees Table**

```sql
CREATE EXTERNAL TABLE employees_staging (
    emp_id INT,
    name STRING,
    age INT,
    job_role STRING,
    salary DOUBLE,
    project STRING,
    join_date STRING,
    department STRING
)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
STORED AS TEXTFILE
LOCATION '/data/employee_data';
```

```sql
LOAD DATA INPATH '/data/employee_data/employees.csv' INTO TABLE employees_staging;
```

```sql
INSERT OVERWRITE TABLE employees PARTITION (department)
SELECT emp_id, name, age, job_role, salary, project, join_date, department
FROM employees_staging;
```

```sql
SHOW PARTITIONS employees;
```

#### **(E) Load Data into Departments Table**

```sql
LOAD DATA INPATH '/data/employee_data/departments.csv' INTO TABLE departments;
```

---

### **Step 4️⃣: Clean & Prepare Data**

```sql
CREATE TABLE employees_cleaned LIKE employees;
```

```sql
INSERT OVERWRITE TABLE employees_cleaned PARTITION (department)
SELECT * FROM employees WHERE department IS NOT NULL;
```

```sql
SHOW PARTITIONS employees_cleaned;
```

```sql
ALTER TABLE employees DROP PARTITION (department='HIVE_DEFAULT_PARTITION');
```

---

### **Step 5️⃣: Automate Queries with HQL**

```sh
touch hql_queries.hql  # Create an HQL file
nano hql_queries.hql  # Open file in an editor
```

Paste the following queries:

```sql
-- Employees who joined after 2015
SELECT * FROM employees_cleaned WHERE CAST(SUBSTRING(join_date, 1, 4) AS INT) > 2015;

-- Average salary per department
SELECT department, AVG(salary) FROM employees_cleaned GROUP BY department;

-- Employees in 'Alpha' project
SELECT * FROM employees_cleaned WHERE project = 'Alpha';

-- Employee count by job role
SELECT job_role, COUNT(*) FROM employees_cleaned GROUP BY job_role;

-- Employees earning above department average
SELECT e.* FROM employees_cleaned e 
JOIN (SELECT department, AVG(salary) FROM employees_cleaned GROUP BY department) dept_avg 
ON e.department = dept_avg.department WHERE e.salary > dept_avg.avg_salary;

-- Department with the most employees
SELECT department, COUNT(*) FROM employees_cleaned GROUP BY department ORDER BY COUNT(*) DESC LIMIT 1;

-- Check for NULL values
SELECT COUNT(*) FROM employees_cleaned WHERE emp_id IS NULL OR name IS NULL;

-- Join Employees with Departments
SELECT e.*, d.location FROM employees_cleaned e JOIN departments d ON e.department = d.department_name;

-- Rank Employees by Salary per Department
SELECT *, RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS salary_rank FROM employees_cleaned;

-- Top 3 highest-paid employees per department
SELECT * FROM (SELECT *, DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS rank FROM employees_cleaned) ranked WHERE rank <= 3;
```

---

### **Step 6️⃣: Execute Queries & Save Output**

```sh
docker cp hql_queries.hql hive-server:/opt/hql_queries.hql  
docker exec -it hive-server hive -f /opt/hql_queries.hql | tee /opt/hql_output.txt  
docker cp hive-server:/opt/hql_output.txt hql_output.txt  
ls -l  # Verify output
```

---

## 🎯 Final Output

- **hql_queries.hql** → Contains all SQL queries.
- **hql_output.txt** → Stores execution results.

This guide takes you **step-by-step** from **data ingestion to analysis** using **Hadoop, Hive, and Hue**. 🚀
