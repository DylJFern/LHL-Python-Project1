# Statistical Modelling with Python

## Project/Goals
The objective of this project was to combine and practice implementing what has been learned thus far. This included working with Python to access data through API calls, parsing through it to obtain necessary information, as well as cleaning and transforming the data and loading it into a database or storing it in various file and tabular formats. The structured data could then be used to perform exploratory data analysis involving statistics and visiualizations/models whereby the interpreted results would help in identifying trends and patterns within the data and be used to predict future outcomes through estimates or conclusions.

In this particular project, the goal was to perform exploratory data analysis to demostrate a relationship between the number of bikes in a particular location and the characteristics of the points of interest in that location.

## <br>Process
### Connecting to CityBikes API
Citybikes is an API that provides bike sharing data for applications, research and projects. We begin by first exploring the structure of the API, how to perform a query for the API and understand the data returned.

The request module will be used to call the APIs, it allows you to send HTTP requests using Python and returns a 'Response Object' with al the response data (content, encoding, status, etc).
```python
import requests 
```                                                                                                                    
We can use the *get()* method to send a GET request (a HTTP method that requests data from a specified source) to the specified url. In return we will have access to all available cities covered by CityBikes API.
```python
# "city_bike_networks" is the URL of the API endpoint
response_networks = requests.get(city_bike_networks)
```
However, our focus was on a specific city, in this case, Montréal, QC, CA: Bixi was selected. When used together, this specific network_id and API endpoint would return all available bike stations in this city. 

<br>When making API calls, errors can occur (also known as exceptions) and when this happens Python will normally stop and generate an error message. In situations such as these a try-except can be implemented where the "try" block lets you test a block of code for erros and "except" lets you handle the error. For the purposes of this project only two specific exceptions were used, they may be commonly seen when making HTTP requests. 
```python
try:
  response_MTL = requests.get(city_bike_MTL)
  # If the request is successful
  print("Request successful. Status code:", response_MTL.status_code)

# HTTP error for a status codes like 404 (page not found) or 429 (too many requests in given time)
except requests.exceptions.HTTPError as http_err:
  print("HTTP error occurred. Error:", str(http_err))

# If the server the request is being made to takes too long to respond
except requests.exceptions.Timeout as timeout_err:
  print("Request timed out. Error:", str(timeout_err))

# General exception 
except Exception as exc:
  print("Request failed. Error:", str(exc))
```
Although it is not necessary for this particular call, it could be considered good practice implementing them.

<br>After the request completed, we could store it as follows,
```python
# Load data into a JSON string format
city_bike_data = response_MTL.json()
```
where *json()* is a standardized format commonly used to transfer data as text that can be sent over a network. 

<br>It consists of name/value pairs (like a Python dictionary). We can access elements within this string using keys and indices to navigate. And by automating this process with a for-loop and an *append()* method we are able to create a DataFrame containing the parsed results.

