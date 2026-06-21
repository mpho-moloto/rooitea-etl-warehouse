RooiTea ETL Data Warehouse
A Kimball-style dimensional data warehouse and SSIS ETL solution built as part of the Data Warehousing module (CSID6853) at the University of the Free State. The project models the operations of a fictional Rooibos tea distribution business covering farming, harvesting, processing, distribution, and sales. It loads three operational source systems and a flat file into a star schema using SQL Server Integration Services.

Important Note on Running This Project
This solution connects to a SQL Server instance hosted on the university's CSI server, which is only accessible from within that environment. The files in this repository are the complete SSIS solution as submitted for assessment. They are included as a transparent technical record of the work rather than a "clone and run" deliverable. The README and accompanying screenshots provide a self-contained explanation of the architecture and design decisions.

Project Demonstrations
Dimensional modelling using the Kimball methodology including star schema design, surrogate keys, and conformed dimensions

A three-package SSIS solution (MasterPackage, LoadDimensions, LoadFacts) with explicit control-flow orchestration and dependency sequencing

ETL from three separate operational source systems (farming, distribution, sales) plus a flat file for currency exchange rates

Slowly changing dimension concepts and a full date dimension with a South African April-March financial year calculation

Lookup-based surrogate key resolution with explicit error handling for unmatched rows

Fully idempotent design allowing the solution to be re-run any number of times without producing duplicate or inconsistent data

Architecture
Farming DB --------+
Distribution DB ---+---> LoadDimensions.dtsx ---> RooiTeaDW (dimension tables)
Sales DB ----------+                                    |
ExchangeRates.csv -+                                    v
                                                 LoadFacts.dtsx
                                             (DimBatch + 3 fact tables)
                                                      |
                                                      v
                                          RooiTeaDW (complete star schema)


MasterPackage.dtsx serves as the entry point. It contains no transformation logic of its own. Its sole purpose is to execute LoadDimensions.dtsx and, only upon successful completion, execute LoadFacts.dtsx. This ordering is enforced because fact tables contain foreign keys referencing dimension surrogate keys. A fact row cannot be inserted before the dimension row it references exists.

Dimensional Model
The warehouse models four hierarchies:

Geography: Farm to Region to Province to Country

Product: Product to Product Type to Product Category

Market: Customer to Market Segment to Market Type, and Distribution Centre to Destination Country to Market Type

Time: Day to Month to Quarter to Year to Financial Year (South African April-March fiscal calendar)

Three fact tables sit at the centre of the model:

FactProduction: Harvest and processing measures per batch

FactInventory: Stock levels and inventory value at a point in time

FactSales: Sales transactions with values converted between local and foreign currency using time-validity exchange rates

Key Engineering Decisions
DimBatch Location
Although DimBatch is technically a dimension table, it depends on DimFarm, DimDate, and DimQualityGrade, all of which load in LoadDimensions. Loading DimBatch in LoadFacts (which runs after LoadDimensions completes) guarantees these dependencies already exist.

DELETE vs TRUNCATE
TRUNCATE is blocked by SQL Server when a foreign key constraint references the table being truncated, even if the referencing table is empty. Since FactSales holds a foreign key to DimExchangeRate, truncating dimension tables while fact tables still hold data would fail. DELETE has no such restriction.

Identity Column Reseeding
DELETE does not reset SQL Server's identity counters. Without explicit DBCC CHECKIDENT reseeding, surrogate keys increment indefinitely across repeated runs. Reseeding keeps surrogate keys predictable and consistent on every run.

Lookup Cache Mode
For dimension tables of this size, loading the entire lookup table into memory once using Full Cache mode is faster than making a database round-trip for every incoming row.

Unmatched Lookup Rows
Rows that fail to find a matching dimension member are written to delimited error files rather than silently discarded or inserted with a null or default key. This surfaces data quality problems for investigation instead of hiding them within the warehouse.

Idempotency Achievement
Both LoadDimensions and LoadFacts begin with an Execute SQL Task that cleans up existing data (respecting foreign key dependency order) before loading new data. Running the full solution twice produces identical row counts both times, verified during development.

Technology Stack
SQL Server Integration Services (SSIS), SQL Server, Visual Studio 2022, T-SQL.

Author
Mpho Moloto
GitHub: https://github.com/mpho-moloto
LinkedIn: www.linkedin.com/in/mpho-moloto-0a996228b

