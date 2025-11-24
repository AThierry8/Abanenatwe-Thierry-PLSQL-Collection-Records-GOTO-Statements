# Abanenatwe-Thierry-PLSQL-Collection-Records-GOTO-Statements
PLSQL-Collection-Records-GOTO-Statements 

This project demonstrates a PL/SQL solution for managing library patrons. It utilizes Oracle Database features such as Nested Tables to store fine details and a Stored Procedure to evaluate a patron's eligibility to borrow restricted (high-demand) items based on their outstanding fines.
📝 Project Structure & Code Explanation
1. Database Schema Setup
What this represents: Defining the complex data types and the main storage table.
Before storing patrons, we define a custom collection type to handle multiple fine records for a single person.
SQL
-- Create a collection type to store a list of fine records (descriptions)
-- Each fine is stored as a string detailing the item and amount.
CREATE OR REPLACE TYPE t_fine_details AS TABLE OF VARCHAR2(100);
/

-- Create the patrons table
-- Note the 'fines' column uses the custom 't_fine_details' type (Nested Table)
CREATE TABLE library_patrons (
    patron_id       NUMBER PRIMARY KEY,
    name            VARCHAR2(50),
    age             NUMBER,
    membership_status VARCHAR2(20) DEFAULT 'active',
    fines           t_fine_details
)
NESTED TABLE fines STORE AS fine_details_table;
/
 
________________________________________
2. Data Insertion
What this represents: Populating the database with test cases using the Nested Table constructor.
We insert patrons with various fine histories to test different borrowing eligibility scenarios.
SQL
-- Inserting data using the t_fine_details() constructor

-- Patron 1: Clear record (Eligible)
INSERT INTO library_patrons VALUES(101, 'Alice Jones', 28, 'active', t_fine_details());

-- Patron 2: High fines (Ineligible)
INSERT INTO library_patrons VALUES(102, 'Bob Smith', 45, 'active', t_fine_details('Late: Da Vinci Code ($15.00)', 'Lost: Calculus Text ($85.00)'));

-- Patron 3: Underage (Ineligible for high-demand items, regardless of fines)
INSERT INTO library_patrons VALUES(103, 'Charlie Brown', 15, 'active', t_fine_details());

-- Patron 4: Low fines (Eligible, assuming threshold > $5)
INSERT INTO library_patrons VALUES(104, 'Dana Scully', 35, 'active', t_fine_details('Late: Journal ($3.50)'));

-- Patron 5: High fines, but suspended status
INSERT INTO library_patrons VALUES(105, 'Eve Miller', 22, 'suspended', t_fine_details('Late: History Book ($20.00)'));

COMMIT;
________________________________________
3. Stored Procedure: check_borrowing_eligibility
What this represents: The core business logic. This procedure accepts a Patron ID and checks their eligibility based on membership status and fine collection size.
SQL
CREATE OR REPLACE PROCEDURE check_borrowing_eligibility(
    p_patron_id IN library_patrons.patron_id%TYPE
)
IS
    -- Record to hold patron data (PL/SQL Record)
    v_patron library_patrons%ROWTYPE;
    
    -- Eligibility Rules
    v_min_age_restricted CONSTANT NUMBER := 18;
    v_max_fines_allowed  CONSTANT NUMBER := 1; -- For simplicity, max 1 offense/fine detail allowed
    
BEGIN
    -- Fetch patron details into a record variable
    SELECT *
    INTO v_patron
    FROM library_patrons
    WHERE patron_id = p_patron_id;

    -- 1. Status Check
    IF v_patron.membership_status <> 'active' THEN
        DBMS_OUTPUT.PUT_LINE(v_patron.name || ' is **ineligible** due to ' || v_patron.membership_status || ' status.');
        RETURN;
    END IF;

    -- 2. Age Check (for Restricted Items)
    IF v_patron.age < v_min_age_restricted THEN
        DBMS_OUTPUT.PUT_LINE(v_patron.name || ' is too young (' || v_patron.age || ') to borrow restricted items.');
        RETURN;
    END IF;

    -- 3. Fine Check (Collection Count)
    IF v_patron.fines IS NOT NULL AND v_patron.fines.COUNT > v_max_fines_allowed THEN
        DBMS_OUTPUT.PUT_LINE(v_patron.name || ' is **ineligible**. They have ' || v_patron.fines.COUNT || ' outstanding fines.');
        -- Optional: List the fines using collection iteration
        DBMS_OUTPUT.PUT_LINE('Fines include: ' || v_patron.fines(1) || '...');
        RETURN;
    END IF;

    -- 4. Eligibility (Passed all checks)
    DBMS_OUTPUT.PUT_LINE(v_patron.name || ' is **eligible** to borrow restricted items.');

EXCEPTION
    -- Error Handling: Triggers if the ID does not exist in the table
    WHEN NO_DATA_FOUND THEN
        DBMS_OUTPUT.PUT_LINE('Error: Patron with ID ' || p_patron_id || ' not found.');
    WHEN OTHERS THEN
        DBMS_OUTPUT.PUT_LINE('An unexpected error occurred: ' || SQLERRM);
END;
/
 
________________________________________
4. Testing & Results
To run these tests, you must ensure SET SERVEROUTPUT ON; is enabled.
To view data we have in our table:
SQL
SELECT
    p.patron_id,
    p.name,
    p.age,
    p.membership_status,
    T.COLUMN_VALUE AS Fine_Detail
FROM
    library_patrons p,
    TABLE(p.fines) T;
Test Case 1: Ineligible Due to High Fines
Input: Patron ID 102 (Bob Smith, 2 Fines)
SQL
BEGIN
    check_borrowing_eligibility(102);
END;
/
 
Output:
Bob Smith is **ineligible**. They have 2 outstanding fines.
Fines include: Late: Da Vinci Code ($15.00)...
Test Case 2: Eligible Patron
Input: Patron ID 101 (Alice Jones, Age 28, 0 Fines)
SQL
BEGIN
    check_borrowing_eligibility(101);
END;
/
Output:
Alice Jones is **eligible** to borrow restricted items.
Test Case 3: Ineligible Due to Age
Input: Patron ID 103 (Charlie Brown, Age 15)
SQL
BEGIN
    check_borrowing_eligibility(103);
END;
/
 
Output:
Charlie Brown is too young (15) to borrow restricted items.
Test Case 4: Invalid ID
Input: Patron ID 500 (Does not exist)
SQL
BEGIN
    check_borrowing_eligibility(500);
END;
/
  
Output:
Error: Patron with ID 500 not found.

