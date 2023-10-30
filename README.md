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
