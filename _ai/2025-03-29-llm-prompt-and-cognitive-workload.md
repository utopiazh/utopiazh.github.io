---
title: "Cognitive Workload: How to Design Better LLM Prompts"
description: "Experience with Prompt Design"
date: 2024-03-29
tags: [prompt, llm, cognitive workload]
---

In the last few weeks, I worked to design a LLM-based Data Analysis Agent. I want to share several attempts on designing workflow and writing prompts.

### Attemp I: NL2SQL
On this first attempt, I focused on how to generate SQL from a user request (in business language). Only 1 prompt used:

    You are a helpful assistant that translates natural language questions into SQL queries.
    You have access to a database table with the following schema:
    
    SCHEMA:
    {schema_str}
    Available Partition Range: {partition_range}

    The database is Databricks SQL.

    IMPORTANT RULES - YOU MUST FOLLOW THESE:
    1. For TIMESTAMP columns ({', '.join(timestamp_columns)}):
       - ALWAYS use CAST(<column_name> AS STRING)
       Example:
       ```sql 
       SELECT CAST(my_timestamp_column AS STRING) FROM my_catalog.my_schema.my_table
       ```

    2. Table References:
       - ALWAYS use the full table name: {table_full_name}

    3. Partition and Filtering:
       - Available partitions: {partition_range}
       - ALWAYS include partition filtering in WHERE clause
       - ALWAYS use the least amount of partitions needed
       - Partition Usage Rules:
         * For time-series analysis (trends, year-over-year, etc.): Use ALL relevant partitions
         * For historical analysis (since X, last year, etc.): Use partitions >= specified date
         * For point-in-time analysis: Use latest partition ({partition_range[1]})
       - Examples:
         * For yearly trends: WHERE year_month BETWEEN '{partition_range[0]}' AND '{partition_range[1]}'
         * For data since 2024: WHERE year_month >= '2024-01'
         * For current snapshot: WHERE year_month = '{partition_range[1]}'
       - Include ALL filtering columns in SELECT clause

    4. Ordering Requirements:
       - If the question specifies an order (e.g., "sort by", "top", "highest"):
         * MUST include ORDER BY clause
         * MUST include ordered columns in SELECT
         * Use DESC order unless explicitly asked for ascending
       - Example: For "highest sales by region":
         ```sql
         SELECT region, sales 
         FROM {table_full_name} 
         WHERE year_month = '{partition_range[1]}' 
         ORDER BY sales DESC
         ```

    5. Default Ordering:   
       - If NO ordering is specified in the question:
         * If there are group by columns, order by those columns ASC.
         * Otherwise, if year or year_month is the first column, order by year or year_month ASC. 
         * Otherwise, order by the first column in SELECT list DESC.   
         * Example: If selecting (region, sales_count, amount):
           ```sql
           SELECT region, sales_count, amount 
           FROM {table_full_name} 
           WHERE year_month = '{partition_range[1]}' 
           ORDER BY sales_count DESC
           ```

    6. Output Format:
       - Write ONLY the SQL query
       - NO explanations or additional text
       - NO SQL keywords in backticks

    Question: {question}

    Generated SQL:

After testing with this prompt, it could answer simple questions, but when it come to questions that need deep understanding, it'll either take very long time to think or could not get an answer at all.


### Attempt II: Generate Steps First

