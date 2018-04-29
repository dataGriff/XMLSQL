# XML in SQL 

## Overview
1. What is XML? What is its structure? What is well-formed XML?
1. Demonstrate some XQuery
1. Generate some test data for dogs entering kennels in a table
1. Show XML FOR functionality
1. Create XML schema for dogs entering kennels
1. Shows schema validation and fails if not correct schema
1. Import XML into SQL Table using MERGE
1. Bonus round! Example use of XML in SQL

## 0.0 Create Database and Schema 

Execute the below on your sql server to create the required database. 

```sql
CREATE DATABASE XMLDemo;
GO
CREATE SCHEMA import;
GO
```

#### Code to Clean Up Database when Needed 

```sql
/*
USE XMLDemo;
GO 
IF OBJECT_ID('import.DogKennelArrival') IS NOT NULL
DROP TABLE import.DogKennelArrival;
IF OBJECT_ID('import.DogKennelArrivalXML') IS NOT NULL
DROP TABLE import.DogKennelArrivalXML;
IF OBJECT_ID('import.XMLDogKennelArrival') IS NOT NULL
DROP XML SCHEMA COLLECTION import.XMLDogKennelArrival;
*/
```

## 1.0 What is XML? 
XML stands for Extensible Markup Language. It's intent is a universal messaging langugage to describe data across
multiple environments. It is hierarchical in nature with nodes represented as attributes or elements nested
within one another.  

Below you can see some self describing XML.

```sql
USE XMLDemo;
GO
DECLARE @x XML;
SET @x = '<?xml version = "1.0"?> 
			<!-- This is a comment node-->
			<RootElementNode attribute = "Root element attribute text">
				<ElementNode>Text node</ElementNode>
				<ElementNode>Text node 2
				<NestedNode>Nested Text Node</NestedNode>
				</ElementNode>
			</RootElementNode>'
SELECT @x;
GO
```
For XML to be well-formed it must have...
	1. One of more elements
	2. Only one root element
	3. Elements properly nested within one another and dont overlap 

```sql
USE XMLDemo;
GO 
DECLARE @xml XML =
N'<Hospital name = "University Hospital of Wales">
		<Patient sex="M" name="Bill Withers" age="40">
			<Diagnosis>Diabetes</Diagnosis>
			<AdmissionDate>01-10-2014</AdmissionDate>
		</Patient>
		<Patient sex="F" name="April O&apos;Neil" age="32">
			<Diagnosis>Broken Hip</Diagnosis>
			<AdmissionDate>15-12-2014</AdmissionDate>
		</Patient>
		<Patient sex="M" name="Bruce Wayne" age="38">
			<Diagnosis>Cholera</Diagnosis>
			<AdmissionDate>10-01-2015</AdmissionDate>
		</Patient>
	</Hospital>';
SELECT @xml;
GO 
```

Non-Well Formed XML must still follow these rules but it doesnt have to have one root node
Below is an XML fragment. 

```sql
USE XMLDemo;
GO 
DECLARE @xml XML =
		N'<Patient sex="M" name="Bill Withers" age="40">
			<Diagnosis>Diabetes</Diagnosis>
			<AdmissionDate>01-10-2014</AdmissionDate>
		</Patient>
		<Patient sex="F" name="April O&apos;Neil" age="32">
			<Diagnosis>Broken Hip</Diagnosis>
			<AdmissionDate>15-12-2014</AdmissionDate>
		</Patient>
		<Patient sex="M" name="Bruce Wayne" age="38">
			<Diagnosis>Cholera</Diagnosis>
			<AdmissionDate>10-01-2015</AdmissionDate>
		</Patient>';
SELECT @xml;
GO 
```

## 2.0 XQuery 

You can query XML directly using XQuery, there are four main functions:
1.  **Query()** Obtain sections of XML you require
1.  **Value()** Obtain single values from XML
1.  **Exist()** Determine whether value/element exists (1) or not (0)
1.  **Nodes()** Shred XML data into relational format

Execute all of the code below on the XMLDemo database and take a look at the outputs. 

