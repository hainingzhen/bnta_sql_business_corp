# Task - Employee Records

The major corporation BusinessCorp&#8482; wants to do some analysis of varioius metrics around its employees and have contracted the job out to you. They have provided you with two SQL files, you will need to write the queries which will extract the data they are looking for.

## Setup

- Create a database to run the files against
- Use the `psql -d database -f file` command to run the SQL files
- Before starting to write the queries take some time to look at the data and figure out the relationships between the tables - maybe even draw an ERD to help.

## Tasks

### CTEs

1) Calculate the average salary of all employees

```sql
SELECT
	AVG(salary)
FROM
	employees;
```

2) Calculate the average salary of the employees in each team (hint: you'll need to `JOIN` and `GROUP BY` here)

```sql
SELECT 
	department_id,
	AVG(salary)
FROM 
	employees
GROUP BY 
	department_id;	

```

3) Using a CTE find the ratio of each employees salary to their team average, eg. an employee earning £55000 in a team where the average is £50000 has a ratio of 1.1

```sql
WITH average_by_department (department_id, average_salary) AS (
	SELECT 
		department_id,
		AVG(salary)
	FROM 
		employees
	GROUP BY 
		department_id
)
SELECT
	employees.first_name,
	employees.last_name,
	employees.salary,
	average_salary,
	salary / average_salary AS salary_ratio
FROM
	employees
INNER JOIN
	average_by_department
ON
	employees.department_id = average_by_department.department_id;
```

4) Find the employee with the highest ratio in Argentina

```sql
WITH average_by_department (department_id, average_salary) AS (
	SELECT 
		department_id,
		AVG(salary)
	FROM 
		employees
	GROUP BY 
		department_id
)
SELECT
	employees.first_name,
	employees.last_name,
	salary / average_salary AS salary_ratio
FROM
	employees
INNER JOIN
	average_by_department
ON
	employees.department_id = average_by_department.department_id
WHERE
	employees.country = 'Argentina'
ORDER BY
	salary_ratio DESC
LIMIT 
	1;
```

5) **Extension:** Add a second CTE calculating the average salary for each country and add a column showing the difference between each employee's salary and their country average

```sql
WITH average_by_country (country, average_salary) AS (
	SELECT 
		country,
		AVG(salary)
	FROM 
		employees
	GROUP BY 
		employees.country
)
SELECT 
	employees.first_name,
	employees.last_name,
	employees.country,
	average_salary,
	employees.salary,
	employees.salary - average_salary AS difference 
FROM 
	employees
INNER JOIN
	average_by_country
ON
	employees.country = average_by_country.country;
```

---

### Window Functions

1) Find the running total of salary costs as the business has grown and hired more people

```sql
SELECT
	first_name,
	last_name,
	start_date,
	salary,
	SUM(salary) OVER (ORDER BY start_date) AS cumulative_salary
FROM 
	employees
ORDER BY
	start_date;
```

2) Determine if any employees started on the same day (hint: some sort of ranking may be useful here)

```sql
-- Using COUNT
SELECT
	COUNT(employees.id) AS count,
	start_date
FROM 
	employees
GROUP BY
	start_date
ORDER BY 
	count DESC;

-- Using DENSE_RANK
SELECT
	start_date,
	DENSE_RANK() OVER (ORDER BY start_date)
FROM
	employees
ORDER BY start_date DESC;
```

3) Find how many employees there are from each country

```sql
-- Using COUNT
SELECT
	COUNT(id) AS count,
	country
FROM
	employees
GROUP BY
	country;

-- Using COUNT [window]
SELECT 
	DISTINCT (country),
	COUNT(*) OVER (PARTITION BY country)
FROM
	employees
```

4) Show how the average salary cost for each department has changed as the number of employees has increased

```sql
WITH increasing_employee_and_avg_salary 
	(
		department_id,
		employee_firstname,
		employee_lastname,
		employee_start_date,
		employee_salary,
		employee_by_department, 
		avg_salary_by_department, 
		cumulative_salary
	 ) 
	 AS 
	 (
		SELECT
			employees.department_id,
			employees.first_name,
			employees.last_name,
			employees.start_date,
			employees.salary,
			COUNT(*) OVER (PARTITION BY department_id ORDER BY start_date) AS employee_by_department,
			AVG(salary) OVER (PARTITION BY department_id ORDER BY start_date) AS avg_salary_by_department,
			SUM(salary) OVER (PARTITION BY department_id ORDER BY start_date) AS cumulative_salary
		FROM 
			employees
		ORDER BY
			department_id,
			start_date
	)

SELECT
department_id,
	employee_firstname,
	employee_lastname,
	employee_start_date,
	employee_salary,
	employee_by_department AS employee_count_by_department, 
	ROUND(avg_salary_by_department, 2) AS changing_avg_salart_by_department,
	cumulative_salary
FROM
	increasing_employee_and_avg_salary;


```

