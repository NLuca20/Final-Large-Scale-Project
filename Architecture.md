# Architecture
## Data
The project requires internal data of the restaurant I gerenated these data sources and turned them into CSV files. 

Four CSV file was needed: 
* Original menu: [Current-menu.csv](https://github.com/user-attachments/files/24422718/Current-menu.1.csv)
* Sales data between 2025.09.01. and 2025.11.30. : [Sales-sept-nov.csv](https://github.com/user-attachments/files/24422742/Sales-sept-nov.csv)
* Changed menu: [Current-menu.csv](https://github.com/user-attachments/files/24422744/Current-menu.csv)
* Sales data between 2025.12.01. and 2025.12.31. : [Sales-dec.csv](https://github.com/user-attachments/files/24422747/Sales-dec.csv)

## API
The plan of the optimasiation contained enrichment of the menu items with additional data from an external sorce by the [spoonacular API](https://spoonacular.com/food-api). 
This API provides multiple meaningful insightes of the given recipies trought a `lambda` function every time the menu file is updated.

It provides insights on: 
* Nutritional value
* Price per Serving
* Dietary belonging
* Cousine


## Architecture
<img width="1006" height="476" alt="image" src="https://github.com/user-attachments/assets/c6cc8a16-e663-44c9-a1f1-596a6762ad1e" />

**Amazon S3**: Storage space for the internal menu, the sales data, the enriched menu data and the queri results

**AWS Lambda**: Ingests data from an API call from the spoonacular API every time new menu is uploaded to the `S3 bucket` and saves the result to an other `S3 bucket`

**AWS Glue Crawler**: Prepers data for athena by creating tables of the enriched menu dataset and the historical sales dataset

**Amazon Athena**: SQL query tool that lets us analyze the joined data directly

## Cost
**S3 bucket** Monthly cost: 0.02 USD/bucket -> 4 bucket 0.08 USD

**AWS Lambda** The Lambda free tier includes 1M free requests per month and 400,000 GB-seconds of compute time per month -> for my project volume it is free 

**AWS Glue Crawler** Monthly cost: 0.15 USD for the two crawlers

**Amazon Athena** Monthly cost: 0.22 USD

**Total Monthly costs** 0.45 USD -> **Yearly costs** under 6 USD