```sql
USE XMLDemo;
GO
--2.1 Create an XML variable to Query 
DECLARE @xml XML =
N'<Kennel name = "Coedely">
		<Dog sex="M" name="Harvey" neutered="true">
			<Breed>German Shepherd</Breed>
			<KennelArrivalDate>01-10-2014</KennelArrivalDate>
		</Dog>
		<Dog sex="F" name="Lucy" neutered="true">
			<Breed>Labrador</Breed>
			<KennelArrivalDate>15-12-2014</KennelArrivalDate>
		</Dog>
		<Dog sex="M" name="Fudge" neutered="false">
			<Breed>American Bulldog</Breed>
			<KennelArrivalDate>10-01-2015</KennelArrivalDate>
		</Dog>
	</Kennel>';

SELECT @xml;

--2.1.1 Query(): Get dogs
SELECT [QueryDogs] =  @xml.query(N'/Kennel/Dog');
--2.1.2 Query(): Get dog breeds 
SELECT [QueryDogBreeds] =  @xml.query(N'/Kennel/Dog/Breed');
--2.1.3 Query(): Get dog named Harvey
SELECT [QueryDogNamedHarvey] =  @xml.query(N'/Kennel/Dog[@name="Harvey"]');
--2.2 Value(): Get Breed where dog named fudge
SELECT [ValueFudgeBreed] =  @xml.value(N'(/Kennel/Dog[@name="Fudge"]/Breed)[1]', N'nvarchar(100)');
--2.3 Exist():  Using Exist Function in CASE
SELECT [ExistLucy] =  CASE @xml.exist
(
N'/Kennel/Dog[@name="Lucy"]'
)
WHEN 1 THEN N'The kennel has a dog named Lucy'
WHEN 0 THEN N'The kennel does not have a dog named Lucy'
END;
--2.4 Nodes(); Shred XML Data into Relational Format
SELECT 
col.value(N'./@sex[1]',N'nvarchar(100)') AS Sex
,col.value(N'./@name[1]',N'nvarchar(100)') AS Name
,col.value(N'./@neutered[1]',N'nvarchar(100)') AS Neutered
,col.value(N'./Breed[1]',N'nvarchar(100)') AS Breed
FROM @xml.nodes(N'//Kennel/*') Tab(Col);

GO
```

## 3.0 Generate some test data for Dogs arriving in Kennels in relational format

Run the code below on your database so that we can use the data in later parts... 

```sql
USE XMLDemo;
GO
--DROP TABLE import.DogKennelArrival
--SELECT * FROM import.DogKennelArrival
CREATE TABLE import.DogKennelArrival
(
[id] INT IDENTITY PRIMARY KEY CLUSTERED
,[FirstInserted] DATETIME NOT NULL DEFAULT GETDATE()
,[LastUpdated] DATETIME NOT NULL DEFAULT GETDATE()
,[DogName] VARCHAR(25) NOT NULL CONSTRAINT UNQ_DogKennelArrival_DogName UNIQUE
,[DogSex] VARCHAR(8) NOT NULL
,[DogNeutered] BIT NOT NULL
,[DogBreed] VARCHAR(50) NOT NULL
,[KennelArrivalDate] DATE NOT NULL
)

INSERT INTO import.DogKennelArrival
(
[DogName],[DogSex] ,[DogNeutered],[DogBreed] ,[KennelArrivalDate] 
)
VALUES ('Harvey','M',1,'German Shepherd','01-Oct-2014')
, ('Lucy','F',1,'Labrador','15-Dec-2014')
, ('Fudge','M',0,'American Bulldog','10-Jan-2015')

GO
```

## 4.0 Show XML FOR Option Functionality

**4.1 XML RAW**
The default for XML RAW is to create a single outer node with the name of row and generate the XML in an attribute-centric format (4.1.1).
You can add the keyword Elements to XML RAW to generate the XML in an element-centric format (4.1.2).

Execute the below on the XMLDemo database to see examples of XML RAW. 

```sql
USE XMLDemo;
GO 
--4.1 XML Raw-----------------------------------------------------------
--4.1.1 XML Raw Default (attributes)
DECLARE @xml XML;
SET @xml = (SELECT 
[DogName],[DogSex] ,[DogNeutered],[DogBreed] ,[KennelArrivalDate] 
FROM import.DogKennelArrival
FOR XML RAW)
SELECT @xml AS 'XMLRawDefault';
GO

--4.1.2 XML Raw ElementsXML 
DECLARE @xml XML;
SET @xml = (SELECT 
[DogName],[DogSex] ,[DogNeutered],[DogBreed] ,[KennelArrivalDate] 
FROM import.DogKennelArrival
FOR XML RAW, ELEMENTS);
SELECT @xml AS 'XMLRawElements';
GO
```

