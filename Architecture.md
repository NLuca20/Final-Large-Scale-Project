# Architecture
## Data
The project requires internal data of the restaurant I gerenated these data sources and turned them into CSV files. 

Four CSV file was needed: 
* Original menu: [Current-menu.csv](https://github.com/user-attachments/files/24422718/Current-menu.1.csv)
* Sales data between 2025.09.01. and 2025.11.30. : [Sales-sept-nov.csv](https://github.com/user-attachments/files/24422742/Sales-sept-nov.csv)
* Changed menu: [Current-menu.csv](https://github.com/user-attachments/files/24422744/Current-menu.csv)
* Sales data between 2025.12.01. and 2025.12.31. : [Sales-dec.csv](https://github.com/user-attachments/files/24422747/Sales-dec.csv)

## API
The plan of the optimasiation contained enrichment of the menu items with initial data from an external sorce by the [spoonacular API](https://spoonacular.com/food-api). 
This API provides multiple meaningful insightes of the given recipies trought a `lambda` function every time the menu file is updated.

It provides insights on: 
* Nutritional value
* Price per Serving
* Dietary belonging
* Cousine


## Architecture
<img width="1026" height="494" alt="image" src="https://github.com/user-attachments/assets/a50c4250-c14d-4fa0-ae56-452f4ffe06ea" />

**Amazon S3**: Storage space for the internal menu, the sales data, the enriched menu data and the queri results

**AWS Lambda**: Ingests data from an API call from the spoonacular API every time new menu is uploaded to the `S3 bucket` and saves the result to an other `S3 bucket`

**AWS Glue Crawler**: Joins the enriched menu dataset and the historical sales dataset

**Amazon Athena**: SQL query tool that lets us analyze the joined data directly


