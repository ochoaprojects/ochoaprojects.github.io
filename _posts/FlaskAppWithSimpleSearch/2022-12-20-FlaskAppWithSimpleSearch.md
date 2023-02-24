---
title: Building a Search Feature in a Python Flask App
date: 2022-12-20 12:00:00 -500
categories: [Programming,Python]
tags: [python,flask]
---

Going through the steps on how to build a search feature in a Python Flask app. Flask is a popular microweb framework that makes it easy to build web applications in Python. We will use Jinja2 templates to create the user interface and a search function to retrieve search results from a database.

## Table of Contents
- [Creating a Flask App with Search Feature](#creating-a-flask-app-with-search-feature)
- [Creating Search Template](#creating-search-template)
  - [Optional CSS for Search](#optional-css-for-search)
- [Creating the Results Template](#creating-the-results-template)
- [Implementing Search Function to use a Database](#implementing-search-function-to-use-a-database)
- [Creating the Database](#creating-the-database)
- [Conclusion](#conclusion)
    - [Completed FlaskApp.py](#completed-flaskapppy)
    - [Completed Search HTML Template](#completed-search-html-template)
  - [Completed Results HTML Template](#completed-results-html-template)
    - [It Works!](#it-works)

# Creating a Flask App with Search Feature

```python
from flask import Flask, request, render_template

app = Flask(__name__)

@app.route('/')
def search_page():
    return render_template('search.html')

@app.route('/search', methods=['POST'])
def search():
    query = request.form['query']
    # Perform search and return results
    return render_template('results.html', query=query, results=results)

if __name__ == '__main__':
    app.run()
```

This Flask app has two routes: `/` and `/search`. The `/` route renders the `search.html` template, which should include a form with a search bar. The `/search` route handles the form submission and performs the search using the query entered by the user. The search results are then passed to the `results.html` template, which can display the results in a list or table format.

To use this app, we will need to create the `search.html` and `results.html` templates and place them in the `templates` directory. We will also need to implement the search function and define the `results` variable.

# Creating Search Template
We need to create a `template`. To do this, we need create a folder called `templates` in the root directory of the flask app. Our directory should look like this:

```bash
$ tree
.
└── FlaskAppProjectFolder
    └── templates
```

Let's start by creating a simple HTML landing page for our search bar called `search.html` and save it into the newly created `templates` folder with the following code:

```html
<!DOCTYPE html>
<html>
<head>
  <title>Search Page</title>
</head>
<body>
  <h1>Search</h1>
  <form action="/search" method="POST">
    <input type="text" name="query" placeholder="Enter search query">
    <button type="submit">Search</button>
  </form>
</body>
</html>
```

Now our directory should look like this:

```bash
$ tree
.
└── FlaskAppProjectFolder
    └── templates
        └── search.html
```

If you open that HTML file in a browser of your choice, you will see the following:

![Starting Search Page](/project-assets/FlaskAppWithSimpleSearch/starting-search-page.png)

This template includes a simple form with a text input field for the search query and a submit button. The form's action is set to `/search`, which is the URL that the form will submit the search query to. The method is set to `POST`, which means that the search query will be sent to the server as part of the HTTP request body.

You can customize this template further by adding additional form fields or styling the form with CSS. You can also use Flask's template inheritance feature to create a base template that includes the search form, and then extend the base template in other templates as needed.

## Optional CSS for Search
```html
<!DOCTYPE html>
<html>
<head>
  <title>Search Page</title>
  <style>
    form {
      width: 500px;
      margin: 0 auto;
      text-align: center;
      background-color: #eee;
      padding: 20px;
      border-radius: 5px;
    }
    input[type="text"] {
      width: 70%;
      padding: 12px 20px;
      margin: 8px 0;
      box-sizing: border-box;
      border: 2px solid #ccc;
      border-radius: 4px;
    }
    button[type="submit"] {
      width: 20%;
      background-color: #4CAF50;
      color: white;
      padding: 14px 20px;
      margin: 8px 0;
      border: none;
      border-radius: 4px;
      cursor: pointer;
    }
    button[type="submit"]:hover {
      background-color: #45a049;
    }
  </style>
</head>
<body>
  <h1>Search</h1>
  <form action="/search" method="POST">
    <input type="text" name="query" placeholder="Enter search query">
    <button type="submit">Search</button>
  </form>
</body>
</html>
```

This template includes some basic CSS styles to make the form look more visually appealing. The form has a fixed width, a centered layout, and a light grey background color. The input field and submit button have some basic styling applied to them, including a border, padding, and a hover effect for the submit button.

This is what that results of the CSS styling looks like when opening this HTML file in a browser:
![Search Page With CSS](/project-assets/FlaskAppWithSimpleSearch/search-page-with-css.png)

# Creating the Results Template
Next let's create a search results template with the filename `results.html` in the `templates` folder using the following code:

```html
<!DOCTYPE html>
<html>
<head>
  <title>Search Results</title>
</head>
<body>
  <h1>Results for: {{ query }}</h1>
  <ul>
    {% for result in results %}
      <li>{{ result }}</li>
    {% endfor %}
  </ul>
</body>
</html>

```

This template uses Jinja2 syntax to display the search results in an unordered list. The `query` and `results` variables are passed to the template from the `search` route in the Flask app.

Now our directory should look like this:

```bash
$ tree
.
└── FlaskAppProjectFolder
    └── templates
        ├── search.html
        └── results.html
```

# Implementing Search Function to use a Database

Now that we have a base flask app with our search and results templates created in the templates folder, we can add in the following block of code (After `app = Flask(__name__)` and Before `def perform_search(query):`) to implement our search function:

```python
import sqlite3

def perform_search(query):
    # Connect to the database
    conn = sqlite3.connect('database')
    c = conn.cursor()

    # Execute the search query
    c.execute("SELECT * FROM table WHERE column LIKE ?", (query,))
    results = c.fetchall()

    # Close the connection
    conn.close()

    return results
```

> When using `sqlite3.connect()` as shown here in the code block. I have seen `database` and `database.db` passed here. If you getting the following error: `sqlite3.DatabaseError: file is not a database` then it will likely be resolved by dropping the `.db` file extension.
{: .prompt-info }

This search function connects to a SQLite database, executes a search query that matches the query string against a column in a table, and returns the search results. You can customize the search query to meet the specific needs of your database and search requirements.

You will need to make sure that you have a database file named `database` (or `database.db`) in the same directory as your Flask app, and that you have a table with a column that you want to search. You will also need to install the `sqlite3` library in order to use it in your Flask app because now we are importing sqlite3 at the beggining of our code.

# Creating the Database

For this Flask app we will us the following database as an example:

```sql
CREATE TABLE users (
    id INTEGER PRIMARY KEY,
    username TEXT,
    email TEXT
);

INSERT INTO users (username, email) VALUES
    ("user1", "user1@example.com"),
    ("user2", "user2@example.com"),
    ("user3", "user3@example.com");
```

This database file contains a table named `users` with three columns: `id`, `username`, and `email`. It also includes three rows of data for three sample users.

This database file contains only the `CREATE TABLE` and `INSERT INTO` statements, which define the structure of the `users` table and insert data into the table. However, *these statements do not create the actual database file*.

To create the database file, you will need to execute these statements using a SQLite client or by using the `execute()` method in your Flask app. For this example we will use the `execute()` method in our Flask app to create the database file and populate the users table:

```python
import sqlite3

def create_database():
    # Connect to the database
    conn = sqlite3.connect('database.db')
    c = conn.cursor()

    # Create the users table
    c.execute(
        '''CREATE TABLE users (
            id INTEGER PRIMARY KEY,
            username TEXT,
            email TEXT
        )'''
    )

    # Insert data into the users table
    c.execute(
        '''INSERT INTO users (username, email) VALUES
            ("user1", "user1@example.com"),
            ("user2", "user2@example.com"),
            ("user3", "user3@example.com")'''
    )

    # Save changes and close the connection
    conn.commit()
    conn.close()

create_database()
```

This code creates the database file for us and executes the `CREATE TABLE` and `INSERT INTO` statements to create the `users` table and insert data into the table.

Place this block after `app = Flask(__name__)` and before our recently created `def perform_search(query):` lines.

# Conclusion

To recap, we have created a `FlaskApp.py`. Added a search function to use with a database. Added some sample data with the `execute()` method to create that database. Added our search and results templates with working routes to be used in our URLs and serve up the pages.

Our Flask app directory will look like the following:

```bash
$ tree
.
└── FlaskAppProjectFolder
    ├── FlaskApp.py
    ├── database
    └── templates
        ├── search.html
        └── results.html
```

### Completed FlaskApp.py
```python
from flask import Flask, request, render_template
import sqlite3

app = Flask(__name__)

def create_database():
    # Connect to the database
    conn = sqlite3.connect('database')
    c = conn.cursor()

    # Create the users table
    c.execute(
        '''CREATE TABLE users (
            id INTEGER PRIMARY KEY,
            username TEXT,
            email TEXT
        )'''
    )

    # Insert data into the users table
    c.execute(
        '''INSERT INTO users (username, email) VALUES
            ("user1", "user1@example.com"),
            ("user2", "user2@example.com"),
            ("user3", "user3@example.com")'''
    )

    # Save changes and close the connection
    conn.commit()
    conn.close()

create_database()

def perform_search(query):
    # Connect to the database
    conn = sqlite3.connect('database')
    c = conn.cursor()

    # Execute the search query
    c.execute("SELECT * FROM users WHERE username LIKE ?", (query,))
    results = c.fetchall()

    # Close the connection
    conn.close()

    return results

@app.route('/')
def search_page():
    return render_template('search.html')

@app.route('/search', methods=['POST'])
def search():
    query = request.form['query']
    results = perform_search(query)
    return render_template('results.html', query=query, results=results)

if __name__ == '__main__':
    app.run()
```

### Completed Search HTML Template
```html
<!DOCTYPE html>
<html>
<head>
  <title>Search Page</title>
  <style>
    form {
      width: 500px;
      margin: 0 auto;
      text-align: center;
      background-color: #eee;
      padding: 20px;
      border-radius: 5px;
    }
    input[type="text"] {
      width: 70%;
      padding: 12px 20px;
      margin: 8px 0;
      box-sizing: border-box;
      border: 2px solid #ccc;
      border-radius: 4px;
    }
    button[type="submit"] {
      width: 20%;
      background-color: #4CAF50;
      color: white;
      padding: 14px 20px;
      margin: 8px 0;
      border: none;
      border-radius: 4px;
      cursor: pointer;
    }
    button[type="submit"]:hover {
      background-color: #45a049;
    }
  </style>
</head>
<body>
  <h1>Search</h1>
  <form action="/search" method="POST">
    <input type="text" name="query" placeholder="Enter search query">
    <button type="submit">Search</button>
  </form>
</body>
</html>
```

## Completed Results HTML Template
```html
<!DOCTYPE html>
<html>
<head>
  <title>Search Results</title>
</head>
<body>
  <h1>Results for: {{ query }}</h1>
  <ul>
    {% for result in results %}
      <li>{{ result }}</li>
    {% endfor %}
  </ul>
</body>
</html>
```

### It Works
Searching for a user

![Search For User](/project-assets/FlaskAppWithSimpleSearch/search-for-user.png)

Getting results back from database for user1

![Results For User](/project-assets/FlaskAppWithSimpleSearch/results-for-user.png)