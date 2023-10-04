**A. Stakeholders and businesses appreciate revenue and efficiency; therefore, a report that finds the stores with the most rentals per staff member would be an excellent way for the business and its stakeholders to make sure stores are adequately staffed and producing sales as expected. This report would calculate the average sales per staff member by store and display the results in an easily digestible and comparable fashion.**
  1.	This report will require data to identify each store, the number of employees by store, and every rental connected to that employee and their respective store. It will then take this data and find each store's average rentals per employee (please see tables below).
  2.	This report will require the use of the store table, the staff table, the rental table, and two new tables that will hold the detailed and summary sections, respectively.
  3.	The detailed section will include report_id (primary key), store_id (foreign key), total_sales, and the num_staff fields. Similarly, the summary section will consist of report_id (foreign key), store_id (foreign key), and the avg_sales_per_staff.
  4.	A function to convert a store's ID number to a store name could become necessary as the business grows and more stores open. An ID number is great for back-end use but many front-end users will have an easier time with a name.
  5.	The detailed section of the report, could be utilized for budgeting, staffing, or even keeping track of the best/worst-performing stores. The summary section, is more focused on the sales per staff member; this metric could be utilized to track staff performance, provide incentives, calculate merit increases, etc.
  6.	This report should be run on at least a quarterly basis but preferably a monthly basis to monitor trends more accurately.

DETAILED TABLE
  Fields	
    report_id SERIAL (Primary Key)	
    store_id INT (Foreign Key – Store Table)	
    total_sales INT	
    num_staff INT

SUMMARY TABLE
  Fields	
    report_id SERIAL (Foreign Key – Detailed Table)	
    store_id INT (Foreign Key – Store Table)	
    avg_sales_per_staff (total_sales / num_staff) – Detailed Table


**B. SQL Queries to create the two tables above**
  CREATE TABLE store_sales_detailed_report (
    report_id SERIAL,
    store_id INT,
    total_sales INT,
    num_staff INT,
    PRIMARY KEY (report_id),
    CONSTRAINT fk_store
      FOREIGN KEY(store_id)
        REFERENCES store(store_id)
        ON DELETE CASCADE

  CREATE TABLE store_sales_summary_report (
    report_id SERIAL,
    store_id INT,
    avg_sales_per_staff NUMERIC(5,2),
    CONSTRAINT fk_ssdr
      FOREIGN KEY(report_id)
        REFERENCES store_sales_detailed_report(report_id)
        ON  DELETE CASCADE,
    CONSTRAINT fk_store
      FOREIGN KEY(store_id)
        REFERENCES store(store_id)
        ON DELETE CASCADE
 

**C. The INSERT INTO statement that will populate the detailed report and, in turn, the summary table. **
INSERT INTO store_sales_detailed_report (store_id, total_sales, num_staff)
  SELECT store.store_id, x.num_rentals AS total_sales, y.num_staff
  FROM store

  INNER JOIN
    (
      SELECT COUNT(**) AS num_staff, store.store_id AS store_id
      FROM staff
      INNER JOIN store
        ON staff.store_id = store.store_id
      WHERE active = 'true'
      GROUP BY store.store_id
    ) y ON y.store_id = store.store_id;
 
This query populates the table with all the necessary information. Cross referencing the store and staff tables we can verify some of the information.

To verify the rental count, we will use a simple count statement on the rental table.
  SELECT COUNT(**) AS num_rentals_by_staff
  FROM rental
  WHERE staff_id = 1 OR staff_id = 2
  GROUP BY staff_id;

**D. Create a practical function that could assist the organization in this scenario.**
(Solution - create a function that assigns each store a more human-readable name based on their store ID and a store_names table.)
   CREATE OR REPLACE FUNCTION get_store_name(num INT)
     RETURNS VARCHAR(50)
     LANGUAGE plpgsql
   AS
   $$
   BEGIN
     RETURN (
               SELECT store_name
               FROM store_names
               WHERE num = store_id
            );
  END;
  $$   


**E. Create a function/trigger update the summery table each time the detailed report is updated**
  CREATE OR REPLACE FUNCTION add_summary_row()
    RETURNS TRIGGER
    LANGUAGE plpgsql
  AS
  $$
  BEGIN
    INSERT INTO store_sales_summary_report
      (report_id, store_id, avg_sales_per_staff)
    VALUES (NEW.report_id, NEW.store_id, (NEW.total_sales / NEW.num_staff));
    RETURN NULL;
  END;
  $$

  CREATE TRIGGER add_summary_row_trigger
    AFTER INSERT OR UPDATE
    ON "store_sales_detailed_report"
    FOR EACH ROW
  EXECUTE PROCEDURE add_summary_row();


**F. The following store procedure will clear the detailed and summary tables, then insert new data into the detailed table using the previously specified SELECT statement. 
The add_summary_row_trigger then extracts and transforms the data from the detailed table and inserts it into the summary table. 
Unfortunately, POSTGRESQL doesn’t have any built-in scheduling tools, but this procedure can be called a monthly or quarterly basis using a job schedule in Linux CRUNTAB or Agent pgAgent.**
-- Store procedure - should be run monthly and/or quarterly minimum for fresh reporting --
CREATE OR REPLACE PROCEDURE run_store_sales_report()
  LANGUAGE plpgsql
AS
$$
BEGIN
  -- CLEAR DATA FROM REPORT TABLES --
  TRUNCATE store_sales_detailed_report, store_sales_summary_report;

-- INSERT NEW DATA INTO DETAILED REPORT (ALSO POPULATES SUMMARY TABLE VIA TRIGGER) --
INSERT INTO store_sales_detailed_report (store_id, total_sales, num_staff)
  SELECT store.store_id, x.num_rentals AS total_sales, y.num_staff
  FROM store

  INNER JOIN
    (
      SELECT COUNT(**) AS num_rentals, staff.staff_id, staff.store_id AS store_id
      FROM rental
      INNER JOIN staff
        ON staff.staff_id = rental.staff_id
      GROUP BY staff.staff_id
    ) x ON x.store_id = store.store_id

  INNER JOIN
    (
      SELECT COUNT(**) AS num_staff, store.store_id AS store_id
      FROM staff
      INNER JOIN store
        ON staff.store_id = store.store_id
      WHERE active = 'true'
      GROUP BY store.store_id
    ) y ON y.store_id = store.store_id;

END;
$$
  
