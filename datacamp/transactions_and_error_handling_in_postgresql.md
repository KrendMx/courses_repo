# Transactions and Error Handling in PostgreSQL

## Chapter 1: Getting to Know Transactions

#### Making our first transaction

```sql
-- Begin a new transaction
BEGIN;
-- Update RCOP752 to true if RCON2365 is over 5000000
UPDATE ffiec_reci
SET RCONP752 = 'true'
WHERE RCON2365 > 5000000;
-- Commit the transaction
COMMIT;
-- Select a count of records now true
SELECT COUNT(RCONP752)
FROM ffiec_reci
WHERE RCONP752 = 'true';
```

---

### Multiple statement transactions

```sql
-- Begin a new transaction
Begin;
-- Update FIELD48 flag status if US State Government deposits are held
UPDATE ffiec_reci
SET FIELD48 = 'US-STATE-GOV'
WHERE RCON2203 > 0;
-- Update FIELD48 flag status if Foreign deposits are held
UPDATE ffiec_reci
SET FIELD48 = 'FOREIGN'
WHERE RCON2236 > 0;
-- Update FIELD48 flag status if US State Government and Foreign deposits are held
UPDATE ffiec_reci
SET FIELD48 = 'BOTH'
WHERE RCON2203 > 0
AND RCON2236 > 0;
-- Commit the transaction
COMMIT;
-- Select a count of records where FIELD48 is now BOTH
SELECT COUNT(FIELD48)
FROM ffiec_reci
WHERE FIELD48 = 'BOTH';
```

---

### Using and making transactions

| Use transactions                                              | Make small transactions |
| :------------------------------------------------------------ | :---------------------- |
| Handling how statements are affected by concurrent operations | Database Performance    |
| Multiple statements that should succeed or fail as a group    | Error domains           |
|                                                               | Easier to reason about  |

---

### Single statement transactions

```sql
-- Update records to indicate nontransactionals over 100,000,000
UPDATE ffiec_reci
SET FIELD48 = '1'
WHERE RCONB550 > 100000000;
-- Select a count of records where the flag field is not '1'
SELECT COUNT(*)
FROM ffiec_reci
WHERE FIELD48 != '1' OR FIELD48 IS NULL;
```

---

### Using an isolation level

```sql
-- Create a new transaction with an isolation level of repeatable read
START TRANSACTION ISOLATION LEVEL REPEATABLE READ;
-- Count of records over 100000000
SELECT COUNT(RCON2210)
FROM ffiec_reci
WHERE RCON2210 > 100000000;
-- Count of records still over 100000000
SELECT COUNT(RCON2210)
FROM ffiec_reci
WHERE RCON2210 > 100000000;
-- Commit the transaction
COMMIT;
```

---

### Isolation levels and transactions

```sql
-- Create a new transaction with a serializable isolation level
START TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- Update records with a 50% reduction if greater than 100000
UPDATE ffiec_reci
SET RCON0352 = RCON0352 * 0.5
WHERE RCON0352 > 100000;

-- Commit the transaction
COMMIT;

-- Select a count of records still over 100000
SELECT COUNT(RCON0352)
FROM ffiec_reci
WHERE RCON0352 > 100000;
```

---

## Chapter 2: Rolling Back and Savepoints

### Using rollbacks

#### (part 1)

```sql
-- Begin a new transaction
BEGIN;

-- Update RCONP752 to true if RCON2365 is over 5000
UPDATE ffiec_reci
SET RCONP752 = 'true'
WHERE RCON2365 > 5000;

-- Oops that was supposed to be 5000000 undo the statement
ROLLBACK;
```

#### (part 2)

```sql
-- Update RCOP752 to true if RCON2365 is over 5000000
UPDATE ffiec_reci
SET RCONP752 = 'true'
WHERE RCON2365 > 5000000;

-- Commit the transaction
COMMIT;

-- Select a count of records now true
SELECT COUNT(RCONP752)
FROM ffiec_reci
WHERE RCONP752 = 'true';
```

---

### Multistatement Rollbacks