**4.2 XML AUTO**
The default for XML AUTO is to create a single outer node that is the name of the table or alias used and the XML is in an attribute-centric format (4.2.1).
You can add the keyword Elements to XML AUTO to generate the XML in an element-centric format (4.2.2).
You can add aliases to the table name and columns so that these aliases represent the nodes in the XML.
The order of the columns is important in order to get appropriate hierarchical XML.
We will be looking at the XML auto capability to generate XML schema later...

Execute the below on the XMLDemo database to see examples of XML AUTO. 

```sql
USE XMLDemo;
GO 
--4.2 XML AUTO----------------------------------------------------------
--4.2.1 XML Auto Default (attributes)
DECLARE @xml XML;
SET @xml = (SELECT 
[DogName],[DogSex] ,[DogNeutered],[DogBreed] ,[KennelArrivalDate] 
FROM import.DogKennelArrival
FOR XML AUTO);
SELECT @xml AS 'XMLAutoDefault';
GO

--4.2.2 XML Auto Elements 
DECLARE @xml XML;
SET @xml = (SELECT 
[DogName],[DogSex] ,[DogNeutered],[DogBreed] ,[KennelArrivalDate] 
FROM import.DogKennelArrival
FOR XML AUTO, ELEMENTS);
SELECT @xml AS 'XMLAutoElements';
GO

--4.2.3 XML Auto with Alias 
DECLARE @xml XML;
SET @xml = (SELECT 
Dog.[DogName],Dog.[DogSex] ,Dog.[DogNeutered],Dog.[DogBreed] ,Dog.[KennelArrivalDate] 
FROM import.DogKennelArrival Dog
FOR XML AUTO, ELEMENTS);
SELECT @xml AS 'XMLAutoElementsAliases';
GO

```

**4.3 XML PATH**
This is the most customisable of the FOR clauses
The default is returning the XML in an element-centric format with the default "row" outer node (4.3.1).
You can provide the name of the outer node for each row and the root node (4.3.2).
You can force columns to become attributes using @ or cause customised nesting using "/" in the alias name (4.3.3).

Execute the below on the XMLDemo database to see examples of XML PATH. 

```sql
USE XMLDemo;
GO 
--4.3 XML Path-----------------------------------------------------------
--4.3.1 XML Path Default (elements)
DECLARE @xml XML;
SET @xml = (SELECT 
[DogName],[DogSex] ,[DogNeutered],[DogBreed] ,[KennelArrivalDate] 
FROM import.DogKennelArrival
FOR XML PATH);
SELECT @xml AS 'XMLPathDefault';
GO

--4.3.2 XML Path with Outer Node and Root Param (well-formed XML)
DECLARE @xml XML;
SET @xml = (SELECT 
[DogName],[DogSex] ,[DogNeutered],[DogBreed] ,[KennelArrivalDate] 
FROM import.DogKennelArrival
FOR XML PATH('Dog'), root('Kennel'));
SELECT @xml AS 'XMLPathParameters';
GO

--4.3.3 XML Path with Attributes and Path Specified
DECLARE @xml XML;
SET @xml = (SELECT 
[DogName] AS '@name',[DogSex] ,[DogNeutered],[DogBreed] 
,[KennelArrivalDate] AS 'ArrivalDetails/Date'
FROM import.DogKennelArrival
FOR XML PATH('Dog'), root('Kennel'));
SELECT @xml AS 'XMLPathParametersAndAliases';
GO
```

## 5.0 Create XML Schema using a Relational Table Structure

You can create an XML schema from a relational table (created in part 3) by capturing the schema and then capturing it when creating a schema collection.

Execute the below on the XMLDemo database to create an XML schema. 

