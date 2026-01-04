[Test-query.csv](https://github.com/user-attachments/files/24421335/Test-query.csv)# Implementation
## Set up 
### Check IAM roles

### Set up S3 buckets
1) Created 3 buckets for three different puruses:
* The `current-menu-ys39h3` holding the uploaded CSV data in Parquet format of the current menu items
* The `internal-sales-ys39h3` holding the uploaded CSV data in Parquet format of the historical sales data
* The `enriched-menu-ys39h3` endpoint of the lambda function with the detaled menu items

<img width="1241" height="307" alt="Képernyőkép 2026-01-03 214636" src="https://github.com/user-attachments/assets/104a06be-6d66-4b2b-9aa0-f9a2699ba33d" />

I added my cusman ID to the names to make sure the name of the buckets are unique in the system

2) Creating the Lambda function to enrich the current menu items from the spoonacular API
* I named the function `SpoonacularEnrichment`
* I set the Runtime to Python 3.14
* I set the Execution role to LabRole
* Writhe the Python code to the Code source part
```import json
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
        
        search_params = {
            "query": dish_name, 
            "addRecipeNutrition": "true", 
            "addRecipeInformation": "true", 
            "number": 1, 
            "apiKey": API_KEY
        }
        
        search_resp = requests.get(f"{BASE_URL}/complexSearch", params=search_params)
        
        if search_resp.status_code != 200:
            print(f"Search failed for {dish_name}: {search_resp.status_code}")
            continue
            
        search_results = search_resp.json().get('results')
        if not search_results:
            continue
            
        recipe = search_results[0]
        
        nutrients_list = recipe.get('nutrition', {}).get('nutrients', [])
        nutrients = {n['name']: n['amount'] for n in nutrients_list}

        cuisine_params = {"apiKey": API_KEY}
        cuisine_data = {"title": dish_name}
        cuisine_resp = requests.post(f"{BASE_URL}/cuisine", params=cuisine_params, data=cuisine_data)
        cuisine_res = cuisine_resp.json() if cuisine_resp.status_code == 200 else {"cuisine": "Unknown"}

        raw_diets = recipe.get("diets", [])
        if raw_diets and isinstance(raw_diets[0], dict):
            diet_string = ", ".join([d.get('element', '') for d in raw_diets])
        else:
            diet_string = ", ".join(raw_diets)

        enriched_item = {
            "dish_id": row.get('dish_id'),
            "dish_name": dish_name,
            "course": row.get('course'),
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
            "diets": diet_string,
            "cuisine": cuisine_res.get("cuisine", "Unknown")
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

<img width="1899" height="393" alt="image" src="https://github.com/user-attachments/assets/a721e28d-bff6-4b98-b8a2-281d3143c3c3" />

* I edited the General Configuaration to give more time out

<img width="1544" height="241" alt="image" src="https://github.com/user-attachments/assets/719b6a2d-d1e6-4199-b5c8-2c6ec20f3ca2" />

* I added a triger to update every time the current-menu gets a new menu 

<img width="1540" height="484" alt="image" src="https://github.com/user-attachments/assets/964109e8-2f28-4078-8f73-cc2c557763fd" />

3) Upload the Current Menu data to the `current-menu-ys39h3` bucket

<img width="1548" height="415" alt="image" src="https://github.com/user-attachments/assets/c3299222-aa80-49d9-b0cf-4c8b78ce68b5" />

4) The enriched data automaticly loades to the `enriched-menu-ys39h3` bucket

<img width="1551" height="439" alt="image" src="https://github.com/user-attachments/assets/7b15d0a2-2cd8-431b-a45a-63a91fc0b516" />

* I checked the format so I can see that everything is fine 

<img width="1487" height="551" alt="image" src="https://github.com/user-attachments/assets/8b98d7b1-cb3a-4fbd-9ed5-9730a2b728ae" />

5) Upload the sales historical data to the `internal-sales-ys39h3` bucket

<img width="1889" height="411" alt="image" src="https://github.com/user-attachments/assets/ec8eb8bb-51cd-411d-9bdb-e3da9639ad6b" />

6) Use AWS Glue crawler to join all the data 
* I created a crawler for the `internal-sales-ys39h3` bucket
* I set a new `restaurant-data table` as the output place

<img width="1236" height="662" alt="image" src="https://github.com/user-attachments/assets/ef0a6e6d-b7ed-4002-9718-4c1e62bea7ef" />

* And a crawler for the `enriched-menu-ys39h3` bucket

<img width="1239" height="666" alt="image" src="https://github.com/user-attachments/assets/3734b797-f9e2-4318-a75b-9919085b4d8c" />

* I run both crawler and it created two tables

<img width="1575" height="400" alt="image" src="https://github.com/user-attachments/assets/eac83175-7720-4311-8d60-4d89fd74dfff" />

<img width="1575" height="241" alt="image" src="https://github.com/user-attachments/assets/ddf591f1-4617-46ef-96e6-331eab2e1f3f" />

7) Before analysing the insights I set up a new bucket for the query output at set it up in AWS Athena as `business-insights-ys39h3` and run an easy query with join to see if everything works properly

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

## First insights 
## Re-upload
## Second insight
## Limitations

