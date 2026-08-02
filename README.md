# SQL-Data-Analysis-Portfolios
SQL-Employee-Workforce-Analytics

# Advanced SQL Case Study: Employee & Workforce Analytics

## Project Overview
This repository contains a comprehensive suite of advanced SQL queries engineered to extract critical business intelligence and workforce analytics from a relational employee database. The primary focus of this project is to solve realistic corporate problems—such as compensation structures, organizational design, demographic diversity, and performance ranking—using highly optimized, production-ready SQL scripts.

---

## Tech Stack & Key SQL Concepts
* **Database Engine:** MySQL
* **Advanced Engineering Techniques:** 
  * Common Table Expressions (CTEs) & Derived Tables
  * Advanced Window Functions (`RANK()`, `DENSE_RANK()`, `ROW_NUMBER()`)
  * Conditional Aggregation (`CASE WHEN` PIVOT structures)
  * View Creation & Stored Procedures
  * Multi-table Complex `INNER JOIN` logic
  * Strict Grouping Enforcement (`ONLY_FULL_GROUP_BY` compliant)

---

## Core Business Challenges & Solutions

### 1. Workforce Distribution & Role Optimization
**Business Goal:** Identify the dominant operational role within every single company department to optimize global workforce planning and hiring pipelines.

* **Technical Highlights:** Window functions partitioned by grouping variables, subqueries vs. CTE execution variants, and strict aggregation alignment.
* **SQL Solution (CTE Approach):**
```sql
WITH cte_count AS (
    SELECT
        de.dept_no,
        t.title,
        COUNT(de.dept_no) AS count_dept,
        RANK() OVER (PARTITION BY de.dept_no ORDER BY COUNT(de.dept_no) DESC) AS highest_rank
    FROM titles t
    INNER JOIN dept_emp de
    ON de.emp_no = t.emp_no
    GROUP BY de.dept_no, t.title
)
SELECT 
    cc.dept_no,
    d.dept_name,
    cc.title AS most_common_job_title,
    cc.count_dept
FROM cte_count cc
INNER JOIN departments d
ON cc.dept_no = d.dept_no
WHERE cc.highest_rank = 1;
```

---

### 2. High-Earner Granular Performance Tracking
**Business Goal:** Isolate the top 5 highest-paid earners split by both department and gender, while simultaneously calculating how their earnings deviate from their specific group's average.

* **Technical Highlights:** Nested CTE architectures, deep window partitioning, multi-layered metric comparisons.
* **SQL Solution:**
```sql
WITH de_salaries AS (
    SELECT 
        de.dept_no,
        d.dept_name,
        e.gender,
        e.emp_no,
        ROUND(AVG(s.salary)) AS emp_avg_salary
    FROM employees e
    INNER JOIN dept_emp de
    ON e.emp_no = de.emp_no
    INNER JOIN departments d
    ON de.dept_no = d.dept_no
    INNER JOIN salaries s
    ON e.emp_no = s.emp_no
    GROUP BY de.dept_no, d.dept_name, e.gender, e.emp_no
),
dept_gender_stats AS (
    SELECT 
        dept_no,
        dept_name,
        gender,
        emp_no,
        emp_avg_salary,
        ROUND(AVG(emp_avg_salary) OVER(PARTITION BY dept_no, gender)) AS group_avg_salary,
        DENSE_RANK() OVER(PARTITION BY dept_no, gender ORDER BY emp_avg_salary DESC) AS salary_rank
    FROM de_salaries
)
SELECT 
    dept_no,
    dept_name,
    gender,
    emp_no,
    emp_avg_salary,
    group_avg_salary,
    (emp_avg_salary - group_avg_salary) AS variance_from_avg,
    salary_rank
FROM dept_gender_stats
WHERE salary_rank <= 5
ORDER BY dept_no, gender, salary_rank;
```

---

### 3. Corporate Gender Diversity Matrix
**Business Goal:** Evaluate gender distribution across all business branches to identify departments with the most balanced representation.

* **Technical Highlights:** Matrix pivoting via conditional aggregation (`COUNT(CASE WHEN...)`), table joins, and custom index calculations using mathematical absolute values (`ABS()`).
* **SQL Solution:**
```sql
SELECT
    d.dept_name,
    COUNT(CASE WHEN e.gender = 'M' THEN 1 END) AS male_count,
    COUNT(CASE WHEN e.gender = 'F' THEN 1 END) AS female_count,
    COUNT(e.emp_no) AS total_employees,
    ABS(COUNT(CASE WHEN e.gender = 'M' THEN 1 END) - COUNT(CASE WHEN e.gender = 'F' THEN 1 END)) AS gender_gap
FROM employees e
INNER JOIN dept_emp de
ON de.emp_no = e.emp_no
INNER JOIN departments d
ON de.dept_no = d.dept_no
GROUP BY d.dept_no, d.dept_name
ORDER BY gender_gap ASC; -- Lower gap indicates superior demographic balance
```

