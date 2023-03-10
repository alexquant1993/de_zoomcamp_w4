# Week 4 - Analytics Engineering
Steps to follow to complete the assignment:

1. Create a new repository on your GitHub account
2. Create a BigQuery service account
    - Open the BQ credential wizard to create a service account in your taxi project.
    - Grant BigQuery Admin role to the service account.
    - Download the JSON key file.
3. Create a dbt cloud project
    - Create a dbt cloud account
    - Name your project
    - Choose BQ as your data warehouse
    - Upload the JSON service account key
    - Set up development credentials and specify the dataset name
    - Click on Test and continue with the GitHub repository setup
    - Select git clone and paste the SSH key from your repo
    - You will get a deploy key, head to your GH repo and go to the settings tab. Under security, deploy keys, add the key you just copied. Make sure to tick on write access.
4. Initialize dbt project
    - Create a new branch in your repo
    - Change branch in the dbt IDE and then click initialize dbt project
    - Edit the dbt_project.yml file with your project name

## Build the first dbt models
### dbt models
Dbt models are SELECT statements written in a SQL file. An interesting feature of dbt is that we can integrate SQL statements with the Jinja templating language. Jinja code is always written between {{ ... }} or {% ... %} and can be used to call macros (which are similar to functions in standard programming languages) that allow us to dynamically generate and reuse SQL statements. After compiling the dbt models, the Jinja blocks are substituted by the SQL statement generated by them.

A generic example of a dbt model is:
```sql
{{ config(materialized='table') }}
SELECT * FROM staging.source_table WHERE record_state = 'ACTIVE'
```
The config() macro is used to define settings related to materialization, which can be one of the following:
- table: the model will be materialized as a table in the data warehouse. This will create a table with the name of my model in the schema currently being used. 
- view: the model will be materialized as a view in the data warehouse
- ephemeral: when a model is marked as ephemeral, it will be created as a temporary table during the dbt run, and will be automatically dropped when the run is complete. Ephemeral models can be useful in certain situations, such as when you need to create a temporary table for use in a complex query or transformation, or when you want to avoid cluttering your database with intermediate tables that are only needed for a single dbt run,
- incremental: the model will be materialized as a table in the data warehouse and will be updated incrementally. It allows us to run our model incrementally.

The example above would be compiled by dbt in the data warehouse as the following SQL code:
```sql
CREATE TABLE my_schema.my_model as (
    SELECT * FROM staging.source_table WHERE record_state = 'ACTIVE'
)
```

## The FROM clause of a dbt model
### Sources
- The data loaded to our data warehouse that we use as sources for our model.
- Configuration defined in the yml files in the models folder.
- Used with the source macro that will resolve the name to the right schema, plus build the dependencies automatically, i.e. `{{source('staging', 'source_table'))}}`.
- Source freshness can be defined and tested. For instance, check if the data is fress six times every hour. Defined in the yml files in `freshness` of a given table.

### Seeds
- CSV files stored in our repository under the seed folder
- Benefits of version controlling
- Equivalent to a copy command
- Recommended for data that doesn't change frequently
- Runs with `dbt seed -s file_name`
- `FROM {{ source('staging','green_tripdata') }}`

### Ref

- Macro to reference the underlying tables and views that were building the data warehouse
- Run the same code in any environment, it will resolve the correct schema for you
- Dependencies are built automatically

Example of dbt model:
```sql
with green_data as (
    select *, 'green' as service_type 
    from {{ ref('green_tripdata') }}
)
```
Compiled SQL code in the production environment:
```sql
with green_data as (
    select *, 'green' as service_type 
    from "production"."dbt_development"."green_tripdata"
)
```

## Run a dbt model

- Create a new folder with the model name in the models directory
- Create a schema.yml file with the model configuration
```yml
version: 2

sources:
  - name: staging
    database: dtc-data-engineering-377412
    schema: trips_data_all

    tables:
      - name: green_tripdata
      - name: yellow_tripdata
```
- Create a model.sql file with the SQL code
```sql
{{ config(materialized = "view") }}

select
    *
from {{ source('staging', 'green_tripdata') }}
limit 100	
```
- Run model with `dbt run -m model_name` and you will be able to check the created view in the data warehouse under the selected schema when the dbt project was initialized. You can use as well: `dbt run --select model_name`.

## Macros
- Use control structures (e.g. if statements and for loops) in SQL
- Use environment variables in your dbt project for production deployments
- Operate on the results of one query to generate another query
- Abstract snippets of SQL into reusable macros - these are analogous to functions in most programming languages.
```sql
-- Definition in the macros folder
{# 
    This macro returns the description of the payment_type
#}
{% macro get_payment_type_description(payment_type) -%}
   
   case {{ payment_type }}
    when 1 then 'Credit card'
    when 2 then 'Cash'
    when 3 then 'No charge'
    when 4 then 'Dispute'
    when 5 then 'Unknown'
    when 6 then 'Voided trip'
   end  

{%- endmacro %}

-- Usage of the macro
select 
    {{ get_payment_type_description('payment_type') }} as payment_type_description,
    congestion_surcharge::double_precision
from {{ source('staging', 'green_tripdata_2021_01') }}
where vendorid is not null

-- Compiled SQL code
create or alter view production.dbt_development.stg_green_trip_data as (
    select 
        case payment_type
            when 1 then 'Credit card'
            when 2 then 'Cash'
            when 3 then 'No charge'
            when 4 then 'Dispute'
            when 5 then 'Unknown'
            when 6 then 'Voided trip'
        end as payment_type_description,
        congestion_surcharge::double_precision
    from "production"."trips_data_all"."green_tripdata_2021_01"
    where vendorid is not null
)
```