![Capture](https://github.com/DylJFern/LHL-Python-Project1/assets/128000630/416ddd52-07cb-43d1-b596-e82afe362274)

This DataFrame contains a latitute and longitude for each station which can contain free_bikes (bikes available for rent), empty_slots (bikes unavailable for rent), ebikes (total number of unavailable/available bikes).

### <br> Connecting to Foursquare and Yelp APIs
Foursquare and Yelp are a type of geolocation sharing application. They contain information about places like their latitude and longitude, ratings, popularity, number of reviews, and price.

In this step, we perform a similar process to the CityBikes API. However, the difference is that we are going to find only the places within a 1000m distance of the CityBike stations based on their latitute and longitude. To perform as accurate of a comparison as possible between [Foursquare (FSQ) places](https://location.foursquare.com/places/docs/categories) and [Yelp places](https://docs.developer.yelp.com/docs/resources-categories) the list of points of interest (categories) for each platform was exhaustively examined to determine which ones would closely match.        

<br>To name a few, similar categories listed on both platforms were: 
  - restaurants (FSQ id: 13065)
  - arts and entertainment (FSQ id: 10000); arts (Yelp)
  - landmarks and outdoors (FSQ id: 160000); landmarks and historical buildings (Yelp)    

<br> Limited the number of API calls to 350 for both FSQ and Yelp, this was due to the time-consuming process associated with making each API calls and due Yelp's daily limiter of 500  API calls a day. This also provides a fair comparison such that both platforms recevive the same amount of calls.

<br> Selected similar categories for both platforms, the ones that will be mainly relevant for the analysis are:
  - distance from place to bike station (FSQ 'integer'; Yelp 'float')
  - rating (FSQ 'float', scale: 0.0 to 10.0; Yelp 'float', scale 0.0 to 5.0)
  - price (FSQ 'float', scale: 0.0 to 4.0; Yelp 'object', scale: $ to $$$$)

<br>As mentioned, the API calls for both Foursquare and Yelp follow a similar process as CityBikes API. However, the location of the available places within a 1000m distance of the latitute and longitude of the bike stations. For starters, Foursquare and Yelp have different procedures for passing in parameters and receiving a response, the url will be a combination of the number of query parameters and features selected. For details on how to construct a Foursquare or Yelp API, one should refer to the online documentation.
The nested for-loop structure is more or less the same for both platforms (they just have varying naming conventions and JSON path structures).
```python
...
# Iterate over the rows of all_stations_df (CityBikes DataFrame) 350 times
for _, df_row in all_stations_df.iloc[:350].iterrows():
  # Function call to find the places in Foursquare based on CityBikes latitude and longitude within a 1000m radius
  fsq_res = fsq_place_data(latitude = df_row['latitude'], longitude = df_row['longitude'], radius = 1000)
  # Convert API call into human-readable format
  fsq_json = fsq_res.json()
  try:
    ...
      fsq_place_details = {
        ...
      }
      fsq_data.append(fsq_place_details)

  except requests.exceptions.HTTPError as http_err:
    # HTTP error for status code 429 (too many requests), we delay the program's execution
    if fsq_res.status_code == 429:
      time.sleep(30)
  ...
fsq_place_df = pd.DataFrame(fsq_data)
```
In addition to the nested for-loop structure, the API calls for Foursquare and Yelp contain a 'time.sleep(30)' to delay the program's execution. This is useful mainly as a countermeasure against Yelp's rate limiter which is 500 API calls per day, but also by the number of queries per second (QPS). Essentially, whenever the program reaches the set limitation, the exception is thrown and then after the delay, the program resumes its normaly behaviour iterating through the for-loops. In CityBikes, it did contain an HTTPError which code '429' falls under; however, in that case we were not specifically looking for it such that we could prevent it from occurring.

<br>Once the 350 API calls completed, we could obtain the following results for Foursquare and Yelp, respectively.

![Capture1](https://github.com/DylJFern/LHL-Python-Project1/assets/128000630/a0e74364-1cb3-47a6-b6f3-6736acd5c038)

![Capture11](https://github.com/DylJFern/LHL-Python-Project1/assets/128000630/4d4e7d4f-0562-4146-b88e-6129fa4235f2)

Comparing the results, we notice Foursquare returns 3488 entries and Yelp returns 6497 entries. As previously mentioned, the relevant data we are concerned with is 'distance', 'rating', 'price'. Using the method *info()* we observe that Foursquare has 1768 null rows ('fsq_rating' = 600, 'fsq_price' = 1120) and Yelp has 1730 null rows ('yelp_price' = 1730). However, when comparing the data overall it is determined that Yelp provided more results for the same amount of API calls.

### <br> Joining Data
Now that we have all of our data (CityBikes, Foursquare and Yelp) we can combine it to perform exploratory data analysis (EDA) using statistics and visualizations. Using the following code, we can get a quick summary of null values for the merged DataFrame.
```python
print(f"\n{all_data_df.isna().sum()}")
```

![Capture12](https://github.com/DylJFern/LHL-Python-Project1/assets/128000630/dc4c4eaa-d791-4c94-96c2-bed0e5d9e0f4)

<br>Before performing the EDA process we should work on eliminating irrelevant data, null rows, and work standardizing similar columns. The following are the procedures used to clean the data in this case.
```python
# Remove the irrelevant columns (no relation between the two)
data_df_irrelevant_col = all_data_df.drop(columns = ['fsq_popularity', 'yelp_review_count'])

# Remove rows containing null values (default is row-wise or axis = 0)
data_df_non_null = data_df_irrelevant_col.dropna(subset = ['fsq_rating', 'fsq_price', 'yelp_price'])

clean_data_df = data_df_non_null.copy()

# Standardize the 'rating' data by scaling "yelp_rating"
clean_data_df['yelp_rating'] = (clean_data_df['yelp_rating'])*2

# Standardize the 'price' data by mapping "yelp_rating" to numbers
yelp_price_map = {
  '$': 1,
  '$$': 2,
  '$$$': 3,
  '$$$$': 4
}
clean_data_df['yelp_price'] = clean_data_df['yelp_price'].map(yelp_price_map)

# Standardize by convert data to more appropriate types
clean_data_df['fsq_price'] = clean_data_df['fsq_price'].astype('int64')
clean_data_df['fsq_distance (m)'] = clean_data_df['fsq_distance (m)'].astype('float64')
```

<br> The data obtained previously can also be used to create a database. In this case, the assumption is made is to use the raw data exported to CSV files. The process of creating a SQLite database may look like the following.
```python
# Define a function that accepts the path to the database
def create_connection(path):
  connection = None
  try:
    # If the database exists at the specified location, then the connection is established
    # If the database does not exist at the specified location, it is created and the connection is established
    connection = sqlite3.connect(path)
    print("Connection to 'city_bike_poi' database successful")
  # Handle cases where 'connect()' fails to establish a connection
  except Error as e:
    print(f"The error '{e}' occurred")     
  return connection
```
```python
connection = create_connection("../data/city_bike_poi.sqlite")

# Write the DataFrames to the database
fsq_df.to_sql('foursquare_poi', connection, if_exists = 'replace', index = False)
yelp_df.to_sql('yelp_poi', connection, if_exists = 'replace', index = False)
city_bikes_df.to_sql('montreal_bike_stations', connection, if_exists = 'replace', index = False)

# Close the database connection
connection.close()
```

## <br>Results
In this section we will primarily looking relationships between all the variables. A heatmap can be used to observe any correlations between the variables, the colour will vary depending on the correlation of the relationship.

![Capture13](https://github.com/DylJFern/LHL-Python-Project1/assets/128000630/3f67abb5-79c0-4003-a2cc-b4524769b6f9)

The heatmap shows us that there is a moderate negative correlation between 'fsq_rating' and 'fsq_distance (m)' and a weak negative correlation between 'yelp_rating' and 'yelp_distance(m)'. For the Foursquare data, this may suggest that as the distance from the station increases, the rating tends to decrease (indicating that places closer to the station tend to have higher ratings).

It also shows a strong positive correlation between 'fsq_rating' and 'total_bikes' and a moderate positive correlation between 'yelp_rating' and 'total_bikes'. For both platforms, this could indicate that stations with more bikes are highly populated or popular areas, where people tend to frequent. These popular areas may have higher-rated establishments thus contributing to the correlation.

In general, there is a moderate positive correlation between the rating and available/unavailable bikes, but there is a weak correlation between the rating and available/unavailable ebikes.

Furthermore, there is a moderate negative correlation between 'fsq_distance (m)' and 'total_bikes', and a weak negative correlation between 'yelp_distaince (m)' and 'total_bikes'. For Foursquare, the data may suggest that the total number of bikes at a station decreases as the distance from the place and the station increases.

<br>A multivariate regression model can help us find a relationship between multiple variables or features.

![Capture14](https://github.com/DylJFern/LHL-Python-Project1/assets/128000630/9876ec31-d298-4ddd-ad17-e8719ba96bb1)

The r-squared shows the goodness of fit of the model, for this specific model output it means that 50.6% of the varability in the total number of bikes at a station can be explained. The adjusted r-squared in this case is 50% which means it can explain about half of the varability, considering the trade-off between model complexity (number of predictor variables) and goodness of fit. The null hypothesis is that there is no relationship between the total number of bikes and the places, for this model all the predictor variables have p-values less than 0.05 which means that the independent variables are statistically significant in predicting the total number of bikes (the dependent variable).

## <br>Challenges 
  - The time constraint affecting the ability to perform in-depth analysis.
  - The daily 500 API calls set by Yelp requires collecting data in advance.
  - Uncertainty in the approach to take when conducting EDA due not general guidelines.

## <br>Future Considerations
  - Performing a more in-depth EDA process, exploring different types of visualizations (not already considered) and different plots that could been made examining different relationships.
  - Perform API calls over a time period to gather more data, which may also help showcase any new trends over time.
  - Examine a highly populated city like Toronto or New York and see how the data can vary between lesser populated ones.
  - Developing models containing additional variables that can be used for comparison (if any), this may even allow us to conduct feature selection using a p-value elimination approach.
