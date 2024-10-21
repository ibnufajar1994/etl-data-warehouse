# **Build ETL Data pipeline Using Luigi**


This project is the next project of building the datawarehouse model using kimball approach, you can access the previous project on [here.](https://medium.com/@ibnufajar_1994/building-robust-data-warehouses-using-the-kimball-approach-fa95481018f3)

This project focus on **build automatic data pipeline** using luigi as data orchestration tool.

## **1. Requirement Gathering**
  
  In this step, we create a scenario where we have a meeting with a stakeholder and ask them 5 important questions based on the dataset and business needs. This information will guide us in choosing the right SCD strategy to be implemented in the data warehouse. 

  **Question 1**: Which entity in your system do you think is the most likely to change frequently among the others?

  **Possible Answer:** From our perspective, the Orders entity is the one that changes the most frequently. The reason for this is that orders go through multiple stages after they’re placed. For instance:
  - An order starts with a pending status when a customer makes the purchase.
  - Once the payment is confirmed, the status moves to approved. As we process the order, the status changes to shipped, and finally, it updates to delivered when the customer receives their package.
  Additionally, each of these status changes is tied to different timestamps. We track when the order is placed, approved, shipped, and delivered.

  **Question 2:** When an order status changes, do you also need to track how long each stage (e.g., pending, approved, shipped) takes? Should we store the duration between each status change?

  **Possible Answer:** Yes, Tracking how long an order stays in each status is very important for us. For example, we want to know the average time between when an order is created and when it gets approved, and   how long it takes to ship and deliver the order. This helps us improve efficiency and identify potential bottlenecks. We’d like to see these transitions and analyze time intervals.

  **Question 3:** For customer reviews, would you like to track historical changes in reviews or review scores, if ever updated?

  **Possible Answer:** Reviews are typically not changed once submitted, but there could be occasional updates. It’s not a priority to track review changes historically, but if changes do happen, we would prefer to keep only the latest version of the review.

  **Question 4:** How often do your product attributes change, such as product descriptions, weight, or dimensions?

   **Possible Answer:** Product attributes like descriptions, weight, and dimensions rarely change. However, there are occasional updates when sellers refine their listings or update information. We would want the ability to track those changes for analysis purposes, but retaining historical data is more important for certain attributes like product dimensions and weights.

   **Question 5:** Do you need to track changes to customer information, such as their address or zip code?

   **Possible Answer:** Yes, tracking changes to customer information, such as address or zip code, is important for delivery analytics and customer behavior analysis. If a customer moves, it would be helpful to see their new location for future orders while still keeping a history of their previous location for past transactions.

   **Question 6:** How often do sellers update their information, such as zip codes or business addresses? Should we keep a history of these changes?

   **Possible Answer:** Sellers may update their zip codes or business addresses when they expand to new locations or move their operations. Yes, we need to retain a history of these changes to analyze seller performance and order fulfillment across different regions over time.

   ## **2. Determine the SCD Type of Data Warehouse**

  After all the information gathered from the stakeholder, we can conclude that the **SCD Type 2** is suitable for our datawarehouse. Key Reasons for SCD Type 2 Implementation:
  
  **- Tracking Historical Changes:**
  The stakeholders emphasized the need to track changes to the order status over time, including timestamps for each status change (e.g., from pending to approved to delivered). This requires keeping a full history of the changes to analyze the order lifecycle effectively.

**- Capturing Changes in Estimated Delivery Dates:**
Since delivery dates may change (e.g., if there are delays), it is essential to maintain a history of the original estimated delivery date and any subsequent updates. This requires the ability to keep multiple records with their respective valid timeframes, which is characteristic of SCD Type 2.

**- Handling Cancellations and Returns:**
The stakeholders mentioned the importance of retaining information on order cancellations or returns, along with the timestamps and reasons for these events. SCD Type 2 allows for tracking these events without losing the previous status of the order.

**- Time-based Metrics:**
To calculate metrics such as time between order creation and approval or average delivery times, the data warehouse needs to store the historical states of the order as it moves through different stages. SCD Type 2 supports the analysis of these time-based metrics by keeping a record of each stage of the process.

**Why Not SCD Type 1 or SCD Type 3?**
- SCD Type 1 (Overwrite): This strategy simply overwrites old data with new data, which would result in losing valuable historical information. Since the stakeholders need to track changes over time (e.g., order status, delivery date changes), SCD Type 1 would not be suitable.

- SCD Type 3 (Limited History): This strategy keeps both the current and the previous state, but it does not store a full history of changes. For example, if an order goes through multiple status changes, SCD Type 3 would only store the most recent and the previous change, but not the complete history. This is inadequate for Olist’s needs, as they want to retain full details about the entire lifecycle of an order.

# SCD Type 2 Implementation Table

| **Table Name**             | **Implement SCD Type 2?** | **Reason**                                                                                                          |
|----------------------------|---------------------------|---------------------------------------------------------------------------------------------------------------------|
| **dim_product**             | **No**                    | Product details like dimensions and weight rarely change. Changes can be overwritten (SCD Type 1). Historical tracking is not crucial. |
| **dim_customer**            | **Yes**                   | Customer addresses, zip codes, and cities may change over time. It is important to keep historical records for analysis of customer behavior over different locations. |
| **dim_seller**              | **Yes**                   | Seller locations (city, state) and other business details can change if the business relocates. Keeping a history of these changes is essential for performance analysis across regions. |
| **dim_geolocation**         | **No**                    | Geolocation data (latitude, longitude) is typically static. Changes to geolocations are rare and can be handled with SCD Type 1 (overwrite). |
| **dim_payment_type**        | **No**                    | Payment types (e.g., credit card, debit) are unlikely to change for a specific transaction. Overwriting is sufficient. |
| **dim_order**               | **Yes**                   | Order status and timestamps (purchase, approval, delivery) change as orders progress through different stages. It’s critical to keep a full history to track the lifecycle of each order. |
| **dim_review**              | **No**                    | Customer reviews rarely change once submitted. Any updates can be overwritten with the latest information, so SCD Type 1 is sufficient. |
| **fact_order_processing**   | **Yes**                   | Order processing metrics (e.g., order status, delivery estimates) change frequently. Tracking the history of each change allows detailed analysis of the order lifecycle and performance over time. |
| **fact_payment_processing** | **No**                    | Payments are generally final once processed. There is no need to track historical changes to the payment type or value. |
| **fact_customer_review**    | **No**                    | Customer reviews are typically not updated after submission. In the rare case of updates, overwriting the review with the latest data is sufficient. |
| **fact_delivery_performance**| **Yes**                   | Delivery performance metrics (e.g., estimated delivery vs. actual delivery) may change over time due to delays. Keeping a history of changes to delivery estimates is essential for performance analysis. |

Since we implemented the SCD type 2 to the data warehouse, there is some addition on our table schema, these columns are:
- valid_from: A DATE or TIMESTAMP column that records the date when the record became valid.
- valid_to: A DATE or TIMESTAMP column that indicates when the record was superseded (or NULL if it is the current version).
- is_current: A BOOLEAN flag to mark the latest version of the record (optional, but useful for querying the current state quickly).
- 
With SCD Type 2, every time a change occurs (e.g., a customer moves to a new location), a new record will be inserted into the dimension table, representing the updated information. The old record will remain in the table, but the valid_to date will be set to indicate the time it became inactive, and the is_current flag (if implemented) will be set to FALSE for this outdated record. The fact tables (e.g., fact_order_processing, fact_delivery_performance) will not need to change in terms of structure. They will continue to reference the dimension tables as foreign keys. However, because SCD Type 2 involves inserting new rows into the dimension tables, fact tables will implicitly reference the correct version of a dimension record based on the foreign key relationship and the time when the fact occurred.
Here is the table that provides information about which tables were changed due to the implementation of SCD Type 2 and details the changes made to each table:
# SCD Type 2 Changes Table

| **Table Name**             | **Changes Due to SCD Type 2**                                                                                               |
|----------------------------|----------------------------------------------------------------------------------------------------------------------------|
| **dim_product**             | No changes. SCD Type 1 is implemented (no need for historical tracking).                                                    |
| **dim_customer**            | Added columns: `valid_from`, `valid_to`, `is_current` to track historical changes (SCD Type 2).                             |
| **dim_seller**              | Added columns: `valid_from`, `valid_to`, `is_current` to track historical changes (SCD Type 2).                             |
| **dim_geolocation**         | No changes. SCD Type 1 is implemented (no need for historical tracking).                                                    |
| **dim_payment_type**        | No changes. SCD Type 1 is implemented (no need for historical tracking).                                                    |
| **dim_order**               | Added columns: `valid_from`, `valid_to`, `is_current` to track historical changes (SCD Type 2).                             |
| **dim_review**              | No changes. SCD Type 1 is implemented (no need for historical tracking).                                                    |
| **fact_order_processing**   | No changes. This is a fact table and references SCD Type 2 dimensions where applicable.                                      |
| **fact_payment_processing** | No changes. This is a fact table and references SCD Type 2 dimensions where applicable.                                      |
| **fact_customer_review**    | No changes. This is a fact table and references SCD Type 2 dimensions where applicable.                                      |
| **fact_delivery_performance**| No changes. This is a fact table and references SCD Type 2 dimensions where applicable.                                      |


# **BUILD ELT PIPELINE USING LUIGI**

We have design the concept of datawarehouse,now the next step is to implemented the design and build the ELT pipeline. In this project we will use LUIGI to orchestrate
automatic data pipeline, the schema of ELT process is shown bellow:

![ELT SCHEMA](https://github.com/user-attachments/assets/53c1e259-8908-4df9-96fe-fc9acefee610)


   

    
