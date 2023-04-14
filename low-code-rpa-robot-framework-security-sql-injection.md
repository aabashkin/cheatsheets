# Cheatsheet - Low Code Robot Framework / RPA Framework: SQL Injection (SQLi)

- [Cheatsheet - Low Code Robot Framework / RPA Framework: SQL Injection (SQLi)](#cheatsheet---low-code-robot-framework--rpa-framework-sql-injection-sqli)
  - [Prevention](#prevention)
    - [Parameterized Queries](#parameterized-queries)
    - [Stored Procedures](#stored-procedures)
    - [Escaping](#escaping)
    - [Input Validation](#input-validation)


## Prevention

* Parameterized queries
* Stored Procedures
* Escaping
* Input Validation
	
None of these techniques are novel or unique to our Robot Framework example. They are tried and tested methods of avoiding SQL injection. The only difference is how they are applied specifically within Robot Framework.

Let's go step by step.

<br>

### Parameterized Queries

When generating the query, replace the `Format String` keyword with the `Set Variable` keyword and replace the `VALUES ('{}')` placeholder with a `VALUES (\%s)` placeholder.

Next, add a `data` parameter containing untrusted input, `${student_name}` in this case, to the `Query` [keyword](https://robocorp.com/docs/libraries/rpa-framework/rpa-database/keywords#query).

The data parameter accepts [tuples](https://www.w3schools.com/python/python_tuples.asp), so even if you have a single input item you must still add an extra empty item at the end in order to execute the query. Simply providing `("${student_name}")` won't work, `("${student_name}", )` will.

```RobotFramework
*** Tasks ***

Insert Students Into Database Using Parameterized Query
    @{list_students}=    Get Students From Excel File    resources/new_students.xlsx
    FOR    ${student_name}    IN    @{list_students}
        ${query}=    Set Variable    INSERT INTO students (name) VALUES (\%s);  
        Query    ${query}  data=("${student_name}", )
    END
```

**Please note that every database has a different style of placeholder syntax.** Our example contains the syntax used by PostgreSQL. Other databases rely on a different syntax, so please refer to this [guide](https://bobby-tables.com/python) when necessary.

When we run this example, the platform recognizes the **clear separation between data and code**, thus eliminating the attacker's ability to manipulate the function of the query itself. This separation is the **core principle** for avoiding any type of injection vulnerability.

<br>

### Stored Procedures

Start by creating a stored procedure in the database.

```sql
CREATE PROCEDURE insert_student(student_name VARCHAR(255))
LANGUAGE SQL
AS $$
    INSERT INTO students (name) VALUES (student_name);
$$;
```

Next, utilize the `Call Stored Procedure` [keyword](https://robocorp.com/docs/libraries/rpa-framework/rpa-database/keywords#call-stored-procedure) in your code.

```RobotFramework
*** Tasks ***

Insert Students Into Database Using Stored Procedure
    @{list_students}=    Get Students From Excel File    resources/new_students.xlsx
    FOR    ${student_name}    IN    @{list_students}
        @{params}     Create List   ${student_name}
        Call Stored Procedure   insert_student  ${params}
    END
```

**Please note that due to an [issue](https://www.psycopg.org/docs/cursor.html?highlight=callproc#cursor.callproc) in the `psycopg` Python module this approach is currently incompatible with any Postgres 11+ database.** Instead, one can use the `Query` keyword along with the `CALL` instruction. However, in order to properly generate this query string one must use parameterized queries as described in the previous section. For the sake of simplicity it may make more sense to just use parameterized queries and avoid the extra overhead of obtaining access to the database and creating a stored procedure. However, if a stored procedure already exists in your use case and you would like to leverage it, see the example below.

```RobotFramework
*** Tasks ***

Insert Students Into Database Using Stored Procedure (Postgres)
    @{list_students}=    Get Students From Excel File    resources/new_students.xlsx
    FOR    ${student_name}    IN    @{list_students}
        ${query}=    Set Variable    CALL insert_student (\%s)
        Query    ${query}  data=("${student_name}", )
    END
```

This example also conforms to the principle of separating code and data. Manipulation of the query is no longer possible.

<br>

### Escaping

Escaping neutralizes any special control characters in our input, thus preventing it from altering the query. Apply escaping by surrounding the `{}` placeholder with double dollar signs `$$`.

Since Robot Framework recognizes a dollar sign followed by a curly brace as a variable declaration we have to add an extra backslash `\` to `$${}$$`, so the final result should be `$\${}$$`.

```RobotFramework
*** Tasks ***

Insert Students Into Database Using Query With Escaping
    @{list_students}=    Get Students From Excel File    resources/new_students.xlsx
    FOR    ${student_name}    IN    @{list_students}
        ${query}=    Format String    INSERT INTO students (name) VALUES ($\${}$$);    ${student_name}
        Query    ${query}
    END
```

**Please note that every database has a different style of escaping.** Our example contains the syntax used by [PostgreSQL](https://www.postgresql.org/docs/15/sql-syntax-lexical.html#id-1.5.3.5.9.7.2). Please refer to your database's documentation for guidance relevant to your use case.

Just like the techniques mentioned above, this example separates code and data and secures our query.

<br>

### Input Validation

As an example, let's say that we've decided that valid student names for our database should contain only letters from the Latin alphabet and hyphens. We use this rule to create a regular expression `^[a-zA-Z-]+$` and apply it to each student name using the `Should Match Regexp` keyword.

```RobotFramework
*** Tasks ***

Insert Students Into Database Query With Input Validation
    @{list_students}=    Get Students From Excel File    resources/new_students.xlsx

    ${pattern}=    Set Variable    ^[a-zA-Z-]+$

    FOR    ${student_name}    IN    @{list_students}
        TRY
            ${result}=    Should Match Regexp    ${student_name}    ${pattern}
            ${query}=    Format String    INSERT INTO students (name) VALUES ('{}');    ${student_name}
            Query    ${query}
        EXCEPT   * does not match *    type=glob
            Log To Console    Input validation failed. Student name does not meet requirements.\n
        END
    END
```

**Please note that input validation is most effective when heavily constrained.** The example above could actually become vulnerable if we allowed additional special characters, such as `'`, `)`, and `;`. Deciding if input validation is sufficiently constrained is a difficult topic, even for tech savy users. For this reason, it is recommended to avoid relying entirely on input validation, and instead use it as a secondary security control. That being said, input validation provides benefits beyond just security, such as ensuring data accuracy and consistency, so it is still a good idea to consider it.

