# 💡 PowerBI-findings
Valuable DAX + Power Query annotations and snippets.

## 📆 Dynamic Date Table - DT (march 05, 2026)
Using PowerQuery , this code allows for dynamic generation of date columns, especially for a dedicated table apart from the facts or dimensions table.

- Code adapted from [Fernan's Reusable Dynamic Calendar tutorial](https://youtu.be/p0ZCev6u024?si=0VQpUn5Own6EwSu6).

> [!Note]
> First and foremost, disable the **auto time intelligence generation**, as it creates hidden tables that exponentially pile up and bloat file size. Aside from allowing custom intervals, dedicated date tables are paramount for greater scalability and control.
> 1. File;
> 2. Options and Settings;
> 3. Options;
> 4. Global;
> 5. Data Load;
> 6. Time Intelligence;
> 7. Disable _Auto date/time for new files_.

With that out of the way:
1. With an open file, Transform Data;
2. In Home tab, Advanced Editor
3. Name the query, and paste the code:
```
let
    StartDate = Date.AddYears(Date.From(DateTime.LocalNow()), -1),
    EndDate = Date.AddYears(Date.From(DateTime.LocalNow()), 1),
    NumberOfDays = Duration.Days(EndDate - StartDate),
    Dates = List.Dates(StartDate, NumberOfDays + 1, #duration(1,0,0,0)),
    #"Converted to Table" = Table.FromList(Dates, Splitter.SplitByNothing(), null, null, ExtraValues.Error),
    #"Changed Type" = Table.TransformColumnTypes(#"Converted to Table",{{"Column1", type date}}),
    #"Renamed Columns" = Table.RenameColumns(#"Changed Type", {"Column1", "Full Date"}),
in
    #"Renamed Columns"
```
- As it is, the query sets the start as the current date -1 year, and the end as current +1 year. The rows created in the ```Full Date``` column will serve as base for additional columns. As shown in Fernan's video, you can either type in the script for each new column, or:
1. **Add Column** tab
2. **Column From Examples**
3. Check the formats available and auto create as desired. The new columns will be inserted in the query, like so:
```
let
    StartDate = Date.AddYears(Date.From(DateTime.LocalNow()), -1),
    EndDate = Date.AddYears(Date.From(DateTime.LocalNow()), 1),
    NumberOfDays = Duration.Days(EndDate - StartDate),
    Dates = List.Dates(StartDate, NumberOfDays + 1, #duration(1,0,0,0)),
    #"Converted to Table" = Table.FromList(Dates, Splitter.SplitByNothing(), null, null, ExtraValues.Error),
    #"Changed Type" = Table.TransformColumnTypes(#"Converted to Table",{{"Column1", type date}}),
    #"Renamed Columns" = Table.RenameColumns(#"Changed Type", {"Column1", "Full Date"}),
    #"Inserted Day" = Table.AddColumn(#"Renamed Columns", "Day", each Date.Day([Full Date]), Int64.Type),
    #"Inserted Day Name" = Table.AddColumn(#"Inserted Day", "Day Name", each Date.DayOfWeekName([Full Date]), type text),
    #"Inserted Month" = Table.AddColumn(#"Inserted Day Name", "Month", each Date.Month([Full Date]), Int64.Type),
    #"Inserted Month Name" = Table.AddColumn(#"Inserted Month", "Month Name", each Date.MonthName([Full Date]), type text),
    #"Inserted Year" = Table.AddColumn(#"Inserted Month Name", "Year", each Date.Year([Full Date]), Int64.Type),
    #"Inserted Quarter" = Table.AddColumn(#"Inserted Year", "Quarter", each Date.QuarterOfYear([Full Date]), Int64.Type)
in
    #"Inserted Quarter"
```
- The ```StartDate``` and ```EndDate``` are always dynamic, since calculations are performed with ```DateTime.LocalNow()```, a **datetime** type (e.g. 2026-03-05 15:33:57), later converted into **date** (2026-03-05) by ```Date.From```; this renders the date usable by ```Date.AddYears``` or ```Date.AddMonths```. The number of years or months can be changed to alter all calculations across columns (using _-0_ or _0_ keeps the date unaltered).
- If you want a **fixed** custom start or end, here's an example:
```
let
    StartDate = #date(2024,1,1),
    EndDate = #date(2025,1,1),
...
```
- After finishing the query writing, apply changes to the model; a new table will appear, with the created columns ready to be used with Time Intelligence functions

> [!Tip]
> Instead of writing or copy/pasting the script on every model, the best use would be sharing it in a group via creation of on-line **dataflows** (Blank Query, pasting the script and saving); all group participants will have access to it, and may import the DT through the Get Data option in a model file.

### DT Update (march 09, 2026)

- In order to maintain a **one to many relationship** between the formatted date columns and dimension/main table, unique values must be extracted from the dedicated Date table, thereby creating several **fact tables**, one for each column that requires specific cardinality (cannot contain duplicates).
- In the following model, 3 tables were created from DT's columns; usually, the ```DISTINCT``` function should be enough to create a new column for new tables, (```Years = DISTINCT('Date'[Year])```, ```Month Name = DISTINCT('Date'[Month Name])```);

![](https://github.com/BrenoEnrico-MDSantos/PowerBI-findings/blob/main/Relationship%20Model.png)

>[!Note]
> The best method to order _Month Number_ and _Month Name_ is to pair all distinct values of text and number in the same table (2 columns of 12 rows), but ```DISTINCT``` could not work like that for _Month Number_, raising the error **"_A table of multiple values was supplied where a single value was expected._"** Hence, this solution of multiple tables for year numbers, month numbers and months names could be better.

- With cardinality and unicity in place, this "custom hierarchy" allows for filtering and regular operations you would expect from the DT; **number columns must obviosuly be formatted as integers, not text**, to then be sorted asc or desc.

![](https://github.com/BrenoEnrico-MDSantos/PowerBI-findings/blob/main/Year%20Slicer.png)
- So far, every new column adapted from the DT calls for an exclusive fact table, otherwise all values from the column will be considered unique, much like the main table's date column.

> [!Warning]
> **NEVER** use dimension table date fields as values. Use dedicated DT originated fact table columns with 1:* active relationship.
