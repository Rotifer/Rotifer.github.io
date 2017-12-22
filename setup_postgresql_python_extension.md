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
![alt text]("figure_plpythonu_extension.png" "")