---

### 4. Career Progression & Salary Growth Velocities
**Business Goal:** Audit historical financial advancement by finding the lifetime salary growth percentage for every employee over their tenure.

* **Technical Highlights:** Twin directional tracking (`ASC`/`DESC`) using row ranking inside an analytical CTE layer to isolate historical entry and exit bounds.
* **SQL Solution:**
```sql
WITH ordered_salaries AS (
    SELECT 
        emp_no,
        salary,
        ROW_NUMBER() OVER (PARTITION BY emp_no ORDER BY from_date ASC) AS rn_first,
        ROW_NUMBER() OVER (PARTITION BY emp_no ORDER BY from_date DESC) AS rn_last
    FROM salaries
),
first_salaries AS (
    SELECT emp_no, salary AS starting_salary
    FROM ordered_salaries
    WHERE rn_first = 1
),
latest_salaries AS (
    SELECT emp_no, salary AS current_salary
    FROM ordered_salaries
    WHERE rn_last = 1
)
SELECT 
    f.emp_no,
    f.starting_salary,
    l.current_salary,
    ROUND(((l.current_salary - f.starting_salary) / f.starting_salary) * 100, 2) AS growth_percentage
FROM first_salaries f
JOIN latest_salaries l
ON f.emp_no = l.emp_no
ORDER BY growth_percentage DESC;
```

---

### 5. Managed Operations & Leadership Evaluation
**Business Goal:** Calculate the financial footprints of active and historical leadership roles by assessing the collective salary volume being managed under each individual manager.

* **Technical Highlights:** Multi-level relational grouping and cross-bridge mapping across hierarchy tables.
* **SQL Solution:**
```sql
SELECT 
    dm.emp_no AS manager_emp_no,
    d.dept_no,
    d.dept_name,
    ROUND(AVG(s.salary), 2) AS avg_managed_employee_salary
FROM dept_manager dm
INNER JOIN departments d
ON dm.dept_no = d.dept_no
INNER JOIN dept_emp de
ON d.dept_no = de.dept_no
INNER JOIN salaries s
ON de.emp_no = s.emp_no
GROUP BY dm.emp_no, d.dept_no, d.dept_name
ORDER BY avg_managed_employee_salary DESC;
```

---

### 6. Data Virtualization & Automated Reporting Layer
**Business Goal:** Construct a secure, abstraction reporting framework isolating real-time employee rosters, supplemented with automated, executable macros for high-level management readouts.

* **Technical Highlights:** Custom view creation parsing current flag data (`'9999-01-01'`), custom stored routine execution block overrides (`DELIMITER`).
* **SQL Solution:**
```sql
-- Step A: Establish the Real-Time Roster View
CREATE OR REPLACE VIEW view_current_emp_salaries AS
SELECT 
    e.emp_no,
    e.first_name,
    e.last_name,
    e.gender,
    d.dept_no,
    d.dept_name,
    s.salary
FROM employees e
INNER JOIN dept_emp de
ON e.emp_no = de.emp_no AND de.to_date = '9999-01-01'
INNER JOIN departments d
ON de.dept_no = d.dept_no
INNER JOIN salaries s
ON e.emp_no = s.emp_no AND s.to_date = '9999-01-01';

-- Step B: Automate Aggregate Generation Stored Routine
DELIMITER $$

CREATE PROCEDURE sp_get_department_summaries()
BEGIN
    SELECT 
        dept_no,
        dept_name,
        COUNT(emp_no) AS total_employees,
        SUM(salary) AS total_payroll,
        ROUND(AVG(salary), 2) AS average_salary
    FROM view_current_emp_salaries
    GROUP BY dept_no, dept_name
    ORDER BY average_salary DESC;
END $$

DELIMITER ;

SELECT * FROM view_current_emp_salaries LIMIT 10;
CALL sp_get_department_summaries();
```

---

##  Key Takeaways & Debugging Competencies
During the engineering of this database analysis module, several engine-level optimizations and structural rules were successfully resolved:
1. **Aggregations & Strict Mode Resolution:** Mastered working around MySQL's `ONLY_FULL_GROUP_BY` enforcement regulations. Ensured that any un-aggregated selection variables were strictly wrapped or matched contextually inside parent `GROUP BY` logic.
2. **Ambiguous Column Avoidance:** Mitigated identifier collision errors (`Error Code: 1052`) across highly joined datasets by implementing uniform alias prefix references (e.g., `cc.`, `d.`) on matching primary/foreign keys.
3. **Optimized Window Computations:** Leveraged analytical partitions (`PARTITION BY`) to compute rankings and rolling groupings natively at the database processing layer.
