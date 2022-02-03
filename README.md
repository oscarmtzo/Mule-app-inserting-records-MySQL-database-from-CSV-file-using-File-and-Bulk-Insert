# Mule App for Inserting Records to MySQL Database from CSV file using *File* & *Bulk Insert* connectors  

- *File - Read* connector allow us to read files to retrieve data in order to process it, in this case the file is a CSV Separated valus by commas.
![](https://mulesy.com/wp-content/uploads/2020/09/word-image-98.png)*Mulesoft FTP Read connector*

![](https://programmerclick.com/images/31/8fb5d15cf1ac4eb73c98e0733f6c6987.png)*Sakila MySQL database.*
- *Sakila* database and *actor* table will contain the records to be added usin the FTP Read connector to read the CSV file `./sample-read-dir/actors.csv`, and the SQL Insert connector to insert new actors to DB.
- Actors to be inserted: 

|actorId| firstName| lastName| lastUpdate|
|---| ---| ---| ---|
|201| Sam| Jhonson| 2021-02-12 13:18:33 |
|202| MARY| ZELLWEGER| 2021-02-12 13:18:33 |
|203| CAMERON| Jhonson| 2021-02-12 13:18:33| 
|204| JULIA| Jhonson| 2021-02-12 13:18:33 |
|205| HUMPHREY| Jhonson| 2021-02-12 13:18:33| 
|206| WILL| Jhonson| 2021-02-12 13:18:33| 
|207| BEN| WILLIS| 2021-02-12 13:18:33 |
|208| CHRISTOPHER| WALKEN| 2021-02-12 13:18:33| 
|209| BELA| WEST| 2021-02-12 13:18:33 |
|210| DAN| STREEP| 2021-02-12 13:18:33 |

### Requirements previous to Inserting new records to database:
- Add, Create or Modify MySQL users' operations to grant access to ``insert`` data with the follwoqing SQL statement:
```sql
-- Creation of a test user to use at the configuration XML for Mule application before pushing to github
CREATE USER 'test'@'localhost' IDENTIFIED BY 'newpassword';
-- Grant access to our new user for all operations with "SELECT" & "INSERT"
GRANT SELECT ON * . * TO 'test'@'localhost';
GRANT INSERT ON * . * TO 'test'@'localhost';
``` 

## Mule connectors
This Mule application needs several connectors to develop an specific flow to get the requiered functionality, which are the following:
- *Scheduler*, is the Mule event trigger, is the connector that will execute for first instance the application every 1 minute.
*Inside XMl of Scheduler component:*
```xml
<scheduler doc:name="Scheduler" doc:id="5ded1c31-8e3a-424a-a9bf-05b23f0b01c7" >
    <scheduling-strategy >
        <fixed-frequency frequency="1" timeUnit="MINUTES"/>
    </scheduling-strategy>
</scheduler>
```
---
- *File - Read*, is a Mule event  processor that reads file content, it can be in several formats such as `.csv`, `.xlsx`, `.json`, etc; for this Mule app CSV will be used as the input data to be inserted to database.
```xml
<file:read doc:id="57f7e325-93ae-4ac3-b820-af569025ad6e" doc:name="Actors CSV File Read" path="./ftp-insert-sql-from-csv\sample-read-dir\actors.csv" config-ref="File_Config"/>
```
- `<file:read>` XMl tag attributes: 
    |path|config-ref|
    |--- |---|
    |set to `./ftp-insert-sql-from-csv\sample-read-dir\actors.csv` which is the source to be read|``File_Config` is a global element which has no configuration attributes to be set as shown here:|
    *`<file:config>` XML tag:*
    ```xml
    <file:config name="File_Config" doc:name="File Config" doc:id="b3d61ed7-b5de-4021-a9b7-d4f639f80bb3" />
    ```

----
- *Transform Message* connector will allow us to view the context or payload data, parse  to different data type format using DataWeave, view a preview of the result.
    - Adding an *Input CSV metadata type*, this serves as sample CSV file format to generate the context of the retrieved data from the previous file Read connector, it is available at `./dataType/csv-input-data.csv`, with the follwoing content:
    ```csv
    actorId, firstName, lastName, lastUpdate
    1, Carlos, Rdoriguez, 2017-12-03 12:03:33
    ```
    *Note: when defining the input metadata at Anypoint Studio, modify the first column for ID to be set as Number*
    |actorId|firstName|lastName|lastUpdate|
    |---|---|---|---|
    |Number|String|String|Date|
    - Adding an *Output JSON metadata type* `./dataType/json-output-data.json` similar to the columns found at the `sakila` database and `actor` table:
    ```json
    [
        {
            "actor_id": 1,
            "first_name": "Nick",
            "last_name": "Davis",
            "last_update": "2006-02-15 04:34:33"
        }
    ]
    ```
    *Value types taken by the output metadata type on Transform Message defined on JSON*
    |actor_id|first_name|last_name|last_update|
    |---|---|---|---|
    |Number|String|String|String|

    - Parsing data using DataWeave, dragging and dropping the columns from the input CSV to the output Json format generates the following Dataweave code:
    ```js
    %dw 2.0
    output application/json
    ---
    payload map ( payload01 , indexOfPayload01 ) -> {
        actor_id: payload01.actorId,
        first_name: payload01." firstName",
        last_name: payload01." lastName",
        last_update: payload01." lastUpdate" as String
    }
    ```
    *Output data is an Array of JSON objects:*
    ```json
    [
        ...,
        {
            "actor_id": "209",
            "first_name": " BELA",
            "last_name": " WEST",
            "last_update": " 2021-02-12 13:18:33 "
        },
        {
            "actor_id": "210",
            "first_name": " DAN",
            "last_name": " STREEP",
            "last_update": " 2021-02-12 13:18:33 "
        }
    ]
    ```
---- 
- *Database - Bulk Insert* connector allow us to insert one by one the payload objects that were inside the Array structure without using a *For Each* connector.
![](https://dz2cdn3.dzone.com/storage/article-thumb/10105198-thumb.jpg)*Bulk Insert allows easy insertion of multiple records of data.*

```xml
<db:bulk-insert doc:name="Bulk insert" doc:id="bab02094-8217-4392-abde-3ae4e56c3cc2" config-ref="Database_Config">
    <db:sql ><![CDATA[INSERT INTO actor(actor_id, first_name, last_name, last_update) 
VALUES (:actor_id, :first_name, :last_name, :last_update)]]></db:sql>
</db:bulk-insert>
```
- The 2 dot notation followed by a placeholder like this: `:actor_id`, is a sample for store data from the JSON format for every record, all this intended for the *Bulk Insert* connector to use it and write all new info to database.

*The result for this Bulk Insert Operation is 10 new records written to the database:*
```
-- Before Bulk Operation
200 row(s) returned

-- After Bulk Operation
210 row(s) returned
```
