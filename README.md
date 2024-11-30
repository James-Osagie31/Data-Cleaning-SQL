# Nashville Housing Data Transformation
## Table of Contents
- [Project Overview](#project-overview)
- [Data Source](#data-source)
- [Tools Used](#tools-used)
- [Data Transformation Steps](#data-transformation-steps)
  
## Project Overview
In this project, I leveraged Microsoft SQL Server to perform a series of data cleaning and transformation tasks on the NashvilleHousing data. This data was messy as it consisted of inconsistent date format, missing property address, and duplicate records. 

## Data Source
Nashville Housing Data: The primary data used for this project is the "NashvilleHousing.csv". The data was downloaded from Kaggle. 

## Tools Used
Microsoft SQL Server 

## Data Transformation Steps
The transformation process includes:
- standardizing date formats
- populating missing property address data
- breaking out addresses into individual components
- handling owner address parsing
- modifying the SoldAsVacant field
- removing duplicate rows.

### 1. Standardizing Date Format
This step aims to standardize the SaleDate column to a Date type without time information.

Step 1: Select SaleDate and convert it to a Date format.
```sql
select SaleDate, CONVERT(Date, SaleDate)
from [dbo].[NashvilleHousing];
```
Step 2: Add a new column SaleDateConverted of type Date to the table.
```sql
alter table NashvilleHousing
add SaleDateConverted Date;
```
Step 3: Updates the SaleDateConverted column with the standardized date value from the SaleDate column.
```sql
update NashvilleHousing
SET SaleDateConverted = CONVERT(Date, SaleDate);
```
### 2. Populating Missing Property Address Data
This section handles populating missing values in the PropertyAddress column by utilizing records with the same ParcelID but different UniqueID. It performs a self-join to copy values where missing.

Step 1: Find records where PropertyAddress is null.
```sql
select *
from NashVilleHousing 
where [PropertyAddress] is null
order by [ParcelID];
```
Step 2: Perform a self-join on the table to find matching ParcelID values and attempt to fill missing addresses with available ones from other records.
```sql
select a.[ParcelID], a.[PropertyAddress], a.[ParcelID], b.[PropertyAddress], ISNULL(a.[PropertyAddress],b.[PropertyAddress])
from NashVilleHousing a
join NashVilleHousing b
	on a.ParcelID = b.ParcelID
	AND a.UniqueID <> b.UniqueID
where a.PropertyAddress is null;
```
Step 3: Update the PropertyAddress in the (a) table with the available data.
```sql
update a
set PropertyAddress = ISNULL(a.[PropertyAddress],b.[PropertyAddress])
from NashVilleHousing a
join NashVilleHousing b
	on a.ParcelID = b.ParcelID
	AND a.UniqueID <> b.UniqueID
where a.PropertyAddress is null;
```
### 3. Breaking Out Property Address into Individual Columns
Here, the goal is to split the PropertyAddress into two separate fields: PropertyStreet and PropertyCity.

Step 1: Select the PropertyAddress and use the SUBSTRING and CHARINDEX functions to split it into street and city parts.
```sql
select SUBSTRING(PropertyAddress, 1, CHARINDEX(',',PropertyAddress)-1) as Address,
SUBSTRING(PropertyAddress, CHARINDEX(',', PropertyAddress)+1,LEN(PropertyAddress)) as address2
from NashvilleHousing;
```
Step 2: Add two new columns: PropertyStreet (varchar(225)) and PropertyCity (varchar(25)).
```sql
alter table NashvilleHousing
add PropertyStreet varchar(225);

alter table NashvilleHousing
add PropertyCity varchar(25);
```
Step 3: Update the PropertyStreet and PropertyCity columns with extracted values.
```sql
update NashvilleHousing
set PropertyStreet = SUBSTRING(PropertyAddress, 1, CHARINDEX(',',PropertyAddress)-1)

update NashvilleHousing
set PropertyCity = SUBSTRING(PropertyAddress, CHARINDEX(',', PropertyAddress)+1,LEN(PropertyAddress))
```
### 4. Parsing Owner Address
The OwnerAddress column is parsed into individual components: OwnerStreet, OwnerCity, and OwnerState. This is achieved using SQL string functions like PARSENAME and REPLACE.

Step 1: Select the OwnerAddress column and use PARSENAME with REPLACE to split the address into its parts.
```Sql
select
PARSENAME(REPLACE(OwnerAddress,',','.'), 3),
PARSENAME(REPLACE(OwnerAddress,',','.'), 2),
PARSENAME(REPLACE(OwnerAddress,',','.'), 1)
from NashvilleHousing;
```
Step 2: Add new columns: OwnerStreet, OwnerCity, and OwnerState for storing the parsed data.
```sql
alter table NashvilleHousing
add OwnerStreet varchar(150);

alter table NashvilleHousing
add OwnerCity varchar(25);

alter table NashvilleHousing
add OwnerState varchar(25);
```
Step 3: Update the table to populate the new columns with parsed address data.
```sql
update NashvilleHousing
set OwnerStreet = PARSENAME(REPLACE(OwnerAddress,',','.'), 3)

update NashvilleHousing
set OwnerCity = PARSENAME(REPLACE(OwnerAddress,',','.'), 2)
```
update NashvilleHousing
set OwnerState = PARSENAME(REPLACE(OwnerAddress,',','.'), 1)

### 5. Changing 'Y' and 'N' to 'Yes' and 'No' in SoldAsVacant
This section standardizes the SoldAsVacant column by converting 'Y' and 'N' into 'Yes' and 'No' respectively.

Step 1: Select distinct values from the SoldAsVacant column.
```sql
select distinct [SoldAsVacant]
from NashvilleHousing
```
Step 2: Use a CASE statement to update the SoldAsVacant column with 'Yes' or 'No'.
```sql
select case
			when [SoldAsVacant]='Y' then 'Yes'
			when [SoldAsVacant]='N' then 'No'
			else [SoldAsVacant]
		end as SoldAsVacant
from NashvilleHousing
```
Step 3: Update the SoldAsVacant column with these changes.
```sql
update NashvilleHousing
set SoldAsVacant = case
			when [SoldAsVacant]='Y' then 'Yes'
			when [SoldAsVacant]='N' then 'No'
			else [SoldAsVacant]
		end
  ```
### 6. Removing Duplicate Records
This section removes duplicates from the table based on several key columns: ParcelID, PropertyAddress, SaleDate, SalePrice, and LegalReference.

Step 1: Create a Common Table Expression (CTE) to partition the table by the specified columns and assign a row number to each partition.
Step 2: Select records where the row number is greater than 1, indicating duplicates.
```sql
with cte as
(select *,
ROW_NUMBER() over(partition by 
							[ParcelID],
							[PropertyAddress],
							[SaleDate],[SalePrice],
							[LegalReference] 
					order by [UniqueID ]) as row_num
from NashvilleHousing
)

select * from cte 
where row_num >1 -- Select records where the row number is greater than 1, indicating duplicates.
```
#### (Optional, not included in the script) You can delete the duplicates based on the row number using a DELETE statement if necessary.

### References 
1. Kaggle (https://www.kaggle.com/datasets)

