# MySQL and Streamlit Docker Setup

This project demonstrates how to create two Docker containers:
1. *MySQL container*: holds your database.
2. *Streamlit container*: displays data fetched from MySQL in JSON format.

## Project Structure


project/
├── docker-compose.yml
├── init.sql
└── streamlit_app/
    ├── Dockerfile
    ├── requirements.txt
    └── app.py


## Setup Instructions

### Step 1: Create a MySQL Container

Create a file named docker-compose.yml:

yaml
version: '3.8'

services:
  mysql:
    image: mysql:8.0
    container_name: mysql_container
    environment:
      MYSQL_ROOT_PASSWORD: rootpass
      MYSQL_DATABASE: testdb
      MYSQL_USER: testuser
      MYSQL_PASSWORD: testpass
    ports:
      - "3306:3306"
    volumes:
      - mysql_data:/var/lib/mysql
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql

  streamlit:
    build: ./streamlit_app
    container_name: streamlit_container
    ports:
      - "8501:8501"
    depends_on:
      - mysql

volumes:
  mysql_data:


### Step 2: Initialize Database

Create an init.sql file to set up your database schema and sample data:

sql
CREATE TABLE IF NOT EXISTS users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(50),
    email VARCHAR(100)
);

INSERT INTO users (name, email) VALUES 
    ('Anisha', 'a@example.com'),
    ('Asha', 'b@example.com');


### Step 3: Setup Streamlit App

Create a folder named streamlit_app and add the following files:

#### Dockerfile

dockerfile
FROM python:3.10

WORKDIR /app

COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["streamlit", "run", "app.py", "--server.port=8501", "--server.address=0.0.0.0"]


#### requirements.txt


streamlit
mysql-connector-python


#### app.py

python
import streamlit as st
import mysql.connector
import json
import time

# Add retries for database connection
max_retries = 5
retry_delay = 3  # seconds

for attempt in range(max_retries):
    try:
        # Connect to MySQL
        conn = mysql.connector.connect(
            host="mysql",
            user="testuser",
            password="testpass",
            database="testdb"
        )
        
        # Create cursor and fetch data
        cursor = conn.cursor(dictionary=True)
        cursor.execute("SELECT * FROM users")
        rows = cursor.fetchall()
        
        # Convert to JSON
        json_data = json.dumps(rows, indent=2)
        
        # Display in Streamlit
        st.title("Data from MySQL")
        st.json(json_data)
        
        # Close connections
        cursor.close()
        conn.close()
        
        break  # Connection successful, exit the retry loop
        
    except mysql.connector.Error as err:
        if attempt < max_retries - 1:
            st.warning(f"Database connection attempt {attempt+1} failed. Retrying in {retry_delay} seconds...")
            time.sleep(retry_delay)
        else:
            st.error(f"Failed to connect to the database after {max_retries} attempts. Error: {err}")


## Running the Application

1. Make sure all files are in place with the structure described above
2. Run the following command from your project root:
   bash
   docker-compose down -v  # Remove existing volumes if any
   docker-compose up --build
   
3. Wait for both containers to start (the Streamlit container may initially fail and retry connecting to MySQL)
4. Access your Streamlit app at http://localhost:8501