## Packages
- Like libraries in other programming languages
- Standalone dbt projects, with models and macros that tackle a specific problem
- By adding a package to your project, the package's models and macros will become part of your own project
- Imported in the packages.yml file and imported by running dbt deps
- A list of useful packages can be find in dbt packages hub

### Process to add a package to your project

- Add the package to the packages.yml file
```yml
packages:
  - package: dbt-labs/dbt_utils
    version: 0.8.0
```
- Call the package in the model
```sql
{{ dbt_utils.dateadd('day', 1, 'pickup_datetime') }}
{{ dbt_utils.surrogate_key(['vendorid', 'lpickup_datetime']) }}
```
- Install the package with `dbt deps`

## Variables
- Variables are useful for defining values that should be used across the project
- With a macro, dbt allows us to provide data to models for compilation
- To use a variable we use the {{ var('...') }} function
- Variables can be defined in two ways:
    - In the dbt_project.yml file
    ```yml
        vars:
            payment_type_values: [1, 2, 3, 4, 5, 6]
    ```
    - On the command line: `dbt build --m model_name --var "{'is_test_run': false}"` and in the code we can use it as `{{ var('is_test_run', default=true) }}`

## Seeds
- CSV files stored in our repository under the seed folder
- Benefits of version controlling
- Equivalent to a copy command
- Files that don't change frequently
Steps to create a seed:
- Create a csv file in the seeds folder
- Create this table in our data warehouse with `dbt seed -s file_name`
- Create a sql model that uses the seed
```sql
{{ config(materialized='table') }}

select 
    locationid, 
    borough, 
    zone, 
    replace(service_zone,'Boro','Green') as service_zone
from {{ ref('taxi_zone_lookup') }}
```
- Create an entry for the seeds in the dbt_project.yml file with certain configurations
```yml
seeds:
  taxi_rides_ny:
    taxi_zone_lookup:
     +column_types:
       locationid: numeric
```
- When there is a change in the csv file, we can drop the previous table and update it with: `dbt seed --full-refresh -s file_name`

### Commands
- To run a given model: `dbt run -m model_name` or `dbt build --select model_name`
- To run all models: `dbt run`
- To run all models and seeds: `dbt build`
- To run a given model and its dependencies: `dbt build --select +model_name`

## Testing and documenting dbt models
### Testing
- Assumptions that we make about our data
- Tests in dbt are essentially a select sql query
- These assumptions get compiled to sql that returns the amount of failing records
- Test are defined on a 'column' in the .yml file
```yml
- name: payment_type_description
  description: Description of the payment_type code
  tests:
    - accepted_values:
        values: [1, 2, 3, 4, 5]
        severity: warn
```
```yml
- name: Pickup_locationid
  description: locationid where the meter was engaged
  tests:
    - relationships:
        to: ref('taxi_zone_lookup')
        field: locationid
        severity: warn
```
- dbt provides basic tests to check if the column values are:
    - Unique
    - Not null
    - Accepted values
    - A foreign key to another table
```yml	
columns:
    - name: tripid
      description: Primary key for the trip
      tests:
        - unique:
            severity: warn
        - not_null:
            severity: warn
```
- You can create your custom tests as queries

### Documentation
- dbt provides a way to generate documentation for your dbt project and render it as a website.
- The documentation for your project includes:
    - Information about your project:
        - Model code - both from .sql file and compiled
        - Model dependencies
        - Sources
        - Auto generated DAG from the ref and source macros
        - Descriptions (from .yml file) and tests
    - Information about your data warehouse (information schema):
        - Column names and data types
        - Table stats like size and rows
- dbt docs can also be hosted in dbt cloud
```yml
models:
    - name: dim_zones
      description: >
        List of unique zones identified by locationid. Includes the service zone they correspond to Green, Yellow or FHV.
    - name: fact_trips
      description: >
        Taxi trips corresponding to both Green and Yellow taxis. The table contains records where both pickup and dropoff locations are within the 5 boroughs of New York City. Each record represents a single trip uniquely identified by tripid.
```

## Deployment
### What is deployment?
- Develop -> Test and document -> Deployment (Version control and CI/CD)
- Process of running the models we created in our development environment in a production environment
- Development and later deployment allows us to continue building models and testing them without affecting our production environment
- A deployment environment will normally have a different schema in our data warehouse and ideally a different user
- A development - deployment workflow will be something like:
    - Develop in a user branch
    - Open a pull request to merge into the main branch
    - Merge the branch to the main branch
    - Run the new models in the production environment using the main branch
    - Schedule the models

### Running a dbt project in production
- dbt cloud includes a scheduler where to create jobs to run in production
- A single job can run multiple commands
- Jobs can be triggered manually or on a schedule
- Each job will keep a log of the runs over time
- Each run will have the logs for each command
- A job could also generate documentation, that could be viewed under the run information
- If dbt source freshness is enabled, the results can also be viewed at the end of a job

### What is Continous Integration - CI?
- CI is the practice of regularly merge development branches into a central repository, after which automated builds and tests are run
- The gol is to reduce adding bugs to the production code and maintain a more stable project
- dbt allows us to enable CI on pull requests
- Enabled via webhooks from Github or Gitlab
- When a PR is ready to be merged, a webhooks is received in dbt Cloud that will enqueue a new run of the specified job.
- The run of the CI job will be against a temporary schema
- No PR will be able to be merged unless the run has been completed successfully

