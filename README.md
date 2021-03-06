# Engineer Data In Google Cloud

This exercise is part of Data Engineering and Machine Learning Fundamental training provided by Google. ‘Engineer Data in Google Cloud Challenge Lab’ is a non-guided lab under the quest ‘Engineer Data in Google Cloud’ from Google Skills Boost. 

## Scenario

You have started a new role as a Data Engineer for TaxiCab Inc. You are expected to import some historical data to a working BigQuery dataset, and build a basic model that predicts fares based on information available when a new ride starts. Leadership is interested in building an app and estimating for users how much a ride will cost. The source data will be provided in your project.

## Challenge

As soon as you sit down at your desk and open your new laptop you receive your first assignment: build a basic BQML fare prediction model for leadership. Perform the following tasks to import and clean the data, then build the model and perform batch predictions with new data so that leadership can review model performance and make a go/no-go decision on deploying the app functionality.

Lab: https://www.cloudskillsboost.google/focuses/12379?parent=catalog

## Task 1: Clean your training 

You've already completed the first step, and have created a dataset taxirides and imported the historical data to table, historical_taxi_rides_raw. This is data prior for rides to 2015.
You may need to wait 1-3 minutes for the data to be fully populated in your project.
To complete this task you will need to:
* Clean the data in <span style="color:red">historical_taxi_rides_raw</span> and make a copy to taxi_training_data_328 in the same dataset. You can use BigQuery, DataPrep, DataFlow, etc. to create this table and clean the data. Make sure your target column is called fare_amount_818 .
Some helpful hints:
* You can see the source dataset in the BQ UI - familiarize yourself with the source schema first.
* As a hint for the data avilable at prediction time, familiarize yourself with the table taxirides.report_prediction_data which shows the format data will arrive at prediction time.
Data Cleaning Tasks:
* Ensure trip_distance is greater than 1 .
* Remove rows were fare_amount is very small (less than $2.0 for example).
* Ensure that the latitudes and longitudes are reasonable for the use case.
* Ensure passenger_count is greater than 1 .
* Be sure to add tolls_amount and fare_amount to fare_amount_818 as the target variable since total_amount includes tips.
* Because the source dataset is large (>1 Billion rows), sample the dataset to less than 1 Million rows.
* Only copy fields that will be used in your model (report_prediction_data is a good guide).

```
CREATE OR REPLACE TABLE taxirides.taxi_training_data_328 AS
SELECT
  (tolls_amount + fare_amount) AS fare_amount_818,
  pickup_datetime,
  pickup_longitude AS pickuplon,
  pickup_latitude AS pickuplat,
  dropoff_longitude AS dropofflon,
  dropoff_latitude AS dropofflat,
  passenger_count AS passengers,
FROM
  taxirides.historical_taxi_rides_raw
WHERE
RAND() < 0.001
AND trip_distance > 0
AND fare_amount >= 2
AND pickup_longitude > -78
AND pickup_longitude < -70
AND dropoff_longitude > -78
AND dropoff_longitude < -70
AND pickup_latitude > 37
AND pickup_latitude < 45
AND dropoff_latitude > 37
AND dropoff_latitude < 45
AND passenger_count > 0
```

## Task 2: Create BQML Model
Based on the data you have in taxirides.taxi_training_data_328 , build a BQML model that predicts fare_amount_818 . Call the model taxirides.fare_model_477 . Your model will need an RMSE of 10 or less to complete the task.
Some helpful hints:
* You can encapsulate any additional data transformationsn in a TRANSFORM() clause
* Keep in mind, only features in the TRANSFORM() clause will be passed to the model. You can use a * EXCEPT(feature_to_leave_out) to pass some or all of the features without explicitly calling them
* ST_distance() and ST_GeogPoint() GIS functions in BigQuery can be used to easily calculate euclidean distance (i.e. how far pickup to dropoff did the taxi travel):

```
CREATE OR REPLACE MODEL taxirides.fare_model_477
TRANSFORM (
* EXCEPT(pickup_datetime),
ST_Distance(ST_GeogPoint(pickuplon, pickuplat), 
ST_GeogPoint(dropofflon, dropofflat)) AS euclidean,
CAST(EXTRACT(DAYOFWEEK FROM pickup_datetime) AS STRING) AS dayofweek, 
CAST(EXTRACT(HOUR FROM pickup_datetime) AS STRING) AS hourofday
)
OPTIONS(input_label_cols=['fare_amount_818'], model_type='linear_reg') 
AS
SELECT * FROM `qwiklabs-gcp-03-24bc07c21bd0.taxirides.taxi_training_data_328`
```

## Task 3: Perform a batch prediction on new data
Leadership is curious to see how well your model performs over new data, in this case, all of the data they've collected in 2015. This data is in taxirides.report_prediction_data. Only values known at prediction time are included in the table.
Use ML.PREDICT and your model to predict Fare amount and store your results in a table called 2015_fare_amount_predictions

```
CREATE OR REPLACE TABLE taxirides.2015_fare_amount_predictions
  AS
SELECT * FROM ML.PREDICT
(MODEL taxirides.fare_model_477,
(SELECT * FROM taxirides.report_prediction_data))
```
