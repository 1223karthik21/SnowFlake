# SnowFlake

Every Snowflake account provides access to sample data sets. You can find corresponding schemas in SNOWFLAKE_SAMPLE_DATA database in your object explorer. For this guide we are going to use a subset of objects from TPC-H set, representing customers and their orders. We also going to take some reference data about nations and regions.

Let's start with the static reference data:
--------------------------------------------------------------------
-- setting up staging area
--------------------------------------------------------------------

USE SCHEMA l00_stg;

CREATE OR REPLACE TABLE stg_nation 
AS 
SELECT src.*
     , CURRENT_TIMESTAMP()          ldts 
     , 'Static Reference Data'      rscr 
  FROM snowflake_sample_data.tpch_sf10.nation src;

CREATE OR REPLACE TABLE stg_region
AS 
SELECT src.*
     , CURRENT_TIMESTAMP()          ldts 
     , 'Static Reference Data'      rscr 
  FROM snowflake_sample_data.tpch_sf10.region src;

![Screenshot (73)](https://github.com/1223karthik21/SnowFlake/assets/104613056/1271398b-72f3-44b1-b79b-5b900106bed0)

## Next, let's create staging tables for our data loading. This syntax should be very familiar with anyone working with databases before. It is ANSI SQL compliant DDL, with probably one key exception - for stg_customer we are going to load the full payload of JSON into raw_json column. For this, Snowflake has a special data type VARIANT.
As we load data we also going to add some technical metadata, like load data timestamp, row number in a file.

CREATE OR REPLACE TABLE stg_customer
(
  raw_json                VARIANT
, filename                STRING   NOT NULL
, file_row_seq            NUMBER   NOT NULL
, ldts                    STRING   NOT NULL
, rscr                    STRING   NOT NULL
);

CREATE OR REPLACE TABLE stg_orders
(
  o_orderkey              NUMBER
, o_custkey               NUMBER  
, o_orderstatus           STRING
, o_totalprice            NUMBER  
, o_orderdate             DATE
, o_orderpriority         STRING
, o_clerk                 STRING
, o_shippriority          NUMBER
, o_comment               STRING
, filename                STRING   NOT NULL
, file_row_seq            NUMBER   NOT NULL
, ldts                    STRING   NOT NULL
, rscr                    STRING   NOT NULL
);

![Screenshot (74)](https://github.com/1223karthik21/SnowFlake/assets/104613056/c23c4448-a340-4c8a-a542-ecd5b1976abe)

### Tables we just created are going to be used by Snowpipe to drip-feed the data as it is lands in the stage. In order to easily detect and incrementally process the new portion of data we are going to create streams on these staging tables:

CREATE OR REPLACE STREAM stg_customer_strm ON TABLE stg_customer;
CREATE OR REPLACE STREAM stg_orders_strm ON TABLE stg_orders;

## Next we are going to produce some sample data. And for the sake of simplicity we are going to take a bit of a shortcut here. 

CREATE OR REPLACE STAGE customer_data FILE_FORMAT = (TYPE = JSON);
CREATE OR REPLACE STAGE orders_data   FILE_FORMAT = (TYPE = CSV) ;

![Screenshot (75)](https://github.com/1223karthik21/SnowFlake/assets/104613056/6eeb51a6-9236-42a9-942b-be26cdf01925)

## Generate and unload sample data. There are couple of things going on. First, we are using object_construct as a quick way to create a object/document from all columns and subset of rows for customer data and offload it into customer_data stage.

COPY INTO @customer_data 
FROM
(SELECT object_construct(*)
  FROM snowflake_sample_data.tpch_sf10.customer limit 10
) 
INCLUDE_QUERY_ID=TRUE;

COPY INTO @orders_data 
FROM
(SELECT *
  FROM snowflake_sample_data.tpch_sf10.orders limit 1000
) 
INCLUDE_QUERY_ID=TRUE;

## You can now run the following to validate that the data is now stored in files:

list @customer_data;
SELECT METADATA$FILENAME,$1 FROM @customer_data; 

![Screenshot (76)](https://github.com/1223karthik21/SnowFlake/assets/104613056/12f60de8-b00c-4ab0-a423-b6a074fab052)

## Next, we are going to setup Snowpipe to load data from files in a stage into staging tables. 

CREATE OR REPLACE PIPE stg_orders_pp 
AS 
COPY INTO stg_orders 
FROM
(
SELECT $1,$2,$3,$4,$5,$6,$7,$8,$9 
     , metadata$filename
     , metadata$file_row_number
     , CURRENT_TIMESTAMP()
     , 'Orders System'
  FROM @orders_data
);

CREATE OR REPLACE PIPE stg_customer_pp 
--AUTO_INGEST = TRUE
--aws_sns_topic = 'arn:aws:sns:mybucketdetails'
AS 
COPY INTO stg_customer
FROM 
(
SELECT $1
     , metadata$filename
     , metadata$file_row_number
     , CURRENT_TIMESTAMP()
     , 'Customers System'
  FROM @customer_data
);

ALTER PIPE stg_customer_pp REFRESH;

ALTER PIPE stg_orders_pp   REFRESH;

![Screenshot (77)](https://github.com/1223karthik21/SnowFlake/assets/104613056/9278dc6d-15a3-4828-ab7c-1cc606c147c6)

## Once this done, you should be able to see data appearling in the target tables and the stream on these tables. As you would notice, number of rows in a stream is exactly the same as in the base table. This is because we didn't process/consumed the delta of that stream yet. Stay tuned!

SELECT 'stg_customer', count(1) FROM stg_customer
UNION ALL
SELECT 'stg_orders', count(1) FROM stg_orders
UNION ALL
SELECT 'stg_orders_strm', count(1) FROM stg_orders_strm
UNION ALL
SELECT 'stg_customer_strm', count(1) FROM stg_customer_strm
;
staged data

## Finally, now that we established the basics and new data is knocking at our door (stream), let's see how we can derive some of the business keys for the Data Vault entites we are going to model.

CREATE OR REPLACE VIEW stg_customer_strm_outbound AS 
SELECT src.*
     , raw_json:C_CUSTKEY::NUMBER           c_custkey
     , raw_json:C_NAME::STRING              c_name
     , raw_json:C_ADDRESS::STRING           c_address
     , raw_json:C_NATIONKEY::NUMBER         C_nationcode
     , raw_json:C_PHONE::STRING             c_phone
     , raw_json:C_ACCTBAL::NUMBER           c_acctbal
     , raw_json:C_MKTSEGMENT::STRING        c_mktsegment
     , raw_json:C_COMMENT::STRING           c_comment     
--------------------------------------------------------------------
-- derived business key
--------------------------------------------------------------------
     , SHA1_BINARY(UPPER(TRIM(c_custkey)))  sha1_hub_customer     
     , SHA1_BINARY(UPPER(ARRAY_TO_STRING(ARRAY_CONSTRUCT( 
                                              NVL(TRIM(c_name)       ,'-1')
                                            , NVL(TRIM(c_address)    ,'-1')              
                                            , NVL(TRIM(c_nationcode) ,'-1')                 
                                            , NVL(TRIM(c_phone)      ,'-1')            
                                            , NVL(TRIM(c_acctbal)    ,'-1')               
                                            , NVL(TRIM(c_mktsegment) ,'-1')                 
                                            , NVL(TRIM(c_comment)    ,'-1')               
                                            ), '^')))  AS customer_hash_diff
  FROM stg_customer_strm src
;

CREATE OR REPLACE VIEW stg_order_strm_outbound AS 
SELECT src.*
--------------------------------------------------------------------
-- derived business key
--------------------------------------------------------------------
     , SHA1_BINARY(UPPER(TRIM(o_orderkey)))             sha1_hub_order
     , SHA1_BINARY(UPPER(TRIM(o_custkey)))              sha1_hub_customer  
     , SHA1_BINARY(UPPER(ARRAY_TO_STRING(ARRAY_CONSTRUCT( NVL(TRIM(o_orderkey)       ,'-1')
                                                        , NVL(TRIM(o_custkey)        ,'-1')
                                                        ), '^')))  AS sha1_lnk_customer_order             
     , SHA1_BINARY(UPPER(ARRAY_TO_STRING(ARRAY_CONSTRUCT( NVL(TRIM(o_orderstatus)    , '-1')         
                                                        , NVL(TRIM(o_totalprice)     , '-1')        
                                                        , NVL(TRIM(o_orderdate)      , '-1')       
                                                        , NVL(TRIM(o_orderpriority)  , '-1')           
                                                        , NVL(TRIM(o_clerk)          , '-1')    
                                                        , NVL(TRIM(o_shippriority)   , '-1')          
                                                        , NVL(TRIM(o_comment)        , '-1')      
                                                        ), '^')))  AS order_hash_diff     
  FROM stg_orders_strm src
;

## Finally let's query these views to validate the results: 

![Screenshot (78)](https://github.com/1223karthik21/SnowFlake/assets/104613056/4885bbd2-48f3-422a-b0fa-d407d405c058)

## We'll start by deploying DDL for the HUBs, LINKs and SATellite tables.

--------------------------------------------------------------------
-- setting up RDV
--------------------------------------------------------------------

USE SCHEMA l10_rdv;

-- hubs

CREATE OR REPLACE TABLE hub_customer 
( 
  sha1_hub_customer       BINARY    NOT NULL   
, c_custkey               NUMBER    NOT NULL                                                                                 
, ldts                    TIMESTAMP NOT NULL
, rscr                    STRING    NOT NULL
, CONSTRAINT pk_hub_customer        PRIMARY KEY(sha1_hub_customer)
);                                     

CREATE OR REPLACE TABLE hub_order 
( 
  sha1_hub_order          BINARY    NOT NULL   
, o_orderkey              NUMBER    NOT NULL                                                                                 
, ldts                    TIMESTAMP NOT NULL
, rscr                    STRING    NOT NULL
, CONSTRAINT pk_hub_order           PRIMARY KEY(sha1_hub_order)
);                                     

-- sats

CREATE OR REPLACE TABLE sat_customer 
( 
  sha1_hub_customer      BINARY    NOT NULL   
, ldts                   TIMESTAMP NOT NULL
, c_name                 STRING
, c_address              STRING
, c_phone                STRING 
, c_acctbal              NUMBER
, c_mktsegment           STRING    
, c_comment              STRING
, nationcode             NUMBER
, hash_diff              BINARY    NOT NULL
, rscr                   STRING    NOT NULL  
, CONSTRAINT pk_sat_customer       PRIMARY KEY(sha1_hub_customer, ldts)
, CONSTRAINT fk_sat_customer       FOREIGN KEY(sha1_hub_customer) REFERENCES hub_customer
);                                     

CREATE OR REPLACE TABLE sat_order 
( 
  sha1_hub_order         BINARY    NOT NULL   
, ldts                   TIMESTAMP NOT NULL
, o_orderstatus          STRING   
, o_totalprice           NUMBER
, o_orderdate            DATE
, o_orderpriority        STRING
, o_clerk                STRING    
, o_shippriority         NUMBER
, o_comment              STRING
, hash_diff              BINARY    NOT NULL
, rscr                   STRING    NOT NULL   
, CONSTRAINT pk_sat_order PRIMARY KEY(sha1_hub_order, ldts)
, CONSTRAINT fk_sat_order FOREIGN KEY(sha1_hub_order) REFERENCES hub_order
);   

-- links

CREATE OR REPLACE TABLE lnk_customer_order
(
  sha1_lnk_customer_order BINARY     NOT NULL   
, sha1_hub_customer       BINARY 
, sha1_hub_order          BINARY 
, ldts                    TIMESTAMP  NOT NULL
, rscr                    STRING     NOT NULL  
, CONSTRAINT pk_lnk_customer_order  PRIMARY KEY(sha1_lnk_customer_order)
, CONSTRAINT fk1_lnk_customer_order FOREIGN KEY(sha1_hub_customer) REFERENCES hub_customer
, CONSTRAINT fk2_lnk_customer_order FOREIGN KEY(sha1_hub_order)    REFERENCES hub_order
);

-- ref data

CREATE OR REPLACE TABLE ref_region
( 
  regioncode            NUMBER 
, ldts                  TIMESTAMP
, rscr                  STRING NOT NULL
, r_name                STRING
, r_comment             STRING
, CONSTRAINT PK_REF_REGION PRIMARY KEY (REGIONCODE)                                                                             
)
AS 
SELECT r_regionkey
     , ldts
     , rscr
     , r_name
     , r_comment
  FROM l00_stg.stg_region;

CREATE OR REPLACE TABLE ref_nation 
( 
  nationcode            NUMBER 
, regioncode            NUMBER 
, ldts                  TIMESTAMP
, rscr                  STRING NOT NULL
, n_name                STRING
, n_comment             STRING
, CONSTRAINT pk_ref_nation PRIMARY KEY (nationcode)                                                                             
, CONSTRAINT fk_ref_region FOREIGN KEY (regioncode) REFERENCES ref_region(regioncode)  
)
AS 
SELECT n_nationkey
     , n_regionkey
     , ldts
     , rscr
     , n_name
     , n_comment
  FROM l00_stg.stg_nation;           

  ![Screenshot (84)](https://github.com/1223karthik21/SnowFlake/assets/104613056/59c987c6-4513-4c1f-85ff-6b38d600d420)

## Now we have source data waiting in our staging streams & views, we have target RDV tables. Let's connect the dots. We are going to create tasks, one per each stream so whenever there is new records coming in a stream,

CREATE OR REPLACE TASK customer_strm_tsk
  WAREHOUSE = dv_rdv_wh
  SCHEDULE = '1 minute'
WHEN
  SYSTEM$STREAM_HAS_DATA('L00_STG.STG_CUSTOMER_STRM')
AS 
INSERT ALL
WHEN (SELECT COUNT(1) FROM hub_customer tgt WHERE tgt.sha1_hub_customer = src_sha1_hub_customer) = 0
THEN INTO hub_customer  
( sha1_hub_customer
, c_custkey
, ldts
, rscr
)  
VALUES 
( src_sha1_hub_customer
, src_c_custkey
, src_ldts
, src_rscr
)  
WHEN (SELECT COUNT(1) FROM sat_customer tgt WHERE tgt.sha1_hub_customer = src_sha1_hub_customer AND tgt.hash_diff = src_customer_hash_diff) = 0
THEN INTO sat_customer  
(
  sha1_hub_customer  
, ldts              
, c_name            
, c_address         
, c_phone           
, c_acctbal         
, c_mktsegment      
, c_comment         
, nationcode        
, hash_diff         
, rscr              
)  
VALUES 
(
  src_sha1_hub_customer  
, src_ldts              
, src_c_name            
, src_c_address         
, src_c_phone           
, src_c_acctbal         
, src_c_mktsegment      
, src_c_comment         
, src_nationcode        
, src_customer_hash_diff         
, src_rscr              
)
SELECT sha1_hub_customer   src_sha1_hub_customer
     , c_custkey           src_c_custkey
     , c_name              src_c_name
     , c_address           src_c_address
     , c_nationcode        src_nationcode
     , c_phone             src_c_phone
     , c_acctbal           src_c_acctbal
     , c_mktsegment        src_c_mktsegment
     , c_comment           src_c_comment    
     , customer_hash_diff  src_customer_hash_diff
     , ldts                src_ldts
     , rscr                src_rscr
  FROM l00_stg.stg_customer_strm_outbound src
;


CREATE OR REPLACE TASK order_strm_tsk
  WAREHOUSE = dv_rdv_wh
  SCHEDULE = '1 minute'
WHEN
  SYSTEM$STREAM_HAS_DATA('L00_STG.STG_ORDERS_STRM')
AS 
INSERT ALL
WHEN (SELECT COUNT(1) FROM hub_order tgt WHERE tgt.sha1_hub_order = src_sha1_hub_order) = 0
THEN INTO hub_order  
( sha1_hub_order
, o_orderkey
, ldts
, rscr
)  
VALUES 
( src_sha1_hub_order
, src_o_orderkey
, src_ldts
, src_rscr
)  
WHEN (SELECT COUNT(1) FROM sat_order tgt WHERE tgt.sha1_hub_order = src_sha1_hub_order AND tgt.hash_diff = src_order_hash_diff) = 0
THEN INTO sat_order  
(
  sha1_hub_order  
, ldts              
, o_orderstatus  
, o_totalprice   
, o_orderdate    
, o_orderpriority
, o_clerk        
, o_shippriority 
, o_comment              
, hash_diff         
, rscr              
)  
VALUES 
(
  src_sha1_hub_order  
, src_ldts              
, src_o_orderstatus  
, src_o_totalprice   
, src_o_orderdate    
, src_o_orderpriority
, src_o_clerk        
, src_o_shippriority 
, src_o_comment      
, src_order_hash_diff         
, src_rscr              
)
WHEN (SELECT COUNT(1) FROM lnk_customer_order tgt WHERE tgt.sha1_lnk_customer_order = src_sha1_lnk_customer_order) = 0
THEN INTO lnk_customer_order  
(
  sha1_lnk_customer_order  
, sha1_hub_customer              
, sha1_hub_order  
, ldts
, rscr              
)  
VALUES 
(
  src_sha1_lnk_customer_order
, src_sha1_hub_customer
, src_sha1_hub_order  
, src_ldts              
, src_rscr              
)
SELECT sha1_hub_order          src_sha1_hub_order
     , sha1_lnk_customer_order src_sha1_lnk_customer_order
     , sha1_hub_customer       src_sha1_hub_customer
     , o_orderkey              src_o_orderkey
     , o_orderstatus           src_o_orderstatus  
     , o_totalprice            src_o_totalprice   
     , o_orderdate             src_o_orderdate    
     , o_orderpriority         src_o_orderpriority
     , o_clerk                 src_o_clerk        
     , o_shippriority          src_o_shippriority 
     , o_comment               src_o_comment      
     , order_hash_diff         src_order_hash_diff
     , ldts                    src_ldts
     , rscr                    src_rscr
  FROM l00_stg.stg_order_strm_outbound src;    

ALTER TASK customer_strm_tsk RESUME;  
ALTER TASK order_strm_tsk    RESUME;  
Once tasks are created and RESUMED (by default, they are initially suspended) let's have a look on the task execution history to see how the process will start.
SELECT *
  FROM table(information_schema.task_history())
  ORDER BY scheduled_time DESC;

## Notice how after successfull execution, next two tasks run were automatically SKIPPED as there were nothing in the stream and there nothing to do.

![Screenshot (83)](https://github.com/1223karthik21/SnowFlake/assets/104613056/015add41-af18-40ff-ba0f-10546b7feb2d)

## We can also check content and stats of the objects involved.

SELECT 'hub_customer', count(1) FROM hub_customer
UNION ALL
SELECT 'hub_order', count(1) FROM hub_order
UNION ALL
SELECT 'sat_customer', count(1) FROM sat_customer
UNION ALL
SELECT 'sat_order', count(1) FROM sat_order
UNION ALL
SELECT 'lnk_customer_order', count(1) FROM lnk_customer_order
UNION ALL
SELECT 'l00_stg.stg_customer_strm_outbound', count(1) FROM l00_stg.stg_customer_strm_outbound
UNION ALL
SELECT 'l00_stg.stg_order_strm_outbound', count(1) FROM l00_stg.stg_order_strm_outbound;

## Data is now automatically flowing through all the layers via asyncronous tasks. 

![Screenshot (85)](https://github.com/1223karthik21/SnowFlake/assets/104613056/c49a5abd-ae18-473f-a646-6d3a7a71ee41)

First things we would like to add to simplify working with satellites is creating views that shows latest version for each key.
--------------------------------------------------------------------
-- RDV curr views
--------------------------------------------------------------------

USE SCHEMA l10_rdv;

CREATE VIEW sat_customer_curr_vw
AS
SELECT  *
FROM    sat_customer
QUALIFY LEAD(ldts) OVER (PARTITION BY sha1_hub_customer ORDER BY ldts) IS NULL;

CREATE OR REPLACE VIEW sat_order_curr_vw
AS
SELECT  *
FROM    sat_order
QUALIFY LEAD(ldts) OVER (PARTITION BY sha1_hub_order ORDER BY ldts) IS NULL;

--------------------------------------------------------------------
-- BDV curr views
--------------------------------------------------------------------

USE SCHEMA l20_bdv;

CREATE VIEW sat_order_bv_curr_vw
AS
SELECT  *
FROM    sat_order_bv
QUALIFY LEAD(ldts) OVER (PARTITION BY sha1_hub_order ORDER BY ldts) IS NULL;

CREATE VIEW sat_customer_bv_curr_vw
AS
SELECT  *
FROM    sat_customer_bv
QUALIFY LEAD(ldts) OVER (PARTITION BY sha1_hub_customer ORDER BY ldts) IS NULL;
Let's create a simple dimensional structure. Again, we will keep it virtual(as views) to start with, but you already know that depending on access characteristics required any of these could be selectively materialized.
USE SCHEMA l30_id;

-- DIM TYPE 1
CREATE OR REPLACE VIEW dim1_customer 
AS 
SELECT hub.sha1_hub_customer                      AS dim_customer_key
     , sat.ldts                                   AS effective_dts
     , hub.c_custkey                              AS customer_id
     , sat.rscr                                   AS record_source
     , sat.*     
  FROM l10_rdv.hub_customer                       hub
     , l20_bdv.sat_customer_bv_curr_vw            sat
 WHERE hub.sha1_hub_customer                      = sat.sha1_hub_customer;

-- DIM TYPE 1
CREATE OR REPLACE VIEW dim1_order
AS 
SELECT hub.sha1_hub_order                         AS dim_order_key
     , sat.ldts                                   AS effective_dts
     , hub.o_orderkey                             AS order_id
     , sat.rscr                                   AS record_source
     , sat.*
  FROM l10_rdv.hub_order                          hub
     , l20_bdv.sat_order_bv_curr_vw               sat
 WHERE hub.sha1_hub_order                         = sat.sha1_hub_order;

-- FACT table

CREATE OR REPLACE VIEW fct_customer_order
AS
SELECT lnk.ldts                                   AS effective_dts     
     , lnk.rscr                                   AS record_source
     , lnk.sha1_hub_customer                      AS dim_customer_key
     , lnk.sha1_hub_order                         AS dim_order_key
-- this is a factless fact, but here you can add any measures, calculated or derived
  FROM l10_rdv.lnk_customer_order                 lnk;  
All good so far? Now lets try to query fct_customer_order and at least in my case this view was not returning any rows. Why? If you remember, when we were unloading sample data, we took a subset of random orders and a subset of random customers. Which in my case didn't have any overlap, therefore doing the inner join with dim1_order was resulting in all rows being eliminated from the resultset. Thankfully we are using Data Vault and all need to do is go and load full customer dataset. Just think about it, there is no need to reprocess any links or fact tables simply because customer/reference feed was incomplete. I am sure for those of you who were using different architectures for data engineering and warehousing have painful experience when such situations occur. So, lets go and fix it:
USE SCHEMA l00_stg;

COPY INTO @customer_data 
FROM
(SELECT object_construct(*)
  FROM snowflake_sample_data.tpch_sf10.customer
  -- removed LIMIT 10
) 
INCLUDE_QUERY_ID=TRUE;

ALTER PIPE stg_customer_pp   REFRESH;
All you need to do now is just wait a few seconds whilst our continious data pipeline will automatically propagate new customer data into Raw Data Vault. Quick check for the records count in customer dimension now shows that there are 1.5Mn records:

USE SCHEMA l30_id;

SELECT COUNT(1) FROM dim1_customer;


COUNT(1)
------
1,500,000
Finally lets wear user's hat and run a query to break down orders by nation,region and order_priority_bucket (all attributes we derived in Business Data Vault). As we are using Snowsight, why not quickly creating a chart from this result set to better understand the data. For this simply click on the ‘Chart' section on the bottom pane and put attributes/measures as it is snown on the screenshot below.
SELECT dc.nation_name
     , dc.region_name
     , do.order_priority_bucket
     , COUNT(1)                                   cnt_orders
  FROM fct_customer_order                         fct
     , dim1_customer                              dc
     , dim1_order                                 do
 WHERE fct.dim_customer_key                       = dc.dim_customer_key
   AND fct.dim_order_key                          = do.dim_order_key
GROUP BY 1,2,3;

![Screenshot (86)](https://github.com/1223karthik21/SnowFlake/assets/104613056/48b73745-5e9e-42dc-9148-007f4ba5e6de)