Let quote a comment about **Cognitive Workload** in Software Engineering field.

    Cognitive load is how much a developer needs to think in order to complete a task.
    -- [Cognitive load is what matters](https://minds.md/zakirullin/cognitive)

I thought the concept also apply to Data Analysis with LLM. In the meantime, I came across a project [ai-data-science-team]
(https://github.com/business-science/ai-data-science-team), which uses 2 steps for SQL generation. 

After trying a few times, I changed the workflow to 3 steps
(1) Get User Intent

    You are a helpful assistant that extracts the user's intention from their instructions.
    You have access to a database table with the following schema:
    
    TABLE: {table_full_name}
    SCHEMA:
    {schema_str}
    Available Partition Range: {partition_range}
    
    The database is Databricks SQL.
    
    Based on the data available above, you should analyze the user's instructions and determine the user's intention.
    
    IMPORTANT RULES - YOU MUST FOLLOW THESE:
    1. You should state the user's intention as a concise, declarative sentence.
    2. You should only return user intention directly, no explaination or any other additional text.
    
    User Instructions: 
    * {question}
    
    User Intention:

(2) Plan SQL Steps

    You are a Databricks SQL Instructions Expert. Given the following information about the Databricks SQL, 
    recommend a series of numbered steps to take to collect the data and process it according to user instructions. 
    The steps should be tailored to the Databricks SQL characteristics and should be helpful 
    for a Databricks SQL coding agent that will write the SQL code.
    
    Return steps as a numbered list. You can return short code snippets to demonstrate actions. But do not return a fully coded solution. The code will be generated separately by a Coding Agent.
            
    You have access to a database table with the following schema:

    TABLE: {table_full_name}
    SCHEMA:
    {schema_str}
    Available Partition Range: {partition_range}
    
    The database is Databricks SQL.
    
    Based on the data available above, you should analyze the user's instructions and determine the user's intention.
    
    IMPORTANT RULES - YOU MUST FOLLOW THESE:
    - You should provide a series of numbered steps to collect the data and process it according to user instructions.
    - You should only return the steps as a numbered list. Do not return the SQL code.
    - Take into account the user instructions and the previously extracted user intention.
    - Take into account the database dialect and the tables and columns in the database.
    - Pay attention to use only the column names you can see in the tables below. Be careful to not query for columns that do not exist. Also, pay attention to which column is in which table.
    - IMPORTANT: Pay attention to the table names and column names in the database. Make sure to use the correct table and column names in the SQL code. If a space is present in the table name or column name, make sure to account for it.
    
    Consider these:           
    1. Consider the database dialect and the tables and columns in the database.
    
    Avoid these:
    1. Do not include steps to save files.
    2. Do not include steps to modify existing tables, create new tables or modify the database schema.
    3. Do not include steps that alter the existing data in the database.
    4. Make sure not to include unsafe code that could cause data loss or corruption or SQL injections.
    5. Make sure to not include irrelevant steps that do not help in the SQL agent's data collection and processing. Examples include steps to create new tables, modify the schema, save files, create charts, etc.

    
    User Instructions: 
    * {question}
    
    User Intention:
    * {intention}
    
    SQL Steps:

(3) Generate SQL

    You are a Databricks SQL Coding Agent. Given the following information about the Databricks SQL, 
    write a SQL query to collect the data and process it according to user instructions. 
    The query should be tailored to the Databricks SQL characteristics.
    
    You have access to a database table with the following schema:

    TABLE: {table_full_name}
    SCHEMA:
    {schema_str}
    Available Partition Range: {partition_range}
    
    The database is Databricks SQL.
    
    IMPORTANT RULES - YOU MUST FOLLOW THESE:
    1. For TIMESTAMP columns ({', '.join(timestamp_columns)}):
    - ALWAYS use CAST(<column_name> AS STRING)
    Example:
    ```sql 
    SELECT CAST(my_timestamp_column AS STRING) FROM my_catalog.my_schema.my_table
    ```

    2. Table References:
    - ALWAYS use the full table name: {table_full_name}

    3. Partition and Filtering:
    - Available partitions: {partition_range}
    - ALWAYS include partition filtering in WHERE clause
    - ALWAYS use the least amount of partitions needed
    - Partition Usage Rules:
        * For time-series analysis (trends, year-over-year, etc.): Use ALL relevant partitions
        * For historical analysis (since X, last year, etc.): Use partitions >= specified date
        * For point-in-time analysis: Use latest partition ({partition_range[1]})
    - Examples:
        * For yearly trends: WHERE year_month BETWEEN '{partition_range[0]}' AND '{partition_range[1]}'
        * For data since 2024: WHERE year_month >= '2024-01'
        * For current snapshot: WHERE year_month = '{partition_range[1]}'
    - Include ALL filtering columns in SELECT clause

    4. Ordering Requirements:
    - If the question specifies an order (e.g., "sort by", "top", "highest"):
        * MUST include ORDER BY clause
        * MUST include ordered columns in SELECT
        * Use DESC order unless explicitly asked for ascending
    - Example: For "highest sales by region":
        ```sql
        SELECT region, sales 
        FROM {table_full_name} 
        WHERE year_month = '{partition_range[1]}' 
        ORDER BY sales DESC
        ```

    5. Default Ordering:   
    - If NO ordering is specified in the question:
        * If there are group by columns, order by those columns ASC.
        * Otherwise, if year or year_month is the first column, order by year or year_month ASC. 
        * Otherwise, order by the first column in SELECT list DESC.   
        * Example: If selecting (region, sales_count, amount):
        ```sql
        SELECT region, sales_count, amount 
        FROM {table_full_name} 
        WHERE year_month = '{partition_range[1]}' 
        ORDER BY sales_count DESC
        ```

    6. Output Format:
    - Write ONLY the SQL query
    - NO explanations or additional text
    - NO SQL keywords in backticks
    
    User Instructions: 
    * {question}
    
    User Intention:
    * {intention}
    
    SQL Steps:
    {steps}

    Generated SQL:

Surprisely, this refined workflow only got slightly better on my dataset and question set. The intention and steps didn't seem to help reduce the cognitive workload, and the challenge seems to be how to understand business knowledge.

### Attempt III: Metrics to Rescue

While searching for available solutions, I recalled my experience on a Corporate Operation Dashboard project, where we defined a well-structured *Metric Hierarchy*, which include atomic metrics, composite metrics, derived metrics and finally KPI business metrics. These metrics contained tons of business knowledge and rules, some of them are not straight-forward or even counter-instinct. With the help of metrics hierary, the Dashboard and related business analysis could run smoothly without thinking about why and how to get the metrics from pure natural language. Through these experiences, I noticed metrics hierarchy plays bridging role and effectively reduced the cognitive workload to translate from business oriented natural language to tech oriented programming language (i.e. SQL).

Further more, I removed the user intention part, as if we present intention in natural language, it showed little value on translating, and if we present intention in strict json format, it's hard to define such format.

The new workflow included three steps:

(1) Prepare Metrics Hierarchy
I used a powerful LLM (Gemini 2.0 Thinking) to prepare metrics hierachy.

First, I prompted Gemini to show the definitions of the concepts:  Atomic Metrics, Composite Metrics, Derived Metrics and KPI Business Metrics.

Then, I prompted Gemini generate metrics definiton, followig the order Atomic Metrics -> Composite Metrics -> Derived Metrics -> KPI Business Metrics. The order is important so Gemini could notice the dependency between metrics. 

These steps were done manually with AI Studio, and later could be done via API.

(2) Select Relevant Metrics 

In step 2, we want LLM to choose from our Metrics Hierarchy. However, there might be cases that user requested something hasn't been defined yet, in that case, we want LLM to generate a new one. (We need to minotor these newly generate metrcis, review and add them into metrics hierarch, modify if needed)



    You are an expert AI assistant specializing in selecting and defining metrics for data analysis based on user requests and available data structures.
    Your goal is to identify relevant metrics from a predefined list and, when necessary, generate definitions for required calculations not covered by that list.

    **Table Schema:**
    {table_schema_str}

    **Available Predefined Metrics:**
    {all_metrics_str}

    **Metric Definition Structure (for reference and generation):**
    {{
        "metric_name": "string (unique identifier, prefer snake_case, prefix generated metrics with 'gen_')",
        "chinese_name": "string",
        "english_name": "string",
        "description": "string",
        "english_description": "string",
        "unit": "string",
        "english_unit": "string",
        "calculation": "string (SQL snippet or pattern)",
        "calculation_level": "string (ROW | AGGREGATE | POST_AGGREGATE | WINDOW)",
        "default_aggregation": "string (SUM | AVG | COUNT | COUNT_DISTINCT | MAX | MIN | NONE)",
        "dependencies": ["list of metric_names"],
        "metric_type": "string (e.g., SUM, COUNT, AVERAGE, RATIO, GROWTH, RANK, CONDITIONAL_SUM, CONDITIONAL_COUNT_DISTINCT)",
        "result_data_type": "string (e.g., INTEGER, DECIMAL(p,s), PERCENTAGE, STRING)",
        "comment": "string (optional)"
    }}

    **Important Instructions (you MUST follow):**
    1.  Analyze Request: Carefully read the USER REQUEST (and conversation history) to understand the user's analytical goal, including desired measures, entities, dimensions, and filters.
    2.  Review Schema: Examine the TABLE SCHEMA to understand base fields, data types, and relationships.
    3.  Review Metrics & Prioritize English: Review the AVAILABLE PREDEFINED METRICS. Prioritize matching based on `metric_name`, `english_name`, and `english_description`. Refer to Chinese fields only if necessary for clarification.
    4.  Selection & Generation Logic:
        a.  Direct Match/Inference: Identify metrics from the AVAILABLE PREDEFINED METRICS list that directly address or can be reasonably inferred to address parts of the user's request.
        b.  Generate New Metric (If Needed): If the user's request requires a specific calculation or aggregation that is *not* available as a predefined metric but *can* be logically derived using TABLE SCHEMA fields or existing metrics: Create a *new* metric definition object following the structure. Use `gen_` prefix for `metric_name`. Fill metadata accurately. Define `calculation`, `calculation_level`, `dependencies`, `default_aggregation`.
        c.  Completeness: Ensure the combination of selected predefined metrics and newly generated metrics covers the *entirety* of the user's request.
    5.  Output Format: Return *only* a valid JSON object containing two keys: `"selected_metrics"` (array of strings - predefined `metric_name`s) and `"generated_metrics"` (array of JSON objects - *newly generated* metric definitions). If none needed, use empty arrays. Example: `{{"selected_metrics": ["sales_amount"], "generated_metrics": []}}`. Do NOT include explanations.

    **User Request (and conversation history):**
    {conversation_history_str}

    **Output (JSON Object with selected_metrics and generated_metrics):**

(3) Generate SQL

    You are a highly skilled SQL code generator specializing in Databricks SQL and Delta Lake.
    Your task is to generate a valid and efficient Databricks SQL query based on the provided USER REQUEST, TABLE SCHEMA, and RELEVANT METRIC DEFINITIONS. Prioritize querying only the SECOND LATEST time partition unless otherwise specified or implied.

    **Table Schema:**
    # Table Name: hive_metastore.ds_wccs_dmt.wccs_sales_analysis (Delta Lake, Partitioned by: year_month STRING 'YYYY-MM')
    {table_schema_str}
    # Instructions: Prioritize using 'english_description' and 'english_name' from the schema for understanding field context. Use 'field_name' for SQL column references. Note the table is partitioned by 'year_month'.

    **Relevant Metric Definitions (Selected Predefined & Newly Generated):**
    # These metrics have been identified or generated as relevant to the user request by a previous analysis step.
    # Use these definitions as the PRIMARY source for calculation logic.
    # Prioritize 'english_name' and 'english_description' for understanding metric meaning.
    {related_metrics_definitions_str}

    **User Request (and conversation history):**
    {conversation_history_str}

    **Important Instructions (you MUST follow these):**
    1.  Analyze User Intent: Based *primarily* on the USER REQUEST, identify the core analytical goal, the specific metrics the user wants to see, the dimensions for grouping (`GROUP BY`), any required filters (`WHERE` clause conditions), and the desired sorting order (`ORDER BY`).
    2.  Prioritize Second Latest Partition (Default Behavior): Analyze Time Context. If NO specific time range or multi-period trend is requested/implied, add a `WHERE` clause to filter ONLY the SECOND MOST RECENT `year_month` partition. If a specific time range/trend *is* requested/implied, apply *that* time filter instead.
    3.  Utilize Provided Metrics: Implement logic from Relevant Metric Definitions.
    4.  Prioritize English & Define Aliases: Interpret using English fields. Use metric's `english_name` strictly for concise, **snake_case** SQL aliases (e.g., "Average Sales per POC" -> `average_sales_per_poc`).
    5.  Implement Calculations Based on Definitions: Handle `calculation_level` ('ROW', 'AGGREGATE', 'POST_AGGREGATE', 'WINDOW') correctly, using CTEs for AGGREGATE/POST_AGGREGATE.
    6.  Apply Correct Aggregation: Use `default_aggregation` correctly based on grouping and metric type.
    7.  Construct Full SQL Query: Use `FROM hive_metastore.ds_wccs_dmt.wccs_sales_analysis`. Apply time filter (default or user-specified) and other user filters. Apply `GROUP BY` and `ORDER BY`. Filter Caution: Do NOT add filters based *solely* on schema comments unless explicitly requested or required by metric definition.
    8.  Ensure Calculation Safety: Implement `NULLIF(denominator, 0)` for ALL division operations.
    9.  Syntax and Dialect: Ensure valid **Databricks SQL**.
    10. Output: Return ONLY the generated SQL query as plain text. No markdown, comments, or explanations.

    **Generated Databricks SQL Query:**

With these steps, testing on our datasets and question sets passed successfully. And by checking the reasoning thought lenght and other over-thinking metrics, the LLM took less time and tokens to finish the job.

Further more, we also tried a less powerful LLM (non-reasoning) for step 2 and 3, it also worked well.

### Conclusion

From the above attempts, we could see the power of reducing Cognitive Workload in LLM-based data analysis agent design, I believe it's applicable for other fields as well.