---
title: "jobTracker - Database Setup"
excerpt_separator: "<!--more-->"
categories: 
  - Project
---

1. TOC
{:toc style="font-size: 0.75em;"}

# Intro

This post is to walkthrough the Postgresql database setup to use with [jobTracker](https://github.com/Ferrallv/jobTracker). Depending on your operating system the installation process will vary. On macOS I used [Postgres.app](https://postgresapp.com/). Other methods can be found [here](https://www.postgresql.org/download/)! Here we will walk through installing Postgres.app for macOS and creating the database.

## Postgres.app Installation

Navigate to [Postgres.app](https://postgresapp.com/) and select the 'Downloads' tab. Press the download button under the option 'Latest Release'

Once downloaded, you can open the `.dmg` file which will direct you to drag and drop the application into your application folder.

{:refdef: style="text-align: center;"}
![drag_and_drop](../../assets/images/jobTracker_Database_Setup_1.gif)
{: refdef}

You can then open Postgres by finding it in your applications folder or search for it using (⌘+SPACE). Once initialized we will want to use the command line tools. To make this possible go to your Terminal and enter:

```
sudo mkdir -p /etc/paths.d && echo /Applications/Postgres.app/Contents/Versions/latest/bin | sudo tee /etc/paths.d/postgresapp
```

Exit your terminal, then re-open and type
`psql`

If it looks like 

{:refdef: style="text-align: center;"}
![psql](../../assets/images/jobTracker_Database_Setup_2.png)
{: refdef}

then it all went swimmingly! Now we can create our database.

## The Database

NOTE: You can find the collection of the following commands [here](https://github.com/Ferrallv/jobTracker/blob/master/db_creation_commands.md)

In the Terminal after entering the Postgres CLI our first step is to create the database. We can do this by entering

### DB

```
CREATE DATABASE jobTracker;
```

And to connect to the database enter `\c jobtracker;`. Once connected we would like to create a user that the jobTracker app will use to connect to the database. We do this with the following commands:

### USER

```
CREATE USER <username> WITH PASSWORD '<password>';

GRANT ALL PRIVILEGES ON DATABASE jobtracker TO <username>;

GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO <username>;
```

Be sure to replace the '<>' items with a username and password you would like the app to use.

These commands create a user and give it the necessary access rights to create, read, update, and delete records from the database.

Before we create the tables we will want to create a custom domain to hold emails. This [discussion](https://dba.stackexchange.com/questions/68266/what-is-the-best-way-to-store-an-email-address-in-postgresql) gives us the tools, links,  and reasoning behind the implementation.

### TABLES

```
CREATE EXTENSION citext;

CREATE DOMAIN email AS citext
  CHECK ( value ~ '^[a-zA-Z0-9.!#$%&''*+/=?^_`{|}~-]+@[a-zA-Z0-9](?:[a-zA-Z0-9-]{0,61}[a-zA-Z0-9])?(?:\.[a-zA-Z0-9](?:[a-zA-Z0-9-]{0,61}[a-zA-Z0-9])?)*$' );
```

Now we can create the tables. 

```
CREATE TABLE contacts (
	id 			serial PRIMARY KEY,
	name 		varchar(45) NOT NULL, 
	position 	varchar(45),
	number 		varchar(15),
	email 		email,
	company 	varchar(45) NOT NULL,
	note 		text
);
```
After entering the command above in the CLI you should receive a response `CREATE TABLE`. You can check the table with `\d contacts`. Continue creating the following tables and you will have the database set up to function with the jobTracker app!

```
CREATE TABLE application (
	id 			serial 	PRIMARY KEY,
	job_title 	varchar(45) NOT NULL,
	description text,
	url 		text,
	company 	varchar(45) NOT NULL,
	resume 		bytea,
	cvr_letter 	bytea,
	app_date 	bigint,
	offer 		bigint,
	rejected 	bigint,
	declined 	bigint
);

CREATE TABLE interview (
	id 		serial PRIMARY KEY,
	date 	bigint NOT NULL,
	method 	varchar(10) NOT NULL,
	job_id 	int REFERENCES application(id)
);
```

For full descriptions of the data types check out [Postgres Docs](https://www.postgresql.org/docs/current/datatype.html). I will also go over what is used here to give the reasoning behind the choices.

- id: Giving each record a unique id is necessary if we want to update or delete specific entries. The SERIAL type is really just an integer type with an auto-incrementing property. 

- varchar(n): Each record is a string of length to maximum n. The number 45 was chosen arbitrarily; I felt that those were enough characters for fields such as company names and job titles. If an entered string is longer than the limit, it will be truncated to the specified limit.

- text: used to store long strings with no upper limit, such as notes or descriptions.

- bytea: used to store byte strings. This is useful for storing the data of images or `.pdf` files like resumes or cover letters. 

- bigint: a large range integer that requires 8 bytes of storage. The app is storing Unix timestamps to guarentee consistent timestamp formatting between jobTracker and the Postgresql database. There were some headaches with the consistency of formatting and parsing the Postrgres TIMESTAMP datatype and Golang time library. The storage usage is equivalent as a TIMESTAMP with or without timezone is also 8 bytes!

- PRIMARY KEY: a constraint put on a column to indicate that each record can be uniquely identified by these values.

- NOT NULL: a constraint put on a column to indicate that every record must contain a valid entry for this field.



