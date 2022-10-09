# Pewlett-Hackard-Analysis

## Overview

Our goal is to migrating an aging employee data from Excel & VBA to a modern database solution like PostGresSQL. The workload is necessary from a human resources perspective because a substantial number of employees are reaching retirement age, and the company will need to determine precisely how many roles will need to be filled, and how many eligible candidates we can provide training mentorships to from within the company.

## Results

### Entity Relationship Diagram

We start by reviewing the CSV files exported from Excel to get an idea of the tables we'll need in our database. Using QuickDBD we're able to put together a quick visual representation of all the necessary tables, as well as isolate unique columns that can be used for primary and foreign keys to link our tables together. Here's an exerpt from `EmployeeDB_QuickDBD_schema.txt`:


	Employees
	-
	dept_no varchar pk fk - Departments.dept_no
	emp_no int pk FK - Salaries.emp_no
	birth_date date
	first_name varchar
	last_name varchar
	gender varchar
	date date


And here's our resulting Entity Relationship Diagram (ERD):

![Entity Relationship Diagram](https://github.com/bristlab/Pewlett-Hackard-Analysis/blob/main/EmployeeDB.png?raw=true)


### Create SQL schema

We need to translate our ERD into SQL queries that will generate the actual tables and their relationships to each other. After a bit of trial and error, and many dropped tables, we arrived at a schema that generates a skeletal framework of six different tables that will hold our data. We notice that 5 out of 6 tables contain an `emp_no` column which represents the employee number. This column proved to be very useful for establishing relationships between our tables. Here's an exerpt from `schema.sql`.

	CREATE TABLE dept_manager (
		dept_no VARCHAR(4) NOT NULL,
		emp_no INT NOT NULL,
		from_date DATE NOT NULL,
		to_date DATE NOT NULL,
		FOREIGN KEY (emp_no) REFERENCES employees (emp_no),
		FOREIGN KEY (dept_no) REFERENCES departments (dept_no),
		PRIMARY KEY (emp_no, dept_no)
	);

### Importing CSVs

Now that we have our framework, we can start importing our CSV data into pgAdmin. Assuming no errors on import, we run `SELECT * FROM tablename` to view the table, we need to take care that everything imports properly by verifying that the data lands in the correct columns.

### Running queries

Once we're satisfied with our framework and imported data, we start writing SQL queries to filter, sort and view section of our database. One of our first queries involves creating a new table from the existing data which contains all employees who are approaching retirement age. Here's an example from `queries.sql`.

	-- Create new table for retiring employees
	SELECT emp_no, first_name, last_name
	INTO retirement_info
	FROM employees
	WHERE (birth_date BETWEEN '1952-01-01' AND '1955-12-31')
	AND (hire_date BETWEEN '1985-01-01' AND '1988-12-31');
	-- Check the table
	SELECT * FROM retirement_info;

It turns out that this query also returns employees who have already left the company, so we need to join this table with another table which contains departure dates. We also provide queries that determine how many employees will be retiring on a *per department basis*, as well as the number of employees who would be eligible to fill those job openings based on seniority.

### Retirement titles

* The query below creates a new table called `retirement_titles` by merging select data from two existing tables. The new table contains the employee number, first name, last name, title, and employment dates. The two tables are joined together using the employee number they have in common, and only employees born between 1952 and 1955 are included in the new table.

	-- Retrieve the emp_no, first_name, and last_name columns from the Employees table.
	SELECT e.emp_no, e.first_name, e.last_name, t.title, t.from_date, t.to_date
	INTO retirement_titles
	FROM employees as e
	INNER JOIN titles as t
	ON (e.emp_no = t.emp_no)
	WHERE (e.birth_date BETWEEN '1952-01-01' AND '1955-12-31')
	ORDER BY emp_no

| emp_no | first_name | last_name | title | from_date | to_date
|---|---|---|---|---|---
10001 | Georgi | Facello | Senior Engineer | 1986-06-26 | 9999-01-01
10004 | Chirstian | Koblick | Engineer | 1986-12-01 | 1995-12-01
10004 | Chirstian | Koblick | Senior Engineer | 1995-12-01 | 9999-01-01
10005 | Kyoichi | Maliniak | Senior Staff | 1996-09-12 | 9999-01-01
10005 | Kyoichi | Maliniak | Staff | 1989-09-12 | 1996-09-12
10006 | Anneke | Preusig | Senior Engineer | 1990-08-05 | 9999-01-01
10009 | Sumant | Peac | Assistant Engineer | 1985-02-18 | 1990-02-18
10009 | Sumant | Peac | Engineer | 1990-02-18 | 1995-02-18
10009 | Sumant | Peac | Senior Engineer | 1995-02-18 | 9999-01-01
10011 | Mary | Sluis | Staff | 1990-01-22 | 1996-11-09

### Unique titles

* While the `retirement_titles` table is a good start, it fails to exclude former employees who are no longer with the company. Also, it contains duplicate entries because some employees have received promotions to different job titles over the years. So we'll create a new table, `unique_titles`, which contains the current title for all current employees.

	-- Use Distinct with Orderby to remove duplicate rows
	SELECT DISTINCT ON (emp_no) emp_no,
	first_name,
	last_name,
	title
	INTO unique_titles
	FROM retirement_titles
	WHERE (to_date = '9999-01-01')
	ORDER BY emp_no, to_date DESC;
	SELECT * FROM unique_titles;

| emp_no | first_name | last_name | title
|---|---|---|---
10001 | Georgi | Facello | Senior Engineer
10004 | Chirstian | Koblick | Senior Engineer
10005 | Kyoichi | Maliniak | Senior Staff
10006 | Anneke | Preusig | Senior Engineer
10009 | Sumant | Peac | Senior Engineer
10018 | Kazuhide | Peha | Senior Engineer
10019 | Lillian | Haddadi | Staff
10020 | Mayuko | Warwick | Engineer
10022 | Shahaf | Famili | Engineer
10023 | Bojan | Montemayor | Engineer

### Retiring titles

* We've determined the number of employees retiring based on their job titles so that human resources can get an idea of how large their mentorship program will need to be in order to fill all the positions.

	-- Retrieve the number of employees by their most recent job title who are about to retire.
	SELECT COUNT(title), title
	INTO retiring_titles
	FROM unique_titles
	GROUP BY title
	ORDER BY count DESC;
	SELECT * FROM retiring_titles;

| count | title
| ---|---
| 25916 | Senior Engineer
| 24926 | Senior Staff
| 9285 | Engineer
| 7636 | Staff
| 3603| Technique Leader
| 1090 | Assistant Engineer
| 2 | Manager


### Mentorship eligibility

* Finally, we have also exported a CSV summarizing all current employees who could be eligible for promotions as their predecessors enter retirement. Here's the SQL inquiry and an excerpt from `mentorship_eligibility.csv`.


	-- Create a Mentorship Eligibility table that holds the employees who are eligible
	-- to participate in a mentorship program.
	SELECT DISTINCT ON (e.emp_no) e.emp_no,
	e.first_name,
	e.last_name,
	e.birth_date,
	de.from_date,
	de.to_date,
	t.title
	INTO mentorship_eligibility
	FROM employees AS e
	INNER JOIN dept_emp as de
	ON (e.emp_no = de.emp_no)
	INNER JOIN titles as t
	ON (e.emp_no = t.emp_no)
	WHERE (e.birth_date BETWEEN '1965-01-01' AND '1965-12-31')
	AND (de.to_date = '9999-01-01')
	ORDER BY emp_no;


| emp_no | first_name | last_name | birth_date | from_date | to_date | title
|---|---|---|---|---|---|---
| 10095 | Hilari | Morton | 1965-01-03 | 1994-03-10 | 9999-01-01 | Senior Staff
| 10122 | Ohad | Esposito | 1965-01-19 | 1998-08-06 | 9999-01-01 | Technique Leader
| 10291 | Dipayan | Seghrouchni | 1965-01-23 | 1987-03-30 | 9999-01-01 | Senior Staff
| 10476 | Kokou | Iisaka | 1965-01-01 | 1987-09-20 | 9999-01-01 | Senior Staff
| 10663 | Teunis | Noriega | 1965-01-09 | 1999-02-12 | 9999-01-01 | Technique Leader
| 10762 | Lech | Himler | 1965-01-19 | 1992-01-21 | 9999-01-01 | Senior Staff
| 10933 | Juyoung | Seghrouchni | 1965-01-24 | 1993-08-02 | 9999-01-01 | Senior Engineer
| 12155 | Keiichiro | Glinert | 1965-01-21 | 1993-09-16 | 9999-01-01 | Engineer
| 12408 | Rasiah | Sudkamp | 1965-01-10 | 1995-04-18 | 9999-01-01 | Senior Engineer
| 12643 | Morrie | Schurmann | 1965-01-30 | 1998-12-31 | 9999-01-01 | Staff


## Summary

* How many roles will need to be filled as the "silver tsunami" begins to make an impact?

We'll incorporate a `COUNT()` method into our query to count all employee numbers listed in the `unique_titles` table.

`SELECT COUNT(emp_no) FROM unique_titles`
`>>> 72458`

Our query returns 72458 unique employees who are approaching retirement and whose positions will need to be filled. We can confirm this count by summing the `count` column of the `retiring_titles` table as follows:

`SELECT SUM(count) FROM retiring_titles`
`>>> 72458`

* Are there enough qualified, retirement-ready employees in the departments to mentor the next generation of Pewlett Hackard employees?

By counting the number of employee numbers in `mentorship_eligibility`, we get the following:

`SELECT COUNT(emp_no) FROM mentorship_eligibility`

`>>> 1549`

It does not appear that there are enough eligible candidates to fill the vacant positions so far. 



