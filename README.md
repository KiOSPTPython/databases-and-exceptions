# Databases and exceptions
## Store my data
Let's assume we're managing a botnet and wish to hold information about our bots. One part of our attack campaign will be this program that can show, each time we access it, what bots are under our control. The properties of each bot will be their IP, MAC and OS but we'll also store last_pinged, as a date, and is_alive, as a boolean. 


Write a program that will be able to display and add to our list of bots. Each bot description will be stored in a dictionary, the keys being IP, MAC, OS, last_pinged and is_alive. For our purpose above, we'll also need 4 functions, `save`, `load`, `add` and `display`. The function `save` receives, as an argument, a list of dictionaries (our bots) and saves the list to the file `~/bots`. The function `load` reads the file `~/bots` and returns a list of dictionaries. The function `add` receives, as an argument, a list of dictionaries, asks the user for details about a new bot machine (IP, MAC and OS), adds it as a new dictionary in the given list with last_pinged as now and is_alive as True. The function `add` uses `save` to save the updated list to disk. Lastly, the function `display` should load the bots from disk using `load` and print them to the screen. The structure of the file is of your choice.
>Don't forget to smile.

## Database
Let someone else store my data. We reached the conclusion that managing data storage is a headache. When talking about data, there are other options that we may want to use, such as searching through it. Searching through the data, even if it is limited to field values and nothing complex, becomes a large task in itself. All this time spent writing code that isn't implementing our project logic can be argued as time wasted. In come databases to save the day. Because this is a requirement many have, companies have answered it with dedicated products that manage storage for either software or direct use. From the simple Excel file to the memory intensive Reddis, that stores data in memory and provides blazing fast read/write speeds. This doesn't mean that Reddis is better than Excel. Different databases are suitable for different tasks and should be employed where they best fit the problem at hand rather. We shouldn't jump the gun just because of cool keyword such as "in memory" and "blazing fast read/write". As useful as it is to save/load data, it is almost never practical to create a new database software.


We'll be using MySQL for our exercises. MySQL is a popular open source database used in many small to medium applications and is available on Kali, by default. MySQL is especially popular for newcomers because of its wide use and abundance of guides online. It is a relational database and uses SQL (use [SQL Zoo](https://sqlzoo.net/) to exercise SQL). What we're going to do is read/write to and from our own MySQL database using Python. To do so, we'll first install the `mysql-connector-python` package that enables us to easily communicate with our MySQL instance using Python. 
```bash
pip install mysql-connector-python
```
This is the official package from Oracle but, if you'd like to refine your package choice, run a [benchmark](https://github.com/methane/mysql-driver-benchmarks) on any recommended alternative. Another package candidate is `mysqlclient` but its installation may need the Linux package `mysql-connector-c++ libmysqlclient-dev python-mysqldb python-dev`, depending on your distribution. 

## SQL in Python and programming
Once we have our MySQL database set up, with some data in it, and our Python has the module to connect to it, we'll be able to execute SQL commands to our DB. Oracle has some [examples](https://dev.mysql.com/doc/connector-python/en/connector-python-examples.html) on using MySQL with Python. Let's look at the connection example, that will help us validate the groundwork, and go on from there.

```python
import mysql.connector

CONFIG = {'user': 'root',
          'host': '127.0.0.1',
          'password': '123456'}


cnx = mysql.connector.connect(**CONFIG)
cnx.close()

```
The function `mysql.connector.connect` accepts connection parameters that instruct it where to connect using which credentials. It returns a connection object, that we can later use but, we close immediately afterwards. We've stored the connection details in a dictionary and used it as an expanded key:value pairs. In other words, this is a shorthand way to write `key1=value1, key2=value2,...`. This is the equivalent to 

```python
import mysql.connector


cnx = mysql.connector.connect(user='root', host='127.0.0.1', password='123456')
cnx.close()

```
Change the credentials according to your installation. The default password is an empty one. Let's look at some SQL examples fired from a Python code.

