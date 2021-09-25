# London Bicycle Hires Data Analysis
Analysis of BigQuery Public Dataset on London Biclycle Hires using BigQuery/MySQL and Python. 

Dataset contains the number of hires of London's Santander Cycle Hire Scheme from 2011 to present. Data includes start and stop timestamps, station names and ride duration.


```python
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
import seaborn as sns
```


```python
# Set up feedback system
from learntools.core import binder
binder.bind(globals())
from learntools.sql_advanced.ex2 import *
```


```python
from google.cloud import bigquery

# Create a "Client" object
client = bigquery.Client()

# Construct a reference to the "london_bicycle" dataset
dataset_ref = client.dataset("london_bicycles", project="bigquery-public-data")

# API request - fetch the dataset
dataset = client.get_dataset(dataset_ref)

# Construct a reference to the "cycle_hire" and "cycle_stations" tables
cycle_hire_table_ref = dataset_ref.table("cycle_hire")
cycle_stations_table_ref = dataset_ref.table("cycle_stations")

# API request - fetch the table
cycle_hire_table = client.get_table(cycle_hire_table_ref)
cycle_stations_table = client.get_table(cycle_stations_table_ref)
```


```python
# Preview the first five lines of cycle_hire_table
client.list_rows(cycle_hire_table, max_results=5).to_dataframe()
```


```python
# Preview the first five lines of cycle_stations_table
client.list_rows(cycle_stations_table, max_results=5).to_dataframe()
```

# Extracting Tabular Data Through BigQuery/SQL 
## Trip Duration


```python
# Query to count the trip duration for the top 100 trips 
bike_trips_query = """
                    SELECT
                        bike_id,
                        trip_duration_minutes,
                        start_station_name,
                        end_station_name,
                        trip_year
                    FROM(
                        SELECT 
                        bike_id,
                        start_station_name,
                        end_station_name,
                        -- Calculating the trip duration in minutes -- 
                        DATETIME_DIFF(end_date,start_date,minute) AS trip_duration_minutes,
                        -- Identifying the trip year--
                        EXTRACT(YEAR FROM start_date) as trip_year
                        FROM `bigquery-public-data.london_bicycles.cycle_hire`
                        -- Between the years of 2016 and 2017 -- 
                        WHERE EXTRACT(YEAR FROM start_date) BETWEEN 2015 AND 2017
                        LIMIT 100 
                    ) AS t
                    ORDER BY trip_duration_minutes DESC
                  """

# Run the query, and return a pandas DataFrame
trip_duration_result = client.query(bike_trips_query).result().to_dataframe()
trip_duration_result.head()
```

We now have a table of the 100 longest bike trips along with their duration, origin, destination between 2015 and 2017.
## Daily Number of Trips


```python
# Query to count the daily number of trips in 2015 as well as the cumalative number of trips
bike_trips_query = """
                    -- Creating a sub query block trips_by_day with trip dates and number of trips --
                    WITH trips_by_day AS
                    (
                    SELECT 
                        DATE(start_date) AS trip_date,
                        COUNT(*) as num_trips
                    FROM `bigquery-public-data.london_bicycles.cycle_hire`
                    WHERE EXTRACT(YEAR FROM start_date) = 2015
                    GROUP BY trip_date
                    )

                    SELECT *,
                    SUM(num_trips) 
                    OVER (
                            ORDER BY trip_date
                            --Calculates the running total of number of trips--
                            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
                         ) AS cumulative_trips
                    FROM trips_by_day
                  """

# Run the query, and return a pandas DataFrame
number_of_trips_result = client.query(bike_trips_query).result().to_dataframe()
number_of_trips_result.head()
```

We now have a table with the number of trips on a given date in 2015 as well as the total number of tips up to that date.
## Bike Location on a Specific Day


```python
# Query to identify each bikes location on a certain day at various times (4th of July, 2015)
bike_trips_query = """
                    SELECT 
                        bike_id,
                        TIME(start_date) AS trip_time,
                        -- First station id tells us which station the bike started trips at that day -- 
                        FIRST_VALUE(start_station_id)
                            OVER (
                                PARTITION BY bike_id
                                ORDER BY start_date
                                ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
                             ) AS first_station_id,
                         -- Last station id tells us which station the bike ended trips at that day-- 
                         LAST_VALUE(end_station_id)
                            OVER (
                                PARTITION BY bike_id
                                ORDER BY start_date
                                ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
                            ) AS last_station_id,
                         start_station_id,
                         end_station_id
                    FROM `bigquery-public-data.london_bicycles.cycle_hire`
                    -- The 21st of June is the summer solstice so which is the longest day of the year --
                    WHERE DATE(start_date) = '2015-06-21' 
                  """

# Run the query, and return a pandas DataFrame
bike_location_result = client.query(bike_trips_query).result().to_dataframe()
bike_location_result.head(10)
```

