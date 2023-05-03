A GTFS file is a ZIP archive containing a number of CSV files. A difference can concern a file, a column in a file or a row in a file.

A “GTFS Diff” file is a CSV file with 7 columns, allowing to express any difference found between two GTFS files. Each row describes a difference found.

## Columns description


|**Field Name**|**Type**|**Required**|**Description**|
| :- | :- | :- | :- |
|id|String|Required|Uniquely identifies a row in the GFTS Diff file.|
|file|String|Required|The name of the file in the GTFS archive concerned by the change|
|action|String. Enum: `add`,  `delete`, `update` |Required|The type of change. Has something been added, deleted or updated? Files and columns can be added or deleted. Only rows can be updated.|
|target|String. Enum: `file`, `column`, `row` |Required|Specify what is concerned by the “action”. Can be a file, a column in a file or just a row.|
|identifier|json|Required|<p>How to uniquely identify what part of the data is concerned by the change.</p><p>- if target is set to file, we identify the file using the “filename” key. For example {“filename”: “shapes.txt”}</p><p>- if target is set to column, we identify the column using the “column” key. For example {“column”: “bikes\_allowed”}</p><p>- if target is set to row, we identify the row using a list of keys. Each key being added to the other as a logical “AND”. For example, {"stop\_id":"A"} identifies a row having the “A” as a stop\_id. {“from\_stop\_id”: “A”, “to\_stop\_id”: “B”} identifies the row where “from\_stop\_id” = “A” AND “to\_stop\_id” = “B”</p>|
|initial\_value|json |Conditionally required|<p>Required in case of an updated row. List the initial values.</p><p>For example {"stop\_name": "Xrain station", “stop\_lat”: “”}</p>|
|new\_value|json|Conditionally required|<p>Required in case of an added or updated row.</p><p></p><p>A json where the keys are the column names and the values are the row values.</p><p></p><p>- For an added row, it contains all the column names and values. For example for a new transfer between stations: {“from\_stop\_id”: “A”, “to\_stop\_id”: “B”, “transfer\_type”: 1, “min\_transfer\_time”: 2}</p><p>- For an updated row, it contains only the modified values. For example {"stop\_name": "Train station", “stop\_lat”: “45.1”}</p>|
|note|String|Optional|A free field where explanations about the change can be given.|

### Ordering
The requested order concerns the “target” column. The differences should be listed in the following order:

1. “target” of type “file” come first. ie modifications on files in the archive
1. then “target” of type “column”, ie modifications on a column in a file
1. then “target” of type “row”, ie modifications on a row in a file

This order makes it easier for a human to grasp the differences between files, and for a computer to apply successive patches of changes (first create a file, then populate it).

For each given type of target, the row order is not specified.

### Example
Here is an example GTFS diff file, with some explanations in the note column about what each row means.

|id|file|action|target|identifier|initial\_value|new\_value|note|
| :- | :- | :- | :- | :- | :- | :- | :- |
|1|transfers.txt|add|file|{“filename”: “shapes.txt”}|||creation of new file|
|2|readme.pdf|delete|file|{“filename”: “readme.pdf”}|||deletion of a file|
|3|shapes.txt|delete|column|{“column”: “internal\_id”}|||delete the column “internal\_id” in the “transfers.txt” file|
|4|transfers.txt|add|row|||{“from\_stop\_id”: “A”, “to\_stop\_id”: “B”, “transfer\_type”: 1, “min\_transfer\_time”: 2}|add a row in the transfers.txt file|
|5|stops.txt|delete|row|{“stop\_id”: “A”}|{“stop\_id”: “A”, “stop\_name”: “town center”, …}||delete the row in stops.txt where “stop\_id” = “A”|
|6 |stops.txt|update|row|{“stop\_id”: “B”}|{“stop\_name”: “”}|{“stop\_name”: “station”}|<p>in stops.txt update the stop\_name of the row identified by “stop\_id” = “B”. The stop\_name was empty, now it is “station”</p><p></p>|
|7|calendar\_dates.txt|update|row|{“service\_id”: “1”, “date”: “*20220928”*}|{“exception\_type”: “1”}|{“exception\_type”: “2”}|in calendar\_dates.txt, update the exception\_type of the row identified by “service\_id” = “1” AND “date” = “*20220928”*. The exception\_type was 1, now it is 2.|

### Example
If you shuffle the rows of the stops.txt file in a GTFS archive, the resulting GTFS Diff is empty, as row order is not a relevant information in a GTFS file.

### Full example
The [examples](/examples/) folder contains simple GTFS files and the resulting GTFS Diff listing the differences between them.

## Possible usages
- Have a quick overview of the changes made to a GTFS file
- Communicate effectively to someone the changes made to a GTFS file and give an explanation for each change.
- Take two corrected GTFS files and merge them together.

## Possible alternatives we thought about
- Using text diff tools. CSV are just text files, so it is possible to use powerful existing tools to compare them. But if the text diff is easily made, the results are harder to interpret. For example if a column is deleted, on a 1000 rows file, text diff will show 1000 differences, whereas the current proposition will just list a single column deletion. Text diff is also order dependent, but GTFS files are not.
- On the complete opposite to the text diff is the use of a GTFS library to load the data in a model. Main advantage is the possibility to interpret the changes with more depth, because the model knows what it is talking about. Could make the difference between changes impacting routing calculations, visual elements (colors, etc). But makes it more difficult to handle wrong data (a pdf file in the archive has been deleted) and needs to constantly keep track of the GTFS extensions (fares V2, pathways, etc)
