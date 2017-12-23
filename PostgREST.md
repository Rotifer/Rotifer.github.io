# PostgREST

[PostgREST](https://postgrest.com/en/v4.3/) is a really easy way to get a REST API up-and-running for your database. I am going to write some rough notes now on how I set this up for my main PostgreSQL database running on RHEL 7. I will flesh this out later with more examples of how to use the REST API.

## Installation
* Make sure that the account to be used by PostgREST is in the *.pgpass* file and that the *.pgpass* file has permissions **0600**. See [Pgpass](https://wiki.postgresql.org/wiki/Pgpass).
* Install PostgREST according to the documentation:
  - 981  git clone https://github.com/begriffs/postgrest.git
  - cd postgrest/
  - ```stack build --install-ghc --copy-bins --local-bin-path /usr/local/bin``` May require some Haskell installs
  - ```which postgrest``` Verify install!
  - ```mkdir postgrest_confs``` Place config files in this directory
  - Create a PosgREST config file (contents shown below)
  ```cd postgrest_confs```
  - ```postgrest <schema_name>.conf``` Start PostgREST

### The PostgREST config file
```
db-uri = 
"postgres://<user_name>:@127.0.0.1:5432/<schema_name>"

# The name of which database schema to expose to REST clients
db-schema    = "<schema_name>"

db-anon-role = "<user_name>"
```

### Testing the PostgREST REST API
I created a very simple view in the schema denoted by *<schema_name>* above that just displays some database server information using built-in PostgreSQL commands or functions. Here is the definition with a comment:

```
CREATE OR REPLACE VIEW <schema_name>.vw_test_postgrest AS
SELECT
  CURRENT_USER user_name,
  CURRENT_DATE execute_date,
  CURRENT_DATABASE();
COMMENT ON VIEW <schema_name>.vw_test_postgrest IS 'Displays some information from the database server. Used to check PostgREST is working.'
```

To test this REST end-point, I return to the UNIX shell and execute the following:

```
curl -i -H "Content-Type: application/json" -X GET http://localhost:3000/vw_test_postgrest
```

When I run this, I see the following:

```
HTTP/1.1 200 OK
Transfer-Encoding: chunked
Date: Tue, 19 Dec 2017 17:13:48 GMT
Server: postgrest/0.4.3.0 (effbec2)
Content-Type: application/json; charset=utf-8
Content-Range: 0-0/*
Content-Location: /vw_test_postgrest

[{"user_name":"<user_name","execute_date":"2017-12-19","current_database":"<database_name>"}]
```

If this is working properly for you, then you should see similar JSON output. I will flesh out the JSON discussion later. You will need to have *curl* or some other such tool installed to test your PostgREST server. Note: to suppress the HTTP headers just drop the *-i* option in the command above and then you will get the JSON data only.

## Making my PostgREST server available on my local machine
I have installed PostgREST on my RHEL machine and the example I gave above of calling it using *curl* was executed fromt he same machine that runs PostgREST. However, I would like to be able to run the REST server from other machines including my MacBook Pro. In order to do this I have to open the posrt that PostgREST is using (3000 by default). Here is how I did this:

```
$ firewall-cmd --zone=public --add-port=3000/tcp --permanent
$ firewall-cmd --reload
```
Then from my MacBook Pro, I re-ran the same *curl* command as above replacing *localhost* with the host name running my PostgREST:

```
$ curl -i -H "Content-Type: application/csv" -X GET http://<host_name>:3000/vw_test_postgrest
```

Tip: You can get the host name using ```uname -a``` command on the host itself.

### Trouble-shooting notes
I have encountered situations where the PostgREST server is running but not responding so in order to re-start it, I first have to kill the running server. Here is hoe I do this:

First I have to get the PID number for the running PostgREST instance like so

``` 
$ sudo netstat -plten |grep postgrest
```

This gives me this output:

```
tcp        0      0 0.0.0.0:3000            0.0.0.0:*               LISTEN      0          13534074   22494/postgrest
```

The PID I need is **22494**, that is the number preceding */postgrest*. 

I can do a hard kill as follows:

```
$ kill -9 22494
```

## Contact information
Michael Maguire  
Written: 2017-12-22  
Email: mick@javascript-spreadsheet-programming.com  
Twitter: https://twitter.com/michaelfmaguir1  