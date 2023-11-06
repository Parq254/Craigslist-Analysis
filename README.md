# Craigslist-Analysis
Before diving into the analysis we first load the necessary libraries;

```
import pandas as pd
import numpy as np
import plotly.express as px
import matplotlib.pyplot as plt
import plotly.graph_objects as go
import seaborn as sns
```
Next we load the data from the dataframe;
```
df = pd.read_csv('/kaggle/input/craigslist-vehicles/craigslist_vehicles.csv')
df.head()
```
Next we look at the available columns in the dataframe;
```
df.columns
```
Since we'll be using 'posting date' for the time series,we convert it to datetime data type;
```
df['posting_date'] = pd.to_datetime(df['posting_date'],  utc=True)
```
Next we check for columns having existing null values;
```
df.isnull().sum()
```
We will handle the missing values in the columns filling numerical ones with mean and categorical with mode;
```
def handle_missing_values(df):
    numerical_columns = ['year', 'odometer']
    df[numerical_columns] = df[numerical_columns].fillna(df[numerical_columns].mean())
    
    categorical_columns = ['manufacturer', 'model', 'condition', 'cylinders', 'fuel', 'title_status',
                           'transmission', 'drive', 'size', 'type', 'paint_color', 'posting_date']
    df[categorical_columns] = df[categorical_columns].apply(lambda x: x.fillna(x.mode().iloc[0]))
    return df
df = handle_missing_values(df)
```
Next we Aggregate data based on 'posting_date', 'region,' and 'type' of vehicle, to be able to do further analysis like temporal patterns, seasonal trends, and demand-supply dynamics.
```
def convert_to_tz_aware(posting_date):
    if not posting_date.tzinfo:
        return posting_date.replace(tzinfo=pytz.utc)
    else:
        return posting_date
df['posting_date'] = df['posting_date'].apply(convert_to_tz_aware)
df_agg = df.groupby(['region', 'type', 'posting_date']).size().reset_index(name='count')
df_agg = df_agg.sort_values(by='posting_date')
print(df_agg.head())
```
Next we dive into the visual aspect by creating a time series frequency graph;
```
df_freq = df_agg.groupby(pd.Grouper(key='posting_date', freq='D')).sum().reset_index()

fig_freq = go.Figure(data=go.Bar(
    x=df_freq['posting_date'],
    y=df_freq['count'],
    marker_color='royalblue',
    opacity=0.8
))

fig_freq.update_layout(
    title='Time Frequency Graph: Number of Vehicle Listings per Day',
    xaxis_title='Posting Date',
    yaxis_title='Number of Vehicle Listings',
    xaxis_tickangle=-45,
)
fig_freq.show()
```
Next we create an interactive time series chart;
```
fig = px.line(df_agg, x='posting_date', y='count', color='region', line_group='type',
              title='Number of Available Vehicles Over Time by Region and Vehicle Type',
              labels={'count': 'Number of Vehicles'})
fig.update_layout(
    xaxis_title='Posting Date',
    yaxis_title='Number of Vehicles',
    hovermode='x',
    showlegend=True,
)
fig.show()
```
# Conerting CSV to SQL

**Basic Algorithm Explanation**
The algorithm mainly needs only one thing - CSV dataset that will be put in functions below as an argument. Some of them returns SQL statements that are executed.

**get_table_name**(csv_file) <-- returns proper table name (or generate new one almost randomly) that is allowed by sqlite3 documentation.

**create_table**(df_dataset, table_name)<-- creates column names with corresponsing data types and returns SQL statement, for example CREATE TABLE "students" ("school" TEXT, ...);

**drop_table_if_exists**(table_name)<-- returns DROP TABLE IF EXISTS "students";

**insert_into_values**(df_dataset, table_name) <-- returns INSERT INTO "students" VALUES (?,?,?, ...);. Then question marks will be replaced by values in function executemany().

**convert_to_str(df_dataset)** <-- the function converts problematic pandas datatypes like datetime or timedelta to be interpreted by sqlite as strings.

**executemany**(df_dataset, table_name)<-- executes insert_into_values() function.

```
import sqlite3
import csv
import re
import pandas as pd # to load column names and their data types
import random, string # to generate random table name if necessary
```
```
csv_file = '/kaggle/input/craigslist-vehicles/craigslist_vehicles.csv'#Loading the dataset
```
```
df_from_csv = pd.read_csv(csv_file, delimiter=',')
df_from_csv.head()#Display head of the dataframe 
```
```
df_from_csv.dtypes.unique() #Display unique data types
```
//ggghh
```
def get_table_name(csv_file):
#Create a table name from CSV file name and convert it to be table name allowed by slite3 documentation.
    # when in CSV file name there are letters too
    regex = re.compile('[^a-z]')
    table_name = csv_file.split("/")[-1].split(".")[0]
    table_name = regex.sub('', table_name)
    # when in CSV file name there aren't any letters
    if table_name == '':
        for i in range(10):
            table_name += random.choice(string.ascii_lowercase)
    return table_name
generated_table_name = get_table_name(csv_file)
generated_table_name
```
Instead of entering each column name with correspondent data type in each SQL STATEMENTS, we define functions making these statements quickly.
```
def create_table(df_dataset, table_name):
#The function returns SQL statement "CREATE TABLE" with needed table name and its column names along with data types which these columns will store.

    cols_with_sql_types = []
    for col_name, col_type in df_dataset.dtypes.items():
        if col_type == "int64":
            cols_with_sql_types.append('"' + col_name + '"' + " " + 'INTEGER')
        elif col_type == "float64":
            cols_with_sql_types.append('"' + col_name + '"' + " " + 'REAL')
        else:
            cols_with_sql_types.append('"' + col_name + '"' + " " + 'TEXT')
    final = str(cols_with_sql_types).replace("'", "").replace(']', '').replace('[', '')
    return f'CREATE TABLE "{table_name}" ({final})'
```
```
def drop_table_if_exists(table_name):
#The function returns SQL statement "DROP TABLE IF EXISTS" with needed table name.
   
    return f'DROP TABLE IF EXISTS {table_name}'
```
# Create a Table
```
conn = sqlite3.connect(f'{generated_table_name}.sqlite')
cur = conn.cursor()
cur.execute(f"{drop_table_if_exists(generated_table_name)}")
cur.execute(f"{create_table(df_from_csv, generated_table_name)}")
```
```
def insert_into_values(df_dataset, table_name):
#The function returns SQL statement "INSERT INTO" with needed table name and values.
    
    numb_of_columns = len(df_dataset.columns)
    values = str(['?' for i in range(numb_of_columns)]).replace("'", "").replace(']', '').replace('[', '')
    return f'INSERT INTO "{table_name}" VALUES ({values})'
```
```
def convert_to_str(df_dataset):
#The function converts problematic dtypes to strings.
    
    for i in df_dataset.select_dtypes(include=['datetime', 'timedelta']):
        df_dataset[i] = df_dataset[i].astype(str)
    return df_dataset
```
```
def executemany(df_dataset, table_name):
    with sqlite3.connect(f'{table_name}.sqlite'):
        conn = sqlite3.connect(f'{table_name}.sqlite')
        cur = conn.cursor()
        values = convert_to_str(df_dataset).values.tolist()
        cur.executemany(f"{insert_into_values(df_dataset, table_name)}", values)
        conn.commit()
executemany(df_from_csv, generated_table_name)
```
```
df_from_sql = pd.read_sql_query(f'select * from {generated_table_name}', con = conn)
df_from_sql
```
# Conclusion 
The final outcome shows dataframe made from SQLITE3 database. As we can see, there are no differences between it and initial dataframe made from CSV.

