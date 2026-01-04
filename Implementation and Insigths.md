# Implementation
## Set up 
1) Check IAM roles
In the Learner Lab environment I can use the LabRole IAM role for everything I need
<img width="1579" height="67" alt="image" src="https://github.com/user-attachments/assets/3f680ee8-bb59-4512-8d5f-63ec87a59a8d" />


2) Set up S3 buckets
Created 3 buckets for three different puruses:
* The `current-menu-ys39h3` holding the uploaded CSV data in Parquet format of the current menu items
* The `internal-sales-ys39h3` holding the uploaded CSV data in Parquet format of the historical sales data
* The `enriched-menu-ys39h3` endpoint of the lambda function with the detaled menu items

<img width="1241" height="307" alt="Képernyőkép 2026-01-03 214636" src="https://github.com/user-attachments/assets/104a06be-6d66-4b2b-9aa0-f9a2699ba33d" />

I added my cusman ID to the names to make sure the name of the buckets are unique in the system

3) Creating the Lambda function to enrich the current menu items from the spoonacular API
* I named the function `SpoonacularEnrichment`
* I set the Runtime to Python 3.14
* I set the Execution role to LabRole
* Writhe the Python code to the Code source part
```
import json
import requests
import boto3
import pandas as pd
import awswrangler as wr
import os

s3 = boto3.client('s3')
API_KEY = os.environ.get('SPOONACULAR_API_KEY')
BASE_URL = "https://api.spoonacular.com/recipes"

def lambda_handler(event, context):
    try:
        source_bucket = event['Records'][0]['s3']['bucket']['name']
        source_key = event['Records'][0]['s3']['object']['key']
    except (KeyError, IndexError):
        return {'statusCode': 400, 'body': 'No S3 record found in event.'}

    df = wr.s3.read_csv(path=f"s3://{source_bucket}/{source_key}")
    enriched_data = []

    for index, row in df.iterrows():
        dish_name = row['dish_name']
        
        # Initialize enriched fields with default "Not Found" values
        recipe_data = {
            "calories": None, "fat": None, "carbohydrates": None, "protein": None,
            "healthScore": None, "pricePerServing": None, "vegetarian": None,
            "vegan": None, "glutenFree": None, "dairyFree": None, "diets": "",
            "cuisine": "Unknown"
        }
        
        # 1. Consolidated Search
        search_params = {"query": dish_name, "addRecipeNutrition": "true", "addRecipeInformation": "true", "number": 1, "apiKey": API_KEY}
        search_resp = requests.get(f"{BASE_URL}/complexSearch", params=search_params)
        
        if search_resp.status_code == 200:
            results = search_resp.json().get('results')
            if results:
                recipe = results[0]
                nutrients_list = recipe.get('nutrition', {}).get('nutrients', [])
                nutrients = {n['name']: n['amount'] for n in nutrients_list}
                
                recipe_data.update({
                    "calories": nutrients.get("Calories"),
                    "fat": nutrients.get("Fat"),
                    "carbohydrates": nutrients.get("Carbohydrates"),
                    "protein": nutrients.get("Protein"),
                    "healthScore": recipe.get("healthScore"),
                    "pricePerServing": recipe.get("pricePerServing"),
                    "vegetarian": recipe.get("vegetarian"),
                    "vegan": recipe.get("vegan"),
                    "glutenFree": recipe.get("glutenFree"),
                    "dairyFree": recipe.get("dairyFree"),
                    "diets": ", ".join(recipe.get("diets", []))
                })

        cuisine_params = {"apiKey": API_KEY, "title": dish_name}
        cuisine_resp = requests.post(f"{BASE_URL}/cuisine", params={"apiKey": API_KEY}, data={"title": dish_name})
        if cuisine_resp.status_code == 200:
            recipe_data["cuisine"] = cuisine_resp.json().get("cuisine", "Unknown")

        enriched_item = {
            "dish_id": row.get('dish_id'),
            "dish_name": dish_name,
            "course": row.get('course'),
            **recipe_data 
        }
        enriched_data.append(enriched_item)
    if enriched_data:
        output_df = pd.DataFrame(enriched_data)
        output_filename = source_key.split('/')[-1].replace('.csv', '.parquet')
        output_path = f"s3://enriched-menu-ys39h3/{output_filename}"
        
        wr.s3.to_parquet(df=output_df, path=output_path)

    return {'statusCode': 200, 'body': f"Enriched {len(enriched_data)} items."}
```
* I added a layer to be able to deploy my code