```sql
USE XMLDemo;
GO 
--5.1 Declare schema variable
DECLARE @myschema NVARCHAR(MAX)
SET @myschema = N''
--5.2 Set Schema Variable using XML AUTO, ELEMENTS and XMLSCHEMA option
SET @myschema = (SELECT 
Dog.[DogName],Dog.[DogSex] ,Dog.[DogNeutered],Dog.[DogBreed] ,Dog.[KennelArrivalDate] 
FROM import.DogKennelArrival Dog
WHERE 1=0
FOR XML AUTO, ELEMENTS, XMLSCHEMA('DogNameSpace'));
--5.3 Look at schema
SELECT CAST(@myschema AS XML);
--5.4 Create Schema Collection
CREATE XML SCHEMA COLLECTION import.XMLDogKennelArrival AS @myschema;
--DROP XML SCHEMA COLLECTION import.XMLDogKennelArrival;
GO
```

## 6.0 Prove Create XML Schema Works

The first part of the code below code creates a table using the XML schema created in part 5.
It then inserts some XML data into the table column which fits the schema, and succeeds. 
It then inserts some XML data into the table column which doesn't fit the schema, and fails. 
In the second part of the code it attempts to declare an XML variable with the XML schema created in part 5. 
It then sets the variable to some XML that fits the schema, and succeeds. 
It then sets the variable to some XML that doesn't the schema, and fails. 

Execute the below on the XMLDemo database to see the validation of the XML schema in action. 

```sql
USE XMLDemo;
GO 
--6.1 Create Table
--DROP TABLE import.DogKennelArrivalXML;
--TRUNCATE TABLE import.DogKennelArrivalXML;
--SELECT * FROM import.DogKennelArrivalXML
CREATE TABLE import.DogKennelArrivalXML
(
[id] INT IDENTITY PRIMARY KEY CLUSTERED
,[FirstInserted] DATETIME NOT NULL DEFAULT GETDATE()
,[LastUpdated] DATETIME NOT NULL DEFAULT GETDATE()
,[XML] XML (import.XMLDogKennelArrival) --here is where we specify the XML schema
);
GO

--6.2 Insert Valid XML 
INSERT INTO import.DogKennelArrivalXML
(
[XML]
)

SELECT '<Dog xmlns="DogNameSpace">
  <DogName>Harvey</DogName>
  <DogSex>M</DogSex>
  <DogNeutered>1</DogNeutered>
  <DogBreed>German Shepherd</DogBreed>
  <KennelArrivalDate>2014-10-01</KennelArrivalDate>
</Dog>
<Dog xmlns="DogNameSpace">
  <DogName>Lucy</DogName>
  <DogSex>F</DogSex>
  <DogNeutered>1</DogNeutered>
  <DogBreed>Labrador</DogBreed>
  <KennelArrivalDate>2014-12-15</KennelArrivalDate>
</Dog>
<Dog xmlns="DogNameSpace">
  <DogName>Fudge</DogName>
  <DogSex>M</DogSex>
  <DogNeutered>1</DogNeutered>
  <DogBreed>American Bulldog</DogBreed>
  <KennelArrivalDate>2015-01-10</KennelArrivalDate>
</Dog>';
GO

--6.3 Insert Invalid XML (according to defined schema) = Fails
--Invalid because have extra node of DogColour not defined in the schema
INSERT INTO import.DogKennelArrivalXML
(
[XML]
)

SELECT '<Dog xmlns="DogNameSpace">
  <DogName>Harvey</DogName>
  <DogSex>M</DogSex>
  <DogColour>Brown</DogColour>
  <DogNeutered>1</DogNeutered>
  <DogBreed>German Shepherd</DogBreed>
  <KennelArrivalDate>2014-10-01</KennelArrivalDate>
</Dog>';
GO

--6.4 Declare Variable with Schema Validation for Valid Data = Works Fine
DECLARE @xml XML (import.XMLDogKennelArrival) = 
N'<Dog xmlns="DogNameSpace">
  <DogName>Harvey</DogName>
  <DogSex>M</DogSex>
  <DogNeutered>1</DogNeutered>
  <DogBreed>German Shepherd</DogBreed>
  <KennelArrivalDate>2014-10-01</KennelArrivalDate>
</Dog>
<Dog xmlns="DogNameSpace">
  <DogName>Lucy</DogName>
  <DogSex>F</DogSex>
  <DogNeutered>1</DogNeutered>
  <DogBreed>Labrador</DogBreed>
  <KennelArrivalDate>2014-12-15</KennelArrivalDate>
</Dog>
<Dog xmlns="DogNameSpace">
  <DogName>Fudge</DogName>
  <DogSex>M</DogSex>
  <DogNeutered>1</DogNeutered>
  <DogBreed>American Bulldog</DogBreed>
  <KennelArrivalDate>2015-01-10</KennelArrivalDate>
</Dog>'
SELECT @xml;
GO

--6.5 Declare Variable with Schema Validation for Invalid Data = Fails
--Invalid because have extra node of DogColour not defined in the schema
DECLARE @xml XML (import.XMLDogKennelArrival) = 
N'<Dog xmlns="DogNameSpace">
  <DogName>Harvey</DogName>
  <DogSex>M</DogSex>
  <DogColour>Brown</DogColour>
  <DogNeutered>1</DogNeutered>
  <DogBreed>German Shepherd</DogBreed>
  <KennelArrivalDate>2014-10-01</KennelArrivalDate>
</Dog>';
SELECT @xml;
GO
```

