
# app_csv_udt - An Oracle PL/SQL CSV Record Generator

Create Comma Separated Value strings (rows) from an Oracle query. 

A CSV row (can be any separator character, but comma (',') and pipe ('|') are most common)
will have the separator between each field, but not one after the last field (which
would make it a delimited file rather than separated). If a field contains the separator
character or newline, then the field value is enclosed in double quotes. In that case, if the field 
also contains double quotes, then those are doubled up 
per [RFC4180](https://www.loc.gov/preservation/digital/formats/fdd/fdd000323.shtml).

The resulting set of strings (rows) can be written to a file, collected into a CLOB, or returned from 
a "TABLE" function in a SQL query via PIPE ROW.

# Content

- [Installation](#installation)
- [Use Cases](#use-cases)
    - [Create a CSV FIle](#create-a-csv-file)
    - [Retrieve a CLOB](#retrieve-a-clob)
    - [Read From Table Function](#read-from-table-function)
    - [Process Results in a Loop](#process-results-in-a-loop)
- [Test Directory](#test-directory)
- [Manual Page](#manual-page)
    - [Constructor](#constructor)
    - [get_rows](#get_rows)
    - [get_clob](#get_clob)
    - [write_file](#write_file)
    - [get_row_count](#get_row_count)
    - [get_header_row](#get_header_row)
    - [get_next_row](#get_next_row)
    - [set_separator](#set_separator)
    - [set_date_format](#set_date_format)
    - [set_num_format](#set_num_format)

# Installation

Clone this repository or download it as a [zip](https://github.com/lee-lindley/app_csv/archive/refs/heads/main.zip) archive.

Run *install.sql*

If you already have a suitable TABLE type, you can globally replace the string *arr_varchar2_udt* in
the .tps and .tpb files and comment out the section that creates it in the install file. You could also
make that a TABLE of CLOB if you have a need for rows longer than 4000 chars in a TABLE function callable
from SQL. If you are dealing exclusively in PL/SQL, the maximum row length is already 32767.

# Use Cases

## Create a CSV File

Produce a CSV file on the Oracle server in a directory to which you have write access. Presumably
you have a process that can then access the file, perhaps sending it to a vendor. *test/test3.sql*
is a contrived example of writing the file, then using TO_CLOB(BFILENAME()) to retrieve it in a SQL statement.

```sql
    DECLARE
        v_src   SYS_REFCURSOR;
        v_csv   app_csv_udt;
    BEGIN
        OPEN v_src FOR SELECT * FROM (
            SELECT TO_CHAR(employee_id) AS "Emp ID", last_name||', '||first_name AS "Fname", hire_date AS "Date,Hire,YYYYMMDD", salary AS "Salary"
            from hr.employees
            UNION ALL
            SELECT '999' AS "Emp ID", 'Baggins, Bilbo "badboy"' AS "Fname", TO_DATE('19991231','YYYYMMDD') AS "Date,Hire,YYYYMMDD", 123.45 AS "Salary"
            FROM dual
        ) ORDER BY "Fname" ;
        v_csv := app_csv_udt(
            p_cursor        => v_src
            ,p_num_format  => '$999,999.99'
            ,p_date_format  => 'YYYYMMDD'
        );
        v_csv.write_file(p_dir => 'TMP_DIR', p_file_name => 'x.csv', p_do_header => 'Y');
    END;
    /
    set echo off
    set linesize 200
    set pagesize 0
    set heading off
    set trimspool on
    set feedback off
    set long 90000
    set serveroutput on
    spool test3.csv
    SELECT TO_CLOB(BFILENAME('TMP_DIR','x.csv')) FROM dual
    ;
    spool off
```

## Retrieve a CLOB

The CSV strings can be concatenated into a CLOB with CR/LF between each row. The resulting CLOB can be
attached to an e-mail, written to a file or inserted/updated to a CLOB column in a table. Perhaps
added to a zip archive. There are many possibilites once you have the CSV content in a CLOB.
See *test/test2.sql* for an example function you can call in a SQL select.

```sql
    DECLARE
        l_clob  CLOB;
        l_src   SYS_REFCURSOR;
        l_csv   app_csv_udt;
    BEGIN
        OPEN l_src FOR SELECT * FROM hr.departments ORDER BY department_name;
        l_csv := app_csv_udt(
            p_cursor        => l_src
            ,p_num_format   => '099999'
        );
        l_clob := l_csv.get_clob(p_do_header => 'Y', p_separator => '|');
        ...
    END;
```

## Read from TABLE Function

You can use SQL directly to read CSV strings as records from the TABLE function *get_rows*, perhaps
spooling them to a text file with sqlplus. The following is *test1.sql* from the *test* directory.

```sql
    set echo off
    set linesize 200
    set pagesize 0
    set heading off
    set trimspool on
    set feedback off
    spool test1.csv
    SELECT a.column_value FROM TABLE(
            app_csv_udt.get_rows(CURSOR(SELECT * FROM hr.departments ORDER BY department_name)
                                ,p_separator=> '|'
                                ,p_do_header => 'Y'
                                )
        ) a
    ;
    spool off
```

## Process Results in a Loop

Although you could run a SELECT from the TABLE function *get_rows* in an implied cursor loop, 
you can also simply step through
the results the same way the preceding methods do. Perhaps you have a more involved use case such
as sending the resulting rows to multiple destinations, or creating a trailer record.

```sql
    DECLARE
        l_src   SYS_REFCURSOR;
        l_rec   VARCHAR2(32767);
        l_file  UTL_FILE.file_type;
        l_csv   app_csv_udt;
    BEGIN
        OPEN l_src FOR SELECT * FROM hr.departments;
        l_csv := app_csv_udt(
            p_cursor                => l_src
            ,p_quote_all_strings    => 'Y'
        );
        l_rec := l_csv.get_next_row;
        IF l_rec IS NOT NULL THEN -- do not want to write anything unless we have data from the cursor
            l_file := UTL_FILE.fopen(
                filename        => 'my_file_name.csv'
                ,location       => 'MY_DIR'
                ,open_mode      => 'w'
                ,max_linesize   => 32767
            );
            UTL_FILE.put_line(l_file, l_csv.get_header_row);
            LOOP
                UTL_FILE.put_line(l_file, l_rec);
                l_rec := l_csv.get_next_row;
                EXIT WHEN l_rec IS NULL;
            END LOOP;
            UTL_FILE.put_line(l_file, '---RECORD COUNT: '||TO_CHAR(l_csv.get_row_count));
            UTL_FILE.fclose(v_file);
        END IF;
    END;
```

# Test Directory

The *test* directory contains sql files and corresponding CSV outputs. These samples demonstrate much of the 
available functionality. *test3* explores quoting.

# Manual Page

## Constructor

Creates the object using the provided cursor. Prepares for reading and converting the result set.

```sql
    CONSTRUCTOR FUNCTION app_csv_utd(
        p_cursor                SYS_REFCURSOR
        ,p_separator            VARCHAR2 := ','
        ,p_num_format           VARCHAR2 := 'tm9'
        ,p_date_format          VARCHAR2 := 'MM/DD/YYYY'
        ,p_bulk_count           INTEGER := 100
        ,p_quote_all_strings    VARCHAR2 := 'N'
    ) RETURN SELF AS RESULT
```

*p_num_format* and *p_date_format* are passed to *TO_CHAR* for number and date/time formats respectively.

When *p_quote_all_strings* starts with a 'Y' or 'y', then all character values are enclosed in double quotes,
not just the ones that contain a separator or newline.

Note that numbers and dates can be double quoted. Consider a number format of '$999,999.99' with a comma separator,
or perhaps a date format that includes a colon with a colon separator.
Luckily, Excel figures that all out just fine.

## get_rows

This function is not a member method. It calls the constructor for you and uses the object internally.
The appropriate use case is to call it from SQL using the construct:

```sql
    SELECT column_value FROM TABLE(app_csv_udt.get_rows(
                                    CURSOR(SELECT * FROM hr.departments)
                                                       )
                                  )
    ;
```

All of the arguments for the constructor are present along with the flag for whether or not to
put the column header names in the first row.

```sql
   STATIC FUNCTION get_rows(
        p_cursor                SYS_REFCURSOR
        ,p_separator            VARCHAR2 := ','
        ,p_do_header            VARCHAR2 := 'N'
        ,p_num_format           VARCHAR2 := 'tm9'
        ,p_date_format          VARCHAR2 := 'MM/DD/YYYY'
        ,p_bulk_count           INTEGER := 100
        ,p_quote_all_strings    VARCHAR2 := 'N'
    ) RETURN arr_varchar2_udt PIPELINED
```

Although you can call this from PL/SQL and iterate through the loop, that will instantiate the entire
result set array before returning control to your program. The *PIPE ROW* optimization is strictly for
the SQL engine, not PL/SQL. If you are thinking about using it that
way, you might be better off with the technique shown in the use 
case [Process Results in a Loop](#process-results-in-a-loop). That said, it is not uncommon to see functions
like this called in PL/SQL in an implicit FOR loop.

```sql
    FOR r IN (SELECT column_value FROM TABLE(app_csv_udt.get_rows(
                                    CURSOR(SELECT * FROM hr.departments)
                                                                 )
                                            )
    ) LOOP
        -- do something with r.column_value
    END LOOP;
```

## get_clob

Returns a CLOB containing all of the rows separated by CR/LF. If the cursor returns no rows, the
returning CLOB is NULL, even if a header row is called for.

```sql
   MEMBER FUNCTION get_clob(
        SELF IN OUT NOCOPY  app_csv_udt
        ,p_do_header            VARCHAR2 := 'N'
    ) RETURN CLOB
```
After executing this method, you cannot restart the cursor. The only practical method remaining for 
the ojbect is *get_row_count*.

## write_file

Writes the CSV strings to the file you specify using the appropriate line ending for your database host OS.
If the cursor returns no rows, the file is created/replaced empty, even if a header row is called for.

```sql
    MEMBER PROCEDURE write_file(
        SELF IN OUT NOCOPY      app_csv_udt
        ,p_dir                  VARCHAR2
        ,p_file_name            VARCHAR2
        ,p_do_header            VARCHAR2 := 'N'
    )
```
After executing this method, you cannot restart the cursor. The only practical method remaining for 
the ojbect is *get_row_count*.

## get_row_count

Intended to be called after all fetch operations are complete, it returns the number of data rows
processed (does not include the optional header row in the count).

```sql
    MEMBER FUNCTION    get_row_count(
        SELF IN OUT NOCOPY  app_csv_udt
    ) RETURN INTEGER
```

## get_header_row

Returns the CSV string of column header names. These will be double quoted if the name contains a separator character.
You can call this method any time after the constructor, but before the last fetch. Generally it is 
called either before the first fetch or immediately after.

```sql
    MEMBER FUNCTION    get_header_row(
        SELF IN OUT NOCOPY  app_csv_udt
    ) RETURN VARCHAR2
```

## get_next_row

Returns the next record from the cursor converted into a CSV string. There is no newline.

When all rows have been 
processed, it returns NULL, and the cursor cannot be restarted. The only practical method remaining for 
the ojbect at that point is *get_row_count*.

```sql
    MEMBER FUNCTION    get_next_row(
        SELF IN OUT NOCOPY  app_csv_udt
    ) RETURN VARCHAR2
```

## set_separator

Normally one would set this when calling the constructor.

```sql
    MEMBER PROCEDURE   set_separator(
        SELF IN OUT NOCOPY  app_csv_udt
        ,p_separator        VARCHAR2
    )
```

## set_date_format

Normally one would set this when calling the constructor.

```sql
    MEMBER PROCEDURE   set_date_format(
        SELF IN OUT NOCOPY  app_csv_udt
        ,p_date_format      VARCHAR2
    )

```
# set_num_format

Normally one would set this when calling the constructor.

```sql
    MEMBER PROCEDURE   set_num_format(
        SELF IN OUT NOCOPY  app_csv_udt
        ,p_num_format       VARCHAR2
    )
```