<img width="1860" height="184" alt="Képernyőkép 2026-01-03 224025" src="https://github.com/user-attachments/assets/0fe12e77-a03c-4599-bcd5-840b512dcc7a" />

* I added my API key as an environment variable

<img width="1509" height="239" alt="image" src="https://github.com/user-attachments/assets/9cdb4f6f-8344-4452-8b88-5c9bc5da21eb" />

* I edited the General Configuaration to give more time out

<img width="1544" height="241" alt="image" src="https://github.com/user-attachments/assets/719b6a2d-d1e6-4199-b5c8-2c6ec20f3ca2" />

* I added a triger to update every time the current-menu gets a new menu 

<img width="1540" height="484" alt="image" src="https://github.com/user-attachments/assets/964109e8-2f28-4078-8f73-cc2c557763fd" />

4) Upload the Current Menu data to the `current-menu-ys39h3` bucket

<img width="1548" height="415" alt="image" src="https://github.com/user-attachments/assets/c3299222-aa80-49d9-b0cf-4c8b78ce68b5" />

5) The enriched data automaticly loades to the `enriched-menu-ys39h3` bucket

<img width="1551" height="439" alt="image" src="https://github.com/user-attachments/assets/7b15d0a2-2cd8-431b-a45a-63a91fc0b516" />

* I checked the format so I can see that everything is fine 

<img width="1487" height="551" alt="image" src="https://github.com/user-attachments/assets/8b98d7b1-cb3a-4fbd-9ed5-9730a2b728ae" />

6) Upload the sales historical data to the `internal-sales-ys39h3` bucket

<img width="1889" height="411" alt="image" src="https://github.com/user-attachments/assets/ec8eb8bb-51cd-411d-9bdb-e3da9639ad6b" />

7) Use AWS Glue crawler to join all the data 
* I created a crawler for the `internal-sales-ys39h3` bucket
* I set a new `restaurant-data table` as the output place

<img width="1236" height="662" alt="image" src="https://github.com/user-attachments/assets/ef0a6e6d-b7ed-4002-9718-4c1e62bea7ef" />

* And a crawler for the `enriched-menu-ys39h3` bucket

<img width="1239" height="666" alt="image" src="https://github.com/user-attachments/assets/3734b797-f9e2-4318-a75b-9919085b4d8c" />

* I run both crawler and it created two tables

<img width="1575" height="400" alt="image" src="https://github.com/user-attachments/assets/eac83175-7720-4311-8d60-4d89fd74dfff" />

<img width="1575" height="241" alt="image" src="https://github.com/user-attachments/assets/ddf591f1-4617-46ef-96e6-331eab2e1f3f" />

8) Before analysing the insights I set up a new bucket for the query output at set it up in AWS Athena as `business-insights-ys39h3` and run an easy query with join to see if everything works properly

<img width="1206" height="203" alt="image" src="https://github.com/user-attachments/assets/024836cd-de5e-4afe-af9b-09205057b891" />

```
SELECT 
    s.date, 
    m.dish_name, 
    m.calories, 
    m.cuisine, 
    m.pricePerServing
FROM "restaurant_data"."internal_sales_ys39h3" s
JOIN "restaurant_data"."enriched_menu_ys39h3" m 
  ON s.dish_id = m.dish_id;
```
<img width="1414" height="602" alt="image" src="https://github.com/user-attachments/assets/8d6989e0-0a92-410e-8f79-c9798c7b47fb" />