## 7.0 Import Data from XML Files into SQL Table 

The following code shows examples of inserts, updates and deletes from XML files (see the XMLImportFiles folder in repo) into SQL server. 
**IMPORTANT NOTE:** For this section to work you must have the XMLImportFiles folder locally and the filepath needs to be correct for the XMLImportFiles in the OPENROWSET bit!

Below demonstrates first insert of XML data from file system using CTE generated from nodes() and simple MERGE

```sql
USE XMLDemo;
GO 
TRUNCATE TABLE import.DogKennelArrival;
SELECT * FROM import.DogKennelArrival;
GO
-----------------------------------------------------------------------------
--7.1 INSERT XML DATA 
-----------------------------------------------------------------------------

--7.1.1 Set Date Format and Declare XML Variable
SET DATEFORMAT DMY;
DECLARE @xml XML;

--7.1.2 Use OPENROWSET to read XML file from filesystem 
SELECT @xml = BulkColumn
FROM OPENROWSET(BULK '\XMLImportFiles\XML_Dog_INSERT.xml'
, SINGLE_BLOB) TempXML;

--7.1.3 Show XML Variable
--SELECT @xml;

--7.1.4 Generate CTE of XML Data in Relational Format Using Nodes Function
;WITH CTE AS
(
SELECT 
col.value(N'./Sex[1]',N'nvarchar(100)') AS Sex
,col.value(N'./Name[1]',N'nvarchar(100)') AS Name
,col.value(N'./Neutered[1]',N'nvarchar(100)') AS Neutered
,col.value(N'./Breed[1]',N'nvarchar(100)') AS Breed
,col.value(N'./KennelArrivalDate[1]',N'nvarchar(100)') AS KennelArrivalDate
FROM @xml.nodes(N'//Kennel/*') Tab(Col)
)

--SELECT * FROM CTE 

--7.1.5 MERGE XML data from CTE into relational table
MERGE INTO import.DogKennelArrival AS Tgt
USING CTE AS Src ON
src.Name = tgt.DogName
WHEN NOT MATCHED THEN INSERT
(DogName, DogSex, DogNeutered, DogBreed, KennelArrivalDate)
VALUES (Name, Sex, Neutered, Breed, KennelArrivalDate)
WHEN MATCHED AND  
(
 tgt.DogSex <> src.Sex 
OR tgt.DogNeutered <>  src.Neutered 
OR  tgt.DogBreed <> src.Breed 
OR  tgt.KennelArrivalDate <> src.KennelArrivalDate 
) THEN UPDATE 
SET  tgt.DogSex = src.Sex 
, tgt.DogNeutered =  src.Neutered 
,  tgt.DogBreed = src.Breed 
,  tgt.KennelArrivalDate = src.KennelArrivalDate ;


--7.1.6 Show data successfully MERGED (INSERT) into SQL table
SELECT * FROM import.DogKennelArrival;

GO
```

Execute the below to demonstrate the update of XML data from file system using CTE generated from nodes() and simple MERGE.