```sql
-- Begin a new transaction
BEGIN;

-- Update FIELD48 flag status if US State Government deposits are held
UPDATE ffiec_reci
SET FIELD48 = 'US-STATE-GOV'
WHERE RCON2203 > 0;

-- Update FIELD48 flag status if Foreign deposits are held
UPDATE ffiec_reci
SET FIELD48 = 'FOREIGN'
WHERE RCON2236 > 0;

-- Update FIELD48 flag status if US State Government and Foreign deposits are held
UPDATE ffiec_reci
SET FIELD48 = 'BOOTH'
WHERE RCON2236 > 0
AND RCON2203 > 0;

-- Undo the mistake
ROLLBACK;

-- Select a count of records that are booth (it should be 0)
SELECT COUNT(FIELD48)
FROM ffiec_reci
WHERE FIELD48 = 'BOOTH';
```

---

### Working with a single savepoint

```sql
BEGIN;

-- Set the flag to indicate that they hold MMDAs where more than $5 million
UPDATE ffiec_reci
SET FIELD48 = 'MMDA'
WHERE RCON6810 > 5000000;

-- Set a savepoint
SAVEPOINT mmda_flag_set;

-- Rollback the whole transaction
ROLLBACK;

COMMIT;
```

---

### Rolling back with a savepoint

```sql
BEGIN;

-- Set the flag to MMDA+ where the value is greater than $6 million
UPDATE ffiec_reci set FIELD48 = 'MMDA+' where RCON6810 > 6000000;

-- Set a Savepoint
SAVEPOINT mmdaplus_flag_set;

-- Mistakenly set the flag to MMDA+ where the value is greater than $5 million
UPDATE ffiec_reci set FIELD48 = 'MMDA+' where RCON6810 > 5000000;

-- Rollback to savepoint
ROLLBACK TO mmdaplus_flag_set;

COMMIT;

-- Select count of records where the flag is MMDA+
SELECT count(FIELD48) from ffiec_reci where FIELD48 = 'MMDA+';
```

---

### Multiple savepoints

```sql
BEGIN;

-- Update FIELD48 to indicate a positive maturity rating when less than $2 million of maturing deposits.
UPDATE ffiec_reci
SET FIELD48 = 'mature+'
WHERE RCONHK07 + RCONHK12 + RCONHK08 + RCONHK13 < 2000000;

-- Set a savepoint
SAVEPOINT matureplus_flag_set;

-- Update FIELD48 to indicate a negative maturity rating when between $2 and $10 million
UPDATE ffiec_reci
SET FIELD48 = 'mature-'
WHERE RCONHK07 + RCONHK12 + RCONHK08 + RCONHK13 BETWEEN 2000000 AND 10000000;

-- Set a savepoint
SAVEPOINT matureminus_flag_set;

-- Update FIELD48 to indicate a double negative maturity rating when more than $10 million
UPDATE ffiec_reci
SET FIELD48 = 'mature--'
WHERE RCONHK07 + RCONHK12 + RCONHK08 + RCONHK13 > 10000000;

COMMIT;

-- Count the records where FIELD48 is a positive indicator
SELECT count(FIELD48)
FROM ffiec_reci
WHERE FIELD48 = 'mature+';
```

---

### Savepoints and rolling back

```sql
BEGIN;

-- Update FIELD48 to indicate a positive maturity rathing when less than $500 thousand.
UPDATE ffiec_reci
SET FIELD48 = 'mature+'
WHERE RCONHK12 + RCONHK13 < 500000;

-- Set a savepoint
SAVEPOINT matureplus_flag_set;

-- Update FIELD48 to indicate a negative maturity rathing when between $500 thousand and $1 million.
UPDATE ffiec_reci
SET FIELD48 = 'mature-'
WHERE RCONHK12 + RCONHK13 BETWEEN 500000 AND 1000000;

-- Set a savepoint
SAVEPOINT matureminus_flag_set;

-- Accidentailly update FIELD48 to indicate a double negative maturity rating when more than 100K
UPDATE ffiec_reci
SET FIELD48 = 'mature--'
WHERE RCONHK12 + RCONHK13 > 100000;

-- Rollback to before the last mistake
ROLLBACK TO matureminus_flag_set;

-- Select count of records with a double negative indicator
SELECT count(FIELD48)
from ffiec_reci
WHERE FIELD48 = 'mature--';
```

---

### Understanding outcomes

