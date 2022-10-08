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

### Retiree roles per department

The query below creates a new table by merging select data from two existing tables. The new table contains the employee number, first name, last name, title, and employment dates. The two tables are joined together using the employee number they have in common, and only employees born between 1952 and 1955 are included in the new table.

	-- Retrieve the emp_no, first_name, and last_name columns from the Employees table.
	SELECT e.emp_no, e.first_name, e.last_name, t.title, t.from_date, t.to_date
	INTO retirement_titles
	FROM employees as e
	INNER JOIN titles as t
	ON (e.emp_no = t.emp_no)
	WHERE (e.birth_date BETWEEN '1952-01-01' AND '1955-12-31')
	ORDER BY emp_no


## Summary

We've determined the number of employees retiring based on their job titles, and at this point it's trivial to create additional queries that return the number of employees retiring per department. See the `Employee_Database_challenge.sql` for more examples.

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

| count | title
| ---|---
| 25916 | Senior Engineer
| 24926 | Senior Staff
| 9285 | Engineer
| 7636 | Staff
| 3603| Technique Leader
| 1090 | Assistant Engineer
| 2 | Manager

Finally, we have also exported a CSV summarizing all current employees who could be eligible for promotions as their predecessors enter retirement. Here's an excerpt from `mentorship_eligibility.csv`.

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