```python
import mysql.connector

CONFIG = {'user': 'root',
        'host': '127.0.0.1',
        'password': '123456',
        'database': 'world'}


cnx = mysql.connector.connect(**CONFIG)
cur = cnx.cursor()

city_name = 'Kabul'
cur.execute("SELECT population FROM city WHERE name=%s", (city_name, ))
row = cur.fetchone()
print(row[0])
cnx.close()

```
I've used Oracle's world database, available [here](https://dev.mysql.com/doc/index-other.html), to search through some of the data and fetch the population of Kabul. The `cnx.cursor()` method returns a cursor object through which we can issue SQL statements, the standard way of working; Although, using the connection object directly also works (don't do that). Once we created our cursor, we executed an SQL statement using `cur.execute` but, also, an unfamiliar set of arguments. Our first guess would be that we used our city name as a single parameter in the SQL. Which is true. Question is, why didn't we just use string formatting as we have up until now. What's the difference? Let's examine this.

```python
city_name = input("City? ")

sql = f"SELECT population FROM city WHERE name='{city_name}'"
cur.execute(sql)
row = cur.fetchone()
print(row[0])

```
Had the user answered with a valid city name, we'd probably get away with this vulnerable code. But, if the input contains a single quote mark `'`, the code break. If the input contains malicious SQL code, it might be a lot worse. Imagine the user typed in `bad'; DROP DATABASE world;-- `.

```python
>>> city_name = input("City? ")
>>> print(city_name)
bad'; DROP DATABASE world;-- 
>>> sql = f"SELECT population FROM city WHERE name='{city_name}'"
>>> print(sql)
SELECT population FROM city WHERE name='bad'; DROP DATABASE world;-- '
>>> cur.execute(sql)
>>> row = cur.fetchone()
>>> print(row[0])

```
This would execute two SQLs in a sequence. The first being a simple SELECT but, the second being a DROP of our database. This is a critical error on our part and may lead to dire consequences. For this reason, we used something called `bind parameters`. They are the correct and secure way to pass variables. Never format your own SQL using variables but use bind parameters and or prepared statements. Even if these are your own variables, unaffected by user input, it is a good habbit to develop early on.


The term bind parameters refers to parameters that have been correctly escaped according to their expected data type using the DB driver. This is not something we do ourselves but use provided tools that do it well. In the example above, the single quote would become two single quotes that translate, in SQL, to the value of a single quote and making it harmless `bad''; DROP DATABASE world;-- `. This is in contrastto it being translated into part of the SQL syntax. In other words, it defuses a ticking bomb. A standard practise is to use prepared statements. These enable repeated execution of the same statement, with different data, efficiently but, more importantly for us, also strictly define what the statement can do. The open source package for MySQL doesn't have this implemented. Instead, we can use parameter markers for bind parameters to shield our program from malicious input.

```python
city_name = input("City? ")

cur.execute("SELECT population FROM city WHERE name=%s", (city_name, ))
```

This will use the string `SELECT population FROM city WHERE name=%s` in conjunction with the variable `city_name` by escaping the variable value, replacing `%s` with the escaped value and execute the result.

>Refactor your botnet rooster code to use a MySQL database.

## Exceptions
The concept of exceptions exists in Python as both a form of indicating there was an error, to the program, and a form of code flow control, for us when we write code. Imagine it as a more sophisticated `if else`, when talking about control. Although exceptions exist in other programming languages, as well, in Python it was engineered as a tool from the ground up to be used as a control mechanism. And, as such, it is faster than `if else`. Exceptions are raised, "created", from code and aren't a hidden internal tool. This, in turn, means we can raise our own exceptions with something as simple as `raise ValueError("Bad credentials.")`. What happens next is the function, where the interpreter encountered this line, will halt and propagate the exception upward in the chain of function calls that led to that line. The exception, no matter the type, will continue to halt functions and propagate upwards until it either reaches the initiating script or encounters a `try except` clause. The `try except` is said to "catch" exceptions raised in code it encompasses. Let's look at some examples.

```python
from time import sleep


def forever():
    """Prints the same thing forever and ever and ever."""
    while True:
        print("And going and going and going...")
        sleep(1)


try:
    forever()
except KeyboardInterrupt as err:
    print("Stopped with Ctrl+C.")

print("Quitting.")
```
We have an endless loop here that can only be stopped by killing the process. The most familiar method is Ctrl+C. When we press Ctrl+C, an exception of type `KeyboardInterrupt` is raised, wherever the code was busy executing something. When the exception is raised, it stops the function execution and begins propagating. Our function is inside a `try except` clause and we handle any `KeyboardInterrupt` exceptions raised inside the `try except` clause. The normal outcome, with exceptions, we're familiar with is a print of the error traceback and not much else. Here, we know this isn't an error that needs debugging so we muffle the traceback and only print a short message.
>Add, to your botnet code, the ability to exit using Ctrl+C.


The `try except` clause, for now, has 2 parts. One, the `try` block, where normal code flow is executed and exceptions may rise. The second, the `except` block, is where we declare what exception types we handle and how. Whatever is written inside the `try` block is executed normally and in sequence, so long as no exception is raised in one of the lines. If no exceptions are raised within the `try` block, the interpreter will execute everything inside it and continue executing code after the `except` block. It will ignore the `except` block. If an exception is raised inside the `try` block, it will stop executing anything written after that line inside the `try` block and jump to the `except` part and see if we declared willingness to handle the raised exception. We can handle multiple types of exceptions, each with its own code. Let's see this in action

```python
USERS = {'Gabe': {'id': '100200300', 
                  'email': "gave@valve.com", 
                  'power_level': 500},
         'Bob': {'id': '200300100',
                 'email': "g0tmi1k@kioptrix.com",
                 'power_level': 9001},
         'Midy': {'id': '300100200',
                  'email': 'htm1dy@gmail.com',
                  'power_level': 240}
}


def display():
    """Displays a user's details."""
    name = input("User to display? ")
    print(USERS[name])


def insert():
    """Asks for a new user's details."""
    name = input("Name? ")
    new_id = input("ID? ")
    email = input("Email? ")
    power_level = input("Power level? ")

    new_guy = {name: {'id': new_id, 'email': email, 'power_level': power_level}}
    USERS.update(new_guy)


def main():
    """Starting point."""
    try:
        while True:
            action = input("To insert or display? ")
            if action == 'insert':
                insert()
            elif action == 'display':
                display()
    except KeyboardInterrupt:
        print("Quitting mini DB explorer.")
    except KeyError as err:
        key = err.args[0]
        print(f"Unknown user {key}.")
        main()
```
Do not use this as a template for a program, calling main from within main. To handle more than one type of exception, we add another block of `except`. This time, we added the `KeyError` exception that simply prints a message before starting the program again. We're already familiar with the first exception, KeyboardInterrupt. The second, KeyError, is raised when the interpreter tries to get a dict's value at a key that doesn't exist. In other words, `d = {1: 2}; d['hello']` will raise KeyError. Unless an exception is raised inside the `try` block, nothing will stop the normal code flow inside it. If an exception is raised and it matches one of the exception types we handle, the relevant `except` block will execute. Once we type into the program an unknown user's name, that isn't inside the `USERS` dictionary, we'll see a "Unknown user X." message. Once we hit Ctrl+C, the program will exit. Another thing we've used here is the error object. Most of the time, we'll want to catch exceptions with `except ExceptionType as err` and use `err` in our debug message. Here `err` is just a variable with the error object assigned to it. Unless we're absolutely certain we don't need it, we can't ignore it. The principal reason for this is to create a correct coding habbit that avoids suppressing errors by mistake and a big debugging headache. How the object is used depends on the package that raised it but, usually it contains a `args` tuple with relevant data. 
>Refactor your botnet code to ask the user, on program start, for the MySQL credentials as long as they are incorrect using `try except`.


We can also raise exceptions on our own and handle them in whatever way we want. Let's see an example.
```python
USERS = {'placeholder': 'placeholder'}


def display():
    print("Dummy function.")


def insert():
    print("Dummy function.")


def work():
    """Starts the mini DB program."""
    if len(USERS) > 5:
        raise ResourceWarning("User DB is full.")

    action = input("Insert or display? ").lower()
    if action == 'insert':
        insert()
    elif action == 'display':
        display()


def main():
    """Starting point."""
    try:
        while True:
            work()
    except KeyboardInterrupt:
        print("Quitting mini DB explorer.")
    except ResourceWarning as err:
        print(f"{err}")
```
The syntax for raising errors is `raise Exception("Error message")` which is what we've done above with `ResourceWarning`. Outside the function, we catch that type of exception and act in a manner we see fit. There's no difference wether we raised the exception or some other package.
>Refactor the code above to check for valid input (insert/display) using a dictionary and the KeyError exception instead of the `if else` inside `work`.

There's also the possibility to re-raise an exception that's currently propagating.
```python
def challenge_math():
    """Trying something that never worked before!"""
    try:
        x = 42/0
        print("Yes! Success!")
        return x
    except ZeroDivisionError:
        print("No :( Failed!")
        raise


def main():
    """Starts the demo."""
    try:
        print("Did you make it?")
        challenge_math()
    except ZeroDivisionError:
        print("Quitting.")
```
Although we caught the `ZeroDivisionError` exception and handled it, we decided to continue propagating it. Had `main()` not caught it, the program would have quit with an exit code of 1 and printed a traceback of the error. We can also define our own exceptions and raise them.
```python
class CustomError(Exception):
    pass


def work():
    raise CustomError("Error message")


def main():
    try:
        work()
    except CustomError as err:
        print(f"{err}")
```
Python comes with an abundance of [built-in exception](https://docs.python.org/3/library/exceptions.html) which cover most of our needs. Do not hurry to create your own exceptions but, if you have to, this is how you would do it.


Lastly, you should know the exception type `Exception` is the base of all exceptions (except system exiting errors) and, if we use it in `try except`, we'll catch all exceptions. Use this type only as a last resort or for small, local test. If you avoid looking up what exceptions are raised by the code you use (either by experimenting or going through the docs) and build a habbit of using `Exception`, you'll substantially increase the time it takes to debug your code in your career and make it harder for you to work with others. These are the main reasons it is against standard practice. 

## Finals
Write a program that will be the backbone of a message board. The program will use a MySQL database and exceptions to handle user login, registration and comments. It should allow a user to login using credentials found in your `users` table, register a new user into the same table and write a comment in the `comments` table. Remember to carefully use any given user data. You do not want to write code vulnerable to SQLi. To speed up your code and see an interesting `try except` example, hold all users in a local dictionary. Only if a user doesn't exist already, should you add it to the DB. Use exceptions to accomplish this. Below is some code to get you started.


Database setup.
```SQL
CREATE TABLE IF NOT EXISTS users(
    id INT NOT NULL AUTO_INCREMENT,
    name VARCHAR(128),
    email VARCHAR(512),
    password VARCHAR(256),
    PRIMARY KEY (id)
);

CREATE TABLE IF NOT EXISTS comments(
    comment_id INT NOT NULL AUTO_INCREMENT,
    user_id INT NOT NULL,
    comment VARCHAR(4096),
    PRIMARY KEY (comment_id),
    FOREIGN KEY (user_id) REFERENCES users(id)
);
```


Inserting new data
```SQL
INSERT INTO users (name, email, password) VALUES ('John', 'example@gmail.com', 'nutella');

INSERT INTO comments (user_id, comment) VALUES (31415, 'The quiter you become.');
COMMIT;
```


```python
USERS = {}


def db_conn():
    """Returns an active DB connections. Remember to close it."""
    pass


def setup_db():
    """SQLs that create the schema."""
    pass


def load_users():
    """Loads all users into a local dict."""
    pass


def login():
    """Attempts to login according to known users."""
    pass


def register():
    """Registers a new user."""
    pass


def comment(user_id, comment):
    """Inserts a comment into the comments table."""
    pass


def main():
    pass

```