| Targeted Rollback to a savepoint                                                | Rollback Whole Transaction                              | Database Error                                          |
| :------------------------------------------------------------------------------ | :------------------------------------------------------ | :------------------------------------------------------ |
| Rollback to a savepoint that has a unique name.                                 | Rollback without a name when a savepoint has been made. | Rollback to a savepoint by name that has been released. |
| Rollback to a savepoint that has a duplicated name, but has been released once. | Rollback with no savepoints.                            |                                                         |

---

### Working with repeatable read

```sql
-- Create a new transaction with a repeatable read isolation level
START TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- Update records for banks that allow consumer deposit accounts
UPDATE ffiec_reci
SET FIELD48 = 100
WHERE RCONP752 = 'true';

-- Update records for banks that do not allow consumer deposit accounts
UPDATE ffiec_reci
SET FIELD48 = 50
WHERE RCONP752 = 'false';

-- Commit the transaction
COMMIT;
```

---

### Isolation levels comparison

| SERIALIZABLE                                                        | REPEATABLE READ                                                                        |
| :------------------------------------------------------------------ | :------------------------------------------------------------------------------------- |
| handles data dependencies among transactions.                       | errors if any data in the snapshot is altered by another transaction since it started. |
| does not use snapshots.                                             | snapshots at transaction start.                                                        |
| errors if another transaction alters the data it is going to alter. | can only see data committed before the transaction started.                            |

---

### Savepoint's effect on isolation levels

```sql
-- Create a new transaction with a repeatable read isolation level
START TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- Update records with a 35% reduction if greater than 1000000000
UPDATE ffiec_reci
SET RCON2203 = CAST(RCON2203 AS FLOAT) * .65
WHERE CAST(RCON2203 AS FLOAT) > 1000000000;

SAVEPOINT million;

-- Update records with a 25% reduction if greater than 500000000
UPDATE ffiec_reci
SET RCON2203 = CAST(RCON2203 AS FLOAT) * .75
WHERE CAST(RCON2203 AS FLOAT) > 500000000;

SAVEPOINT five_hundred;

-- Update records with a 13% reduction if greater than 300000000
UPDATE ffiec_reci
SET RCON2203 = CAST(RCON2203 AS FLOAT) * .87
WHERE CAST(RCON2203 AS FLOAT) > 300000000;

SAVEPOINT three_hundred;

-- Commit the transaction
COMMIT;


-- Select SUM the RCON2203 field
SELECT SUM(CAST(RCON2203 AS FLOAT))
FROM ffiec_reci
```

---

## Handling Exceptions

### Writing do statements

```sql
-- Create a DO $$ function
DO $$
-- BEGIN a transaction block
BEGIN
    INSERT INTO patients (a1c, glucose, fasting, created_on)
    VALUES (5.8, 89, true, '37-03-2020 01:15:54'); -- Adjusted date format
-- Add an EXCEPTION
EXCEPTION
-- For all all other type of errors
WHEN others THEN
    INSERT INTO errors (msg, detail)
    VALUES ('failed to insert', 'bad date'); -- SQLERRM captures the error message
END;
-- Make sure to specify the language
$$ language 'plpgsql';

-- Select all the errors recorded
SELECT * FROM errors;
```

---

### Handling exceptions

```sql
-- Add a DO function
DO $$
-- BEGIN a transaction block
BEGIN
    INSERT INTO patients (a1c, glucose, fasting)
    values (20, 89, TRUE);

-- Add an EXCEPTION
EXCEPTION
-- Catch all exception types
WHEN others THEN
    INSERT INTO errors (msg, detail, context) VALUES
  (
    'failed to insert',
    'This a1c value is higher than clinically accepted norms.',
    'a1c is typically less than 14'
  );
END;
-- Make sure to specify the language
$$ language 'plpgsql';

-- Select all the errors recorded
SELECT * FROM errors;
```

---

### Multiple exception blocks