We not have a table with information on each of the bikes movements throught a certain day,which could help us identify the location of a certain bike in a certain time interval.

## Bike Stations: Total Number of Trips and Average Trip Duration


```python
# Query to count the number of trips by start station
join_query = """
              -- Creating a sub query block c with start station, number of trips and duration--
             WITH c AS
             (
             SELECT 
                 start_station_id, 
                  -- Calculating avergae trip durarion from each station --
                 COUNT(*) as number_of_trips,
                 -- Calculating avergae trip durarion from each station --
                 ROUND(AVG(trip_duration_minutes)) AS avg_trip_duration_minutes
             FROM (
                 SELECT
                 start_station_id,
                 -- Calculating the trip duration in minutes -- 
                 DATETIME_DIFF(end_date,start_date,minute) AS trip_duration_minutes,
                 FROM `bigquery-public-data.london_bicycles.cycle_hire`
             ) 
             GROUP BY start_station_id
             )
             SELECT 
                 s.id as station_id, 
                 s.name,
                 c.number_of_trips,
                 c.avg_trip_duration_minutes 
             FROM `bigquery-public-data.london_bicycles.cycle_stations` AS s
             LEFT JOIN c
             ON s.id = c.start_station_id
             ORDER BY c.number_of_trips DESC
             """

# Run the query, and return a pandas DataFrame
station_trip_duration_result = client.query(join_query).result().to_dataframe()
station_trip_duration_result.head()
```

We now have a table with statistics on the number of trips and average duration for trips starting at that station
# Data Analysis

Now that we have completed extracting the information we can perform some basic analysis on the data.


```python
trip_duration_result.groupby('trip_year').agg(total_trip_duration = ('trip_duration_minutes',np.sum),avg_trip_duration = ('trip_duration_minutes',np.mean) )


```

<b>2016</b> was the year with the most ride time and the the longest aver trip duration.

## Bike Locations and Number of Trips


```python
#Identifying the top five stations where trips began
bike_location_result['end_station_id'].value_counts().head(5)
```


```python
#Identifying the top five stations where trips began
bike_location_result['start_station_id'].value_counts().head(5)
```


```python
# Analyzing the trip start times to identify how many trips begin at various times throughout the day.

# Isolating the time portion from bks_df['start_time']
start_time = data=bike_location_result['trip_time']

start_time_counts = start_time.value_counts()

#X-axis ticks will have to be set manually since the plot is difficult to read otherwise

#Creating a series that will be populated using a for loop 
time_series = []
for x in range(24):
    #Note {:02d} ensures there are leading zeros (01,02,etc.) which is consistent with time formatting
    time_series.append("{:02d}:00:00".format(x))

time_series.append("23:59:00")

fig_dims = (20, 4)
fig, ax = plt.subplots(figsize=fig_dims)

#The x-ticks are set to the time series just created
start_time_counts.plot(xticks=time_series)
plt.title("Ride Start Times throughout the Day", fontsize=20)
```


```python
from datetime import datetime
#Identifying number of bikes that were on a trip between noon and 2 pm 
len(bike_location_result[(bike_location_result['trip_time'] > datetime.strptime('16:00:00','%H:%M:%S').time()) & (bike_location_result['trip_time'] < datetime.strptime('17:00:00','%H:%M:%S').time())])
```


```python
#Identifying the top five bikes that made the most trips
bike_location_result['bike_id'].value_counts().head(5)
```

- The most trips began and ended at the <b>same five stations.</b> 
- The top five bike with the most trips made <b>15-17 trips</b> that day.
- The most trips began between the hours of <b>4 pm and 5 pm</b> (2800 in that hour alone). 

## Bike Stations


```python
#Identifying station with the longest averge trip duration.
station_trip_duration_result[station_trip_duration_result['avg_trip_duration_minutes'] == station_trip_duration_result['avg_trip_duration_minutes'].max()]
```


```python
#Identifying station with the shortest averge trip duration.
station_trip_duration_result[station_trip_duration_result['avg_trip_duration_minutes'] == station_trip_duration_result['avg_trip_duration_minutes'].min()]
```


```python
#Identifying station with the least number of trips
station_trip_duration_result[station_trip_duration_result['number_of_trips'] == station_trip_duration_result['number_of_trips'].min()]
```


```python
round(station_trip_duration_result['number_of_trips'].mean())
```

On average, 

- The most trips start from <b>Belgrove Street , King's Cross station.</b>
- The longest trips start from <b>Black Lion Gate, Kensington Gardens station.</b>
- The least trips start from <b>Here East South, Queen Elizabeth Olympic Park station.</b>
- We see that five stations have the shortest trip duration. The number of trips from these stations are also above average so these stations may be in a high traffic area where people are commuting short distances frequently.