```sql
USE XMLDemo;
GO 
-----------------------------------------------------------------------------
--7.2 UPDATE XML DATA 
-----------------------------------------------------------------------------

--7.2.1 Set Date Format and Declare XML Variable
SET DATEFORMAT DMY;
DECLARE @xml XML;

--7.2.2 Use OPENROWSET to read XML file from filesystem 
SELECT @xml = BulkColumn
FROM OPENROWSET(BULK '\XMLImportFiles\XML_Dog_UPDATE.xml'
, SINGLE_BLOB) TempXML;

--7.2.3 Show XML Variable
SELECT @xml;

--7.2.4 Generate CTE of XML Data in Relational Format Using Nodes Function
;WITH CTE AS
(
SELECT 
col.value(N'./Sex[1]',N'nvarchar(100)') AS Sex
,col.value(N'./Name[1]',N'nvarchar(100)') AS Name
,col.value(N'./Neutered[1]',N'nvarchar(100)') AS Neutered
,col.value(N'./Breed[1]',N'nvarchar(100)') AS Breed
,col.value(N'./KennelArrivalDate[1]',N'nvarchar(100)') AS KennelArrivalDate
FROM @xml.nodes(N'//Kennel/*') Tab(Col)
)

--SELECT * FROM CTE 

--7.2.5 MERGE XML data from CTE into relational table
MERGE INTO import.DogKennelArrival AS Tgt
USING CTE AS Src ON
src.Name = tgt.DogName
WHEN NOT MATCHED THEN INSERT
(DogName, DogSex, DogNeutered, DogBreed, KennelArrivalDate)
VALUES (Name, Sex, Neutered, Breed, KennelArrivalDate)
WHEN MATCHED AND  
(
 tgt.DogSex <> src.Sex 
OR tgt.DogNeutered <>  src.Neutered 
OR  tgt.DogBreed <> src.Breed 
OR  tgt.KennelArrivalDate <> src.KennelArrivalDate 
) THEN UPDATE 
SET  tgt.DogSex = src.Sex 
, tgt.DogNeutered =  src.Neutered 
,  tgt.DogBreed = src.Breed 
,  tgt.KennelArrivalDate = src.KennelArrivalDate ;

--7.2.6 Can see that UPDATES have taken place
--Fudge is now neutered and Lucy is in fact a Golden Retriever!
SELECT * FROM import.DogKennelArrival;

GO
```

Execute the below to demonstrate delete of XML data from file system using CTE generated from nodes() and simple MERGE.

```sql
USE XMLDemo;
GO 
-----------------------------------------------------------------------------
--7.3 DELETE XML DATA - (MERGE STATEMENT HAS EXTRA DELETE CRITERIA)
-----------------------------------------------------------------------------

--7.3.1 Set Date Format and Declare XML Variable
SET DATEFORMAT DMY;
DECLARE @xml XML;

--7.3.2 Use OPENROWSET to read XML file from filesystem 
SELECT @xml = BulkColumn
FROM OPENROWSET(BULK '\XMLImportFiles\XML_Dog_DELETE.xml'
, SINGLE_BLOB) TempXML;

--7.3.3 Show XML Variable
SELECT @xml;

--7.3.4 Generate CTE of XML Data in Relational Format Using Nodes Function
--Note: added delete column
;WITH CTE AS
(
SELECT 
col.value(N'./Sex[1]',N'nvarchar(100)') AS Sex
,col.value(N'./Name[1]',N'nvarchar(100)') AS Name
,col.value(N'./Neutered[1]',N'nvarchar(100)') AS Neutered
,col.value(N'./Breed[1]',N'nvarchar(100)') AS Breed
,col.value(N'./KennelArrivalDate[1]',N'nvarchar(100)') AS KennelArrivalDate
,col.value(N'./Delete[1]',N'nvarchar(100)') AS [Delete]
FROM @xml.nodes(N'//Kennel/*') Tab(Col)
)

--select * from cte;

--7.3.5 MERGE XML data from CTE into relational table
MERGE INTO import.DogKennelArrival AS Tgt
USING CTE AS Src ON
src.Name = tgt.DogName
WHEN NOT MATCHED THEN INSERT
(DogName, DogSex, DogNeutered, DogBreed, KennelArrivalDate)
VALUES (Name, Sex, Neutered, Breed, KennelArrivalDate)
WHEN MATCHED AND  
(
 tgt.DogSex <> src.Sex 
OR tgt.DogNeutered <>  src.Neutered 
OR  tgt.DogBreed <> src.Breed 
OR  tgt.KennelArrivalDate <> src.KennelArrivalDate 
) THEN UPDATE 
SET  tgt.DogSex = src.Sex 
, tgt.DogNeutered =  src.Neutered 
,  tgt.DogBreed = src.Breed 
,  tgt.KennelArrivalDate = src.KennelArrivalDate 
WHEN MATCHED AND [Delete] = 'true'
THEN DELETE;

--7.3.6 Harvey has been deleted from the Kennel (I've taken him home!)
SELECT * FROM import.DogKennelArrival;

GO
```