```sql
-- Make a DO function
DO $$
-- Open a transaction block (Outer)
BEGIN
    -- Open a nested block (First)
    BEGIN
        INSERT INTO patients (a1c, glucose, fasting)
        VALUES
            (5.6, 93, TRUE),
            (6.3, 111, TRUE),
            (4.7, 65, TRUE);
    -- Catch all exception types
    EXCEPTION WHEN others THEN
        INSERT INTO errors (msg) VALUES ('failed to insert');
    -- End nested block
    END;

    -- Begin the second nested block
    BEGIN
        -- We are intentionally trying to update 'fasting' with a string
        UPDATE patients SET fasting = 'true' WHERE id = 1;
    -- Catch all exception types
    EXCEPTION WHEN others THEN
        INSERT INTO errors (msg) VALUES ('Inserted string into boolean.');
    -- End the second nested block
    END;
-- END the outer block
END;
$$ language 'plpgsql';

-- Select all the errors recorded
SELECT * FROM errors;
```

---

### Capturing specific exceptions

```sql
-- Make a DO function
DO $$
-- Open a transaction block
BEGIN
    -- Inserting NULL for glucose to trigger the constraint
    INSERT INTO patients (a1c, glucose, fasting)
    VALUES (7.5, NULL, TRUE);

-- Catch an Exception
EXCEPTION
    -- Catch the specific not_null_violation error
    WHEN not_null_violation THEN
        -- Insert the proper msg and detail
       INSERT INTO errors (msg, detail)
       VALUES ('failed to insert', 'Glucose can not be null.');
END$$;

-- Select all the errors recorded
SELECT * FROM errors;
```

---

### Logging messages on specific exceptions

```sql

-- Make a DO function
DO $$
-- Open a transaction block
BEGIN
    INSERT INTO patients (a1c, glucose, fasting) values (20, null, TRUE);
-- Catch an Exception
EXCEPTION
    -- Make it catch check_violation exception types
    WHEN check_violation THEN
        -- Insert the proper msg and detail
       INSERT INTO errors (msg, detail)
       VALUES ('failed to insert', 'A1C is higher than clinically accepted norms.');

    -- Make it catch not_null_violation exception types
    WHEN not_null_violation THEN
        -- Insert the proper msg and detail
       INSERT INTO errors (msg, detail)
       VALUES ('failed to insert', 'Glucose can not be null.');
END$$;

-- Select all the errors recorded
SELECT * FROM errors;

```

---

### When to use graceful degradation

| Use                                                                                        | Do Not Use                                | Maybe                                                                   |
| :----------------------------------------------------------------------------------------- | :---------------------------------------- | :---------------------------------------------------------------------- |
| When dealing with data from instruments with accuracy ranges                               | When high precision is required           | When the results of the degradation would be hidden by a math operation |
| Set out of bounds data to sentinel values so it's clearly identifiable in later processing | When your team isn't aware of the pattern |                                                                         |

---

### Graceful degradation

#### (part 1)

```sql
-- Start a DO statement with $$ as the end marker
DO $$
-- Begin a code block
BEGIN
     -- Insert the data into patients
     INSERT INTO patients (a1c, glucose, fasting) values (20, 800, False);
-- Catch a check_violation exception
EXCEPTION WHEN check_violation THEN
    -- RAISE the violation notice
    INSERT INTO errors (msg) VALUES ('This A1C is not valid, should be between 0-13');

-- END the code block and declare the language
END;
$$ language 'plpgsql';

-- Select all records in the errors table
SELECT * FROM errors;
```

#### (part 2)

```sql
-- Start a DO statement with $$ as the end marker
DO $$
-- Begin a code block
BEGIN
     -- Insert the data into patients
     INSERT INTO patients (a1c, glucose, fasting) values (20, 800, False);
-- Catch a check_violation exception
EXCEPTION WHEN check_violation THEN
    -- RAISE the violation notice
    INSERT INTO errors (msg) values ('This A1C is not valid, should be between 0-13');

    -- INSERT the max A1C value (Fallback operation)
    INSERT INTO patients (a1c, glucose, fasting) VALUES (13, 800, False);

    -- Record an error that you overrode the prior data insert
    INSERT INTO errors (msg) VALUES ('Set A1C to the maximum of 13');

-- END the code block and declare the language
END;
$$ language 'plpgsql';

-- Select all records in the errors table
SELECT * from errors;
```

---

## Stacked Diagnostics

### Getting stacked diagnostics

