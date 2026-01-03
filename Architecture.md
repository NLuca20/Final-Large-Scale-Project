# Architecture
## Data
The project requires internal data of the restaurant I gerenated these data sources and turned them into CSV files. 

Four CSV file was needed: 
* Original menu: [Current-menu.csv](https://github.com/user-attachments/files/24417262/Current-menu.csv)
* Sales data between 2025.09.01. and 2025.11.30. :[Sales-sept-nov.csv](https://github.com/user-attachments/files/24417252/Sales-sept-nov.csv)
* Changed menu: 
* Sales data between 2025.09.01. and 2025.12.31. :
 
## API
The plan of the optimasiation contained enrichment of the menu items with initial data from an external sorce by the [spoonacular API](https://spoonacular.com/food-api). 
This API provides multiple meaningful insightes of the given recipies trought a `lambda` function every time the menu file is updated.

It provides insights on: 
* Nutritional value
* Dietary belonging
* Cousine
## Architecture
**Amazon S3**: Storage space for the internal menu, the sales data and the enriched menu data

**AWS Cloud9 IDE instance**: Transforms the `CSV` data files to the `Apache Parquet` format and uploads them to `Amazon S3`

**AWS Lambda**: Ingests data from an API call from the spoonacular API every time new menu is uploaded to the `S3 bucket` and saves the result to an other `S3 bucket`

**AWS Glue Crawler**: Joins the enriched menu dataset and the historical sales dataset

**Amazon Athena**: SQL query tool that lets us analyze the joined data directly