Execute the below to demonstrate insert of sparse XML data from file system using CTE generated from nodes() and simple MERGE.

```sql
USE XMLDemo;
GO 
-----------------------------------------------------------------------------
--6.4 INSERT SPARSE XML
-----------------------------------------------------------------------------

--6.4.1 Set Date Format and Declare XML Variable
SET DATEFORMAT DMY;
DECLARE @xml XML;

--6.4.2 Use OPENROWSET to read XML file from filesystem 
SELECT @xml = BulkColumn
FROM OPENROWSET(BULK '\XMLImportFiles\XML_Dog_SPARSE.xml', SINGLE_BLOB) TempXML;

--6.4.3 Show XML Variable
SELECT @xml;

--6.4.4 Generate CTE of XML Data in Relational Format Using Nodes Function
--Note: added delete column
;WITH CTE AS
(
SELECT 
COALESCE(col.value(N'./Sex[1]',N'nvarchar(100)'),'') AS Sex
,COALESCE(col.value(N'./Name[1]',N'nvarchar(100)'),'') AS Name
,COALESCE(col.value(N'./Neutered[1]',N'nvarchar(100)'),'') AS Neutered
,COALESCE(col.value(N'./Breed[1]',N'nvarchar(100)'),'') AS Breed
,COALESCE(col.value(N'./KennelArrivalDate[1]',N'nvarchar(100)'),'') AS KennelArrivalDate
,col.value(N'./Delete[1]',N'nvarchar(100)') AS [Delete]
FROM @xml.nodes(N'//Kennel/*') Tab(Col)
)

--SELECT * FROM CTE; 

--6.4.5 MERGE XML data from CTE into relational table
MERGE INTO import.DogKennelArrival AS Tgt
USING CTE AS Src ON
src.Name = tgt.DogName
WHEN NOT MATCHED THEN INSERT
(DogName, DogSex, DogNeutered, DogBreed, KennelArrivalDate)
VALUES (Name, Sex, Neutered, Breed, KennelArrivalDate)
WHEN MATCHED AND  
(
 tgt.DogSex <> src.Sex 
OR tgt.DogNeutered <>  src.Neutered 
OR  tgt.DogBreed <> src.Breed 
OR  tgt.KennelArrivalDate <> src.KennelArrivalDate 
) THEN UPDATE 
SET  tgt.DogSex = src.Sex 
, tgt.DogNeutered =  src.Neutered 
,  tgt.DogBreed = src.Breed 
,  tgt.KennelArrivalDate = src.KennelArrivalDate 
WHEN MATCHED AND [Delete] = 'true'
THEN DELETE;

SELECT * FROM import.DogKennelArrival;

GO
```

Execute the below to demonstrate the addition of a column to the XML data from file system using CTE generated from nodes() and simple MERGE.