```sql
DO $$
-- Declare our text variables: exc_message and exc_detail
DECLARE
   exc_message text;
   exc_detail text;
BEGIN
    INSERT INTO patients (a1c, glucose, fasting)
    values (20, 89, TRUE);
EXCEPTION
WHEN others THEN
    -- Get the exception message and detail via stacked diagnostics
    GET STACKED DIAGNOSTICS
        exc_message = MESSAGE_TEXT,
        exc_detail = PG_EXCEPTION_DETAIL;
    -- Record the exception message and detail in the errors table
    INSERT INTO errors (msg, detail) VALUES (exc_message, exc_detail);
END;
$$ language 'plpgsql';

-- Select all the errors recorded
SELECT * FROM errors;
```

---

### Capturing a context stack

```sql
DO $$
DECLARE
   exc_message text;
   exc_details text;
   -- Declare a variable, exc_context to hold the exception context
   exc_context text;
BEGIN
    BEGIN
        INSERT INTO patients (a1c, glucose, fasting) values (5.6, 93, TRUE),
            (6.3, 111, TRUE),(4.7, 65, TRUE);
    EXCEPTION
        WHEN others THEN
        -- Store the exception context in exc_context
        GET STACKED DIAGNOSTICS exc_message = MESSAGE_TEXT,
                                exc_context = PG_EXCEPTION_CONTEXT;
        -- Record both the msg and the context
        INSERT INTO errors (msg, context)
           VALUES (exc_message, exc_context);
    END;
    BEGIN
        UPDATE patients set fasting = 'true' where id=1;
    EXCEPTION
        WHEN others THEN
        -- Store the exception detail in exc_details
        GET STACKED DIAGNOSTICS exc_message = MESSAGE_TEXT,
                                exc_details = PG_EXCEPTION_DETAIL;
        -- Record both the error message and the stack context
        INSERT INTO errors (msg, context)
           VALUES (exc_message, exc_context);
    END;
END$$;

SELECT * FROM errors;
```

---

### When to add custom exception logging and recording

| Yes, use stacked diagnostics                      | Don't use stacked diagnostics      |
| :------------------------------------------------ | :--------------------------------- |
| Many possible error conditions                    | Standard error message too generic |
| Need to be able to get more context for the error | Clear error context                |
| Debugging                                         | Expected error condition           |

---

### Creating named functions and declaring variables

```sql
-- Define our function signature
CREATE OR REPLACE FUNCTION debug_statement(
    sql_stmt TEXT
)
-- Declare our return type
RETURNS BOOLEAN AS $$
    DECLARE
        exc_state   TEXT;
        exc_msg     TEXT;
        exc_detail  TEXT;
        exc_context TEXT;
    BEGIN
        BEGIN
            -- Execute the statement passed in
            EXECUTE sql_stmt;
        EXCEPTION WHEN others THEN
            GET STACKED DIAGNOSTICS
                exc_state   = RETURNED_SQLSTATE,
                exc_msg     = MESSAGE_TEXT,
                exc_detail  = PG_EXCEPTION_DETAIL,
                exc_context = PG_EXCEPTION_CONTEXT;

            INSERT INTO errors (msg, state, detail, context)
            VALUES (exc_msg, exc_state, exc_detail, exc_context);

            -- Return True to indicate the statement was debugged (caught an error)
            RETURN TRUE;
        END;
        -- Return False to indicate the statement was not debugged (it ran successfully)
        RETURN FALSE;
    END;
$$ LANGUAGE plpgsql;

-- Test the function
SELECT debug_statement('INSERT INTO patients (a1c, glucose, fasting) values (20, 89, TRUE);');
```

---

### Structure of stacked diagnostics function

1. `CREATE OR REPLACE FUNCTION` and start the body
2. `DECLARE` variables and `BEGIN` SQL statements
3. handle `EXCEPTION`
4. GET STACKED DIAGNOSTICS and store in variables
5. record errors
6. Close body and `END` the function

---

### Putting it all together

```sql
DO $$
-- Begin a code block
DECLARE
    stmt VARCHAR(100) := 'INSERT INTO patients (a1c, glucose, fasting) VALUES (20, 800, False)';
BEGIN
     -- Insert the data into patients by running the statement
     EXECUTE stmt;
-- Catch a check_violation exception and perform the debug_statement function on it.
EXCEPTION WHEN OTHERS THEN
    PERFORM debug_statement(stmt);
-- END the code block and declare the language
END; $$ language 'plpgsql';

-- Select from the errors table
SELECT * FROM errors;
```