5) **Extension:** Research the `EXTRACT` function and use it in conjunction with `PARTITION` and `COUNT` to show how many employees started working for BusinessCorp&#8482; each year. If you're feeling adventurous you could further partition by month...

```sql
-- Partition by YEAR
SELECT
	DISTINCT EXTRACT(YEAR FROM start_date) AS start_year,
	COUNT(*) OVER (PARTITION BY EXTRACT(YEAR FROM start_date))
FROM 
	employees
ORDER BY
	start_year;

-- Partition by MONTH
SELECT
	DISTINCT EXTRACT(YEAR FROM start_date) AS start_year,
	EXTRACT(MONTH FROM start_date) AS start_month,
	COUNT(*) OVER (PARTITION BY EXTRACT(MONTH FROM start_date), EXTRACT(YEAR FROM start_date))
FROM 
	employees
ORDER BY
	start_year;
```

---

### Combining the two

1) Find the maximum and minimum salaries
```sql
SELECT 
	MAX(salary) AS max_salary, 
	MIN(salary) AS min_salary 
FROM 
	employees;
```

2) Find the difference between the maximum and minimum salaries and each employee's own salary

```sql
WITH min_and_max_salary (employeeId, max_salary, min_salary) AS (
	SELECT 
		employees.id,
		MAX(salary) OVER() AS max_salary, 
		MIN(salary) OVER() AS min_salary
	FROM 
		employees
	GROUP BY
		employees.id
)
SELECT
	*,
	salary - min_salary AS diff_min_salary,
	salary - max_salary AS diff_max_salary
FROM
	employees
INNER JOIN
	min_and_max_salary
ON
	employees.id = min_and_max_salary.employeeId;
```

3) Order the employees by start date. Research how to calculate the **median** salary value and the **standard deviation** in salary values and show how these change as more employees join the company

```sql
WITH min_and_max_salary (employeeId, max_salary, min_salary, salary_std_dev) AS (
	SELECT 
		employees.id,
		MAX(salary) OVER(ORDER BY start_date) AS max_salary, 
		MIN(salary) OVER(ORDER BY start_date) AS min_salary,
		STDDEV(salary) OVER(ORDER BY start_date) AS salary_std_dev
	FROM 
		employees
	GROUP BY
		employees.id
)
SELECT
	employees.first_name,
	employees.last_name,
	employees.salary,
	employees.start_date,
	max_salary,
	min_salary,
	salary_std_dev,
	(max_salary - min_salary) / 2 + min_salary AS median_salary
FROM
	employees
INNER JOIN
	min_and_max_salary
ON
	employees.id = min_and_max_salary.employeeId;
```

4) Limit this query to only Research & Development team members and show a rolling value for only the 5 most recent employees.

```sql
WITH five_recent_r_and_d (id, first_name, last_name, department_id, salary, start_date) AS (
	SELECT
		id,
		first_name,
		last_name,
		department_id,
		salary,
		start_date
	FROM
		employees
	WHERE
		department_id
		=
		(
			SELECT 
				id
			FROM 
				departments
			WHERE
				name = 'Research and Development'
		)
	ORDER BY
		start_date DESC
	LIMIT
		5
)
, five_recent_min_and_max_salary(id, first_name, last_name, salary, max_salary, min_salary, salary_std_dev) AS (
	SELECT 
		id,
		first_name,
		last_name,
		salary,
		MAX(salary) OVER(ORDER BY start_date) AS max_salary, 
		MIN(salary) OVER(ORDER BY start_date) AS min_salary,
		STDDEV(salary) OVER(ORDER BY start_date) AS salary_std_dev
	FROM 
		five_recent_r_and_d
	GROUP BY
		five_recent_r_and_d.id,
		five_recent_r_and_d.salary,
		five_recent_r_and_d.start_date,
		five_recent_r_and_d.first_name,
		five_recent_r_and_d.last_name
)

SELECT
	*,
	(max_salary - min_salary) / 2 + min_salary AS median_salary
FROM
	five_recent_min_and_max_salary;
```