[Test-query.csv](https://github.com/user-attachments/files/24421340/Test-query.csv)

Here we can see that we are able to pull data from the two seperet table for more precise business insights

9) Lastly I created a joint table for easier analetics in Athena
```
CREATE TABLE "restaurant_data"."final_analytics_table"
WITH (
     format = 'PARQUET',
     external_location = 's3://business-insights-ys39h3/joined_data/'
) AS 
SELECT s.*, m.calories, m.cuisine, m.healthScore, m.diets
FROM "restaurant_data"."internal_sales_ys39h3" s
JOIN "restaurant_data"."enriched_menu_ys39h3" m 
  ON s.dish_id = m.dish_id;
```
Now I have 3 tables for the analitics 

<img width="408" height="141" alt="image" src="https://github.com/user-attachments/assets/91854914-232f-42fd-a12b-931190abc100" />

## First insights 
1) Revenue by Dish
Wanted to see if there any dis that is prices too low or even the revenue does not cover the cost of it.
```
SELECT 
    f.dish_name, 
    round(SUM(f.revenue)) AS total_revenue,
    SUM(f.units_sold) AS total_units,
    round(SUM(m.priceperserving * f.units_sold)) AS total_cost,
    round((SUM(f.revenue) - SUM(m.priceperserving * f.units_sold))) AS total_profit
FROM "restaurant_data"."final_analytics_table" AS f 
JOIN "restaurant_data"."enriched_menu_ys39h3" AS m 
    ON f.dish_id = m.dish_id
GROUP BY f.dish_name
ORDER BY total_profit DESC;
```
<img width="1432" height="619" alt="image" src="https://github.com/user-attachments/assets/6877d240-1088-4863-bc0e-8de4819c1c76" />

From this query we can get multiple conclusions:
* *Grilled Chuck Burgers* and the *Pork Menudo* are the best performing dishes both in total unite sold (352 and 401) but they have the highest profit too (14136.0 and 23767.0), so keeping these on the menu is a grat strategy for the business
* On the other hand the *Cilantro Lime Halibut* and the *Fish Pie With Fresh and Smoked Salmon* is really not profitable dishes as both have a grat loss probabaly due to the high ingredients costs, as even with the disent unit sold (168 and 214) on this price it really not wort it to have it on the menu with negative profit. And as the loss is high seemingly the price increase wont solve the issue. Both fish dishes so it would help profit to find similar dishes with much ceaper ingredients for the future.
* *Whole Wheat Dinner Rolls* has a high profit margin (86.6%), but it has the lowest total revenue (1,007). Because the price is low (37.31) and volume is very low (27 units) too, it contributes almost nothing. It might be useful to consider choosing a more popular appetizer insted of this one.

2) Revenue by Cousine
I wanted to identify which cuisine type is the biggest money-maker. It helps the owners decide which styles of food to feature more prominently on their menu.
```
SELECT 
    cuisine, 
    SUM(revenue) AS total_revenue,
    SUM(units_sold) AS total_units
FROM "restaurant_data"."final_analytics_table"
GROUP BY cuisine
ORDER BY total_revenue DESC;
```
<img width="1450" height="314" alt="image" src="https://github.com/user-attachments/assets/2e63038f-2392-4b71-b796-241d7f6926d5" />

From this query we can get some important conclusions:
* It is clearly identifiable Mediterranean cuisine is driving the highest gross income, so the owners should consider expanding that section of the mediterran dishes or advertise the other type dishes in a promotion to drive sales there too