```sql
USE XMLDemo;
GO 
-----------------------------------------------------------------------------
--6.5 Add Extra Field
-----------------------------------------------------------------------------

--6.5.1 Add Extra Column
ALTER TABLE import.DogKennelArrival
ADD DogColour VARCHAR(25) NOT NULL DEFAULT '';
GO

--6.5.1 Set Date Format and Declare XML Variable
SET DATEFORMAT DMY;
DECLARE @xml XML;

--6.5.2 Use OPENROWSET to read XML file from filesystem 
SELECT @xml = BulkColumn
FROM OPENROWSET(BULK '\XMLImportFiles\XML_Dog_ADDITEM.xml', SINGLE_BLOB) TempXML;

--6.4.3 Show XML Variable
SELECT @xml;

--6.4.4 Generate CTE of XML Data in Relational Format Using Nodes Function
--Note: added delete column
;WITH CTE AS
(
SELECT 
COALESCE(col.value(N'./Sex[1]',N'nvarchar(100)'),'') AS Sex
,COALESCE(col.value(N'./Name[1]',N'nvarchar(100)'),'') AS Name
,COALESCE(col.value(N'./Neutered[1]',N'nvarchar(100)'),'') AS Neutered
,COALESCE(col.value(N'./Breed[1]',N'nvarchar(100)'),'') AS Breed
,coalesce(col.value(N'./KennelArrivalDate[1]',N'nvarchar(100)'),'') AS KennelArrivalDate
,COALESCE(col.value(N'./Colour[1]',N'nvarchar(100)'),'') AS Colour
,col.value(N'./Delete[1]',N'nvarchar(100)') AS [Delete]
FROM @xml.nodes(N'//Kennel/*') Tab(Col)
)

--SELECT * FROM CTE; 

--6.4.5 MERGE XML data from CTE into relational table
MERGE INTO import.DogKennelArrival AS Tgt
USING CTE AS Src ON
src.Name = tgt.DogName
WHEN NOT MATCHED THEN INSERT
(DogName, DogSex, DogNeutered, DogBreed, KennelArrivalDate, DogColour)
VALUES (Name, Sex, Neutered, Breed, KennelArrivalDate, Colour)
WHEN MATCHED AND  
(
 tgt.DogSex <> src.Sex 
OR tgt.DogNeutered <>  src.Neutered 
OR  tgt.DogBreed <> src.Breed 
OR  tgt.KennelArrivalDate <> src.KennelArrivalDate 
OR  tgt.DogColour <> src.Colour 
) THEN UPDATE 
SET  tgt.DogSex = src.Sex 
, tgt.DogNeutered =  src.Neutered 
,  tgt.DogBreed = src.Breed 
,  tgt.KennelArrivalDate = src.KennelArrivalDate 
, tgt.DogColour = src.Colour 
WHEN MATCHED AND [Delete] = 'true'
THEN DELETE;

SELECT * FROM import.DogKennelArrival;

GO
```

## 8.0 Bonus Round! Turning rows into a csv group

This is really useful for multiple instances per event, e.g. diagnosis or procedures per an admission).

Execute the code below to see the row based data transform into a comma delimited list on a single row...

```sql
USE XMLDemo;
GO 
--8.1 Create small demo table of values
DECLARE @Table1 TABLE(AdmissionID INT, Diagnosis VARCHAR(100));
INSERT INTO @Table1 VALUES (1,'Heart Failure'),(1,'Brain Pain')
,(1,'Leg damage'),(1,'Knee twist'),(1,'Eye bulge')
,(2,'Cholera'),(3,'Tuberculosis'),(2,'Lurgy'),(3,'Spots');
SELECT * FROM @Table1;

--8.2 Us XML path and Stuff to convert into comma delimited string
SELECT  AdmissionID
       ,STUFF((SELECT ', ' + CAST(Diagnosis AS VARCHAR(10)) [text()]
         FROM @Table1 
         WHERE AdmissionID = t.AdmissionID
FOR XML PATH('')),1,1,'')
FROM @Table1 t
GROUP BY AdmissionID;
```

To demonstrate the speed of this, execute the following code, which first generates a million rows of data then performs the same logic as above!

```sql
--8.3.1 Create tally table to CROSS JOIN in part 2
DECLARE @rows INT = 1000000;

IF OBJECT_ID('tempdb..#tally') IS NOT NULL
	DROP TABLE #tally;

WITH cte as 
(
	SELECT 1 as n
	UNION ALL
	SELECT n +1
	FROM cte
	WHERE n < @rows
)
SELECT *
	INTO #tally 
FROM cte 
OPTION (MAXRECURSION 0);  
GO 

--8.3.2 Va va voom!
;WITH CTE AS
(
SELECT n AS AdmissionID, Diagnosis FROM #tally
CROSS JOIN  (SELECT 'Cough' AS Diagnosis UNION SELECT 'Sneeze' UNION SELECT 'Hiccups') AS b
)

SELECT  AdmissionID
       ,STUFF((SELECT ', ' + CAST(Diagnosis AS VARCHAR(10)) [text()]
         FROM Cte 
         WHERE AdmissionID = t.AdmissionID
FOR XML PATH('')),1,1,'')
FROM CTE t
GROUP BY t.AdmissionID;
GO

```



