# Using Python inside PostgreSQL
OWhen you use PostgreSQL, you are not limited to SQL and PL/pgSQL when you need to extend or customise functionality. Its extension mechanism allows you to use Perl, Python, JavaScript and many more *within* the database. I try to use PL/pgSQL as much as possible because:
* It works seamlessly with SQL
* It is always available
* I already knew Oracle PL/SQL so it is very familiar

However, when I need to use external services, a REST API for example, I use what is termed the *untrusted* version of the Python extension. The word *untrusted* sounds alarming but all it really means is that this version of the extension can manipulate external resources such as the file system and email. PL/pgSQL, like the *trusted* version of the Python extension, is effectively sand-boxed and unable to access these resources. Given its abilities, you will not be surprised to learn that only super-users can use it to write functions.

## Installation
I run my production server on RHEL 7. To make the Python 2.7 extension available, I had to install two pieces of software as follows:

```sh
$ sudo yum install postgresql-contrib postgresql-plpython
$ sudo yum install postgresql96-contrib postgresql96-plpython
```

I had already set up Python with the normal toolset including the *pip* installer.

All is now ready to enable the extension in the database where I need Python functionality. To do this I connected to the database using the pgAdmin client and navigated to the *extensions* and clicked *Create* and then navigated through the dialog to the *plpythonu* extension and selected it. Having done this, I now have it available as can be seen in this image:
![alt text]("https://github.com/Rotifer/Rotifer.github.io/blob/master/figure_plpythonu_extension.png" "Extension added")

## Putting the plpythonu extension to the test
Now that I have installed the *plpythonu* extension, I am going to use it to call a REST API. Before I do this, I need to install the *requests* Python package. I do this using *pip* from the UNIX shell like so:

```sh
$ pip install requests
```

Now, back in PostgreSQL, I am going to create a test function to call a ["Fake Online REST API for Testing and Prototyping"](https://jsonplaceholder.typicode.com/).

```python
CREATE OR REPLACE FUNCTION get_rest_api_jsonb()
RETURNS JSONB
AS
$$
	import requests
	import json
	import sys

	server = 'http://jsonplaceholder.typicode.com/todos'
	response = requests.get(server, headers={ "Content-Type" : "application/json"})
	if not response.ok:
		response.raise_for_status()
		sys.exit()
	return json.dumps(response.json())
$$
LANGUAGE 'plpythonu'
STABLE
SECURITY DEFINER;
```

Everything between the opening and closing doube dollars is Python code. It is enclosed in the same function definition code as used for PL/pgSQL except for the *LANGUAGE 'plpythonu'* line used to identify the language that implements the function.
We can now call the function in an identical manner to that used for PL/pgSQL:
```
SELECT * FROM get_rest_api_jsonb();
```
And, sure enough, it returns a large chunk of JSONB.
For future reference, here is the SQL to add a structured comment:

```sql
SELECT create_function_comment_statement('get_rest_api_jsonb',
                                         NULL,
                                         'Demonstrates how the *plpythonu* extension can be used to implement a function to call a REST API.',
                                         $$SELECT * FROM get_rest_api_jsonb();$$,
                                         'This is a demo function only. It calls a mock REST API used for testing and mocking. ' ||
                                         'See: (https://jsonplaceholder.typicode.com/)');
```

So now I have the full power and scope of Python with all its great libraries at my disposal from *within* my PostgreSQL database. You might wonder why I would bother with PL/pgSQL at all but, as I stated earlier, PL/pgSQL is still my preferred language when working within PostgreSQL and I only use Python when I have to. When doing work *ouside* the database, however, Python is my "go-to" language. Functions defined using Python can be called by PL/pgSQL functions just as easily as if if they were written in PL/pgSQL:

```sql
CREATE OR REPLACE FUNCTION get_rest_result_as_table()
RETURNS TABLE(user_id INTEGER, id INTEGER, completed BOOLEAN, title TEXT)
AS
$$
BEGIN
  RETURN QUERY
  SELECT
   ((JSONB_ARRAY_ELEMENTS(rest))->>'userId')::INTEGER user_id,
   ((JSONB_ARRAY_ELEMENTS(rest))->>'id')::INTEGER id,
   ((JSONB_ARRAY_ELEMENTS(rest))->>'completed')::BOOLEAN completed,
   (JSONB_ARRAY_ELEMENTS(rest))->>'title' title
  FROM
    (SELECT * FROM get_rest_api_jsonb() rest) sq;
END;
$$
LANGUAGE plpgsql
VOLATILE
SECURITY DEFINER;
SELECT create_function_comment_statement('get_rest_result_as_table',
                                         NULL,
                                         'Calls a *plpythonu* function and returns its JSON return value as a table.',
                                         $$SELECT * FROM get_rest_result_as_table();$$,
                                         'Demo only. Shows how to call a*plpythonu* function, extract JSON elements and cast them to PostgreSQL types. ' ||
                                         'The JSON it processes is an array of objects. The in-built function *JSONB_ARRAY_ELEMENTS* extracts each ' ||
                                         'each object into its own row and the *->>* JSONB operator is then used to extract the values as TEXT that is then cast ' ||
                                         'as appropriate.');
```

I will discuss the JSONB processing functions used here in a later post.

## Summary
I have shown how the *plpythonu* extension can be installed to perform tasks that are impossible with PL/pgSQL. The example given here uses a Python function to call am external REST API that returns JSON. I called this function in PL/pgSQL where I parsed the returned JSON into a table. I think this demonstrates one of PostgreSQL's greatest features: its extensibility! If a task is not possible in SQL or PL/pgSQL, you can use the extension mechanism to access very powerful languages like Python to do it for you. I have used Python here because I know it and like it. I have also used the JavaScript extension called [plv8](https://pgxn.org/dist/plv8/doc/plv8.html). This one seems to be very popular, especially with people who do a lot of JSON manipulation. It also fits well when PostgreSQL is used as a back-end in JavaScript-based stacks for web applications. There is also an R extension that I have not used but that must surely appeal to those who do a lot of statistical manipulation. Bottom line: you have a great range of options to extend PostgreSQL functionality.


## Contact information
Michael Maguire  
Written: 2017-12-22  
Email: mick@javascript-spreadsheet-programming.com  
Twitter: https://twitter.com/michaelfmaguir1