3) Top Selling Healthy Dishes
By looking for dishes with a high healthscore, the owner can see if the health-conscious options are actually selling than the indulgent items
```
SELECT 
    dish_name, 
    healthscore, 
    SUM(units_sold) AS total_units_sold, 
    SUM(revenue) AS total_revenue
FROM "restaurant_data"."final_analytics_table"
WHERE healthscore > 50
GROUP BY dish_name, healthscore
ORDER BY total_units_sold DESC
LIMIT 10;
```
<img width="1439" height="320" alt="image" src="https://github.com/user-attachments/assets/d6d7326a-7dea-4d6b-a4c0-0bbb0d5453d0" />

* *Fish Pie With Fresh and Smoked Salmon* is the leading healthy dish, with a health score of 80.0 and the highest sales volume at 214 units, but as we earlier established it has a big loss. For the next menu needs a similarly healthy dish that makes profit 
* *Slow Cooker Beef Stew* achieved a perfect health score of 100.0 while maintaining very high popularity with 213 units sold, might indicating customers do not feel they are sacrificing flavor for health with this item, so the healthy items have a place on the menu too
* *Corn Avocado Salsa* scores significantly lower suggesting that customers are more willing to spend on healthy "main" dishes than healthy "appetizers"

4) Sales Volume by Dietary Requirement
This query groups sales by the diets strings, it allows the owners to see the demand for specific dietary needs like "vegan" or "gluten-free"
```
SELECT 
    diets, 
    SUM(units_sold) AS total_units_sold
FROM "restaurant_data"."final_analytics_table"
WHERE diets != ''
GROUP BY diets
ORDER BY total_units_sold DESC;
```

<img width="1432" height="508" alt="image" src="https://github.com/user-attachments/assets/b20155a5-e093-44dd-b032-5198f00f62bb" />

Some insights from the query:
* Dishes labeled as both "gluten free, dairy free" are clearly the most popular with 614 units sold
* The second most popular category involves a broad range of restrictions (gluten free, dairy free, paleolithic, lacto ovo vegetarian, primal, whole 30, and vegan), this suggests the customers heavily favor dishes that cater to multiple dietary needs
* Gluten-Free Prioritysation as every single one of your top six selling dietary categories includes "gluten free" as a standard. This indicates that gluten-free compatibility is a critical driver for the sales volume
* Since the combination of "gluten free, dairy free" is your top seller, the ovners should ensure these are clearly marked on the physical menuso they can further boost sales

5) Average Calories per Category vs. Popularity
This query helps understand the nutritional profile of the different categories (appetizers, mains, desserts) and if customers prefer lower-calorie or higher-calorie options in those groups
```
SELECT 
    category, 
    AVG(calories) AS avg_calories, 
    SUM(units_sold) AS total_popularity
FROM "restaurant_data"."final_analytics_table"
GROUP BY category
ORDER BY total_popularity DESC;
```

<img width="1445" height="279" alt="image" src="https://github.com/user-attachments/assets/64f7f04a-bbbe-4ed2-bea8-eb4546bee340" />

* The "main" category is overwhelmingly the most popular, with a total popularity score of 1,932 and customers are willing to accept higher caloric counts for their primary meal, as the "main" category also carry the highest average calories at approximately 364 kcal (what seems very little, probably due to some data quality issue)
* Desserts are significantly more popular than appetizers, despite desserts having a higher average caloric density. This suggests that when customers deviate from a main course, they are more likely to choose a higher-calorie sweet treat over a lighter starter
* Appetizers are currently the least popular category. They also have the lowest average calories, so the owners could try introducing more "indulgent" appetizers to see if increasing the caloric density would drive higher engagement in this segment

## Re-upload
As goal was to create a data pipeline that is capable analysing an updated version of the menu and analysing further sales. 
1) Reuploading the current menu to the `current-menu-ys39h3` bucket with the same name but changed menu items

<img width="1571" height="391" alt="image" src="https://github.com/user-attachments/assets/0bb1b0ee-47df-4b1f-9fa6-ead1b0b65573" />

2) Our Lambda function will automaticly regeneret the enriched bucket with the new wnriched data

<img width="1568" height="405" alt="image" src="https://github.com/user-attachments/assets/9fe0a4dc-499c-4072-a0ac-9fd3e274b617" />

3) I uploaded the the december sales to the `internal-sales-ys39h3` bucket

<img width="1569" height="393" alt="image" src="https://github.com/user-attachments/assets/f3ac8c3e-25f3-4dcc-8987-3f9cfd517eaf" />

4) Re-run both of the crawlers

<img width="1587" height="319" alt="image" src="https://github.com/user-attachments/assets/9b76efb5-fc51-4111-8dbb-c47242631c14" />

5) Now we can run some queries in Athena on this month's sales data, enriched with the information of the new menu items

## Second insight
1) Joining the tables again for easier queriing
```
CREATE TABLE "restaurant_data"."second_menu_analytics_table"
WITH (
     format = 'PARQUET',
     external_location = 's3://business-insights-ys39h3/joined_data_second/'
) AS 
SELECT s.*, m.calories, m.cuisine, m.healthScore, m.diets
FROM "restaurant_data"."internal_sales_ys39h3" s
JOIN "restaurant_data"."enriched_menu_ys39h3" m 
  ON s.dish_id = m.dish_id;
```
2) Profit per dish
```
SELECT 
    s.dish_name, 
    round(SUM(s.revenue)) AS total_revenue,
    SUM(s.units_sold) AS total_units,
    round(SUM(m.priceperserving * s.units_sold)) AS total_cost,
    round((SUM(s.revenue) - SUM(m.priceperserving * s.units_sold))) AS total_profit
FROM "restaurant_data"."second_menu_analytics_table" AS s
JOIN "restaurant_data"."enriched_menu_ys39h3" AS m 
    ON s.dish_id = m.dish_id
GROUP BY s.dish_name
ORDER BY total_profit DESC;
```

<img width="1445" height="625" alt="image" src="https://github.com/user-attachments/assets/aaba891a-26e5-4d49-b1d0-9105c160d5f2" />

* We can clearly see that non of the dishes generate loss
* And especialy the new fish dish, the *Salmon Quinoa Risotto* is clearly was a good addition to the menu, but the other new dishes are performing moderatly well too in terms of generating profit

3) Dish category popularity
```
SELECT 
    category, SUM(units_sold) AS total_popularity
FROM "restaurant_data"."second_menu_analytics_table"
GROUP BY category
ORDER BY total_popularity DESC;
```

<img width="1430" height="268" alt="image" src="https://github.com/user-attachments/assets/644dd7ac-2151-41e8-8216-3c9a7f760d9b" />

* Even with the better performing new appetizer the people not prefer to order appetizers, they much reather order a dessert at the end of the meal
* It could be good to test with different menus that people wants more/different options of appetizers or just not that open for it generaly, than keeping few not that costly appetizer otpions can be a good way to go

4) Continuing the analyzis further
* The second version can be of course further analysed but it is clearly visible how can different insights and information help the owners figure out the proper menu
* These steps can be repeted multiple times with bigger and smaller changes of the menu or the prices of the items on it
* It could be insightful to add new data to the enriched-menu so new insight could be drawn from it 


## Limitations
This project have multiple limitations and some anomaies that could be an issue in a real life inplementation: 
* Nameing convention: In the way I used the API needs the dish names to perfectly match with the ones the API find on different food websites, so the restaurant can't get creative with the name
* Data validity and realism: I took the data that the API found valid and didn't fact checked it, for example one of the dish per serving costs more than 1000, as there is no mesurment I took it as $ value but probably it is either a mistake or not in that measurment in reality
* API limitation: with the free version I couldn't make a larger scale menu as I wouldn't be able to finish both round of API calls in one day but with a larger scale menu it can be an issue too
* It does not account for multiple questions: Like storage length of different ingredients generating waste therefor loss or different times of the year like holidays, summer, seasonal items
