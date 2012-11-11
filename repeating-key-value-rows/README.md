# Foreword
While working on this I received one crucial review point from Steve Hanson of the WMB development team. This was about how to get a key-value mapping represented one-to-one in the tree thus creating a name-value element whose name is determined by the key and the value as value. So if we have a file that looks like this: `foo "bar"`
our tree will look this (without us actually having modelled a element explicitly named foo):
```
MRM
|
+-- foo = "bar"
```
However both we both agreed on this being a bit much for this article and that this in itself does a good work at illustrating modelling and delimiter based parsing. So I kept this in its original format but aim at producing a second article on the technique suggested by Steve.

The **key point** being the end result in this article might not be very well suited for production usage because it's a quite verbose tree. If you have a real scenario whose format resembles the example used here your aim should **probably** be to create a simpler tree.
# IntroductionThis article takes a simple "by example" approach to message modelling within WebSphere Message Broker. It does so by showing, step by step, how to model and implement a message set within the TDS MRM domain for a format which consists of a repeating structure with variable amount of key-value pair rows.# Input## About
The input message consists of rows delimited by carriage return (_ASCII decimal 13, hexadecimal 0D_) and line feed (_ASCII decimal 10, hexadecimal 0A_). The data in each row within a structure is of the type key value where a single space (_ASCII 32, hexadecimal 20_) separates the key from the value. A structure starts with a three character tag (_TBL_) followed by the structure name, again with a single space between the tag and the name. A structure ends with a row with the tag _EOT_. In addition there's also a static header structure.## Example message
For all data samples in this article, carriage return and line feed are marked as \r and \n respectively
### Plain input message

   # FILE 0128_154659_ins.00175150.00233289
   
   ACTION "ins"
   TRANS_ID 314874
   
   XACTKEY 321321
   OPERATIONS 1
   WARNING ""
   OPERATION_TYPE "UPDATE"
   
   TBL Entities
   Entities_Id 1
   Entities_ShortName "FOO"
   Dataservers_Id 1
   Cities_Id 9621
   EOT
   
   TBL Users
   Users_Id 6541
   Users_ShortName "FB"
   UsersGrp_Id 81815
   Users_Name "FOO BAR"
   OriginalId 0
   EOT

### Input message with "visible" white space

   # FILE 0128_154659_ins.00175150.00233289\r\n
   \r\n
   ACTION "ins"\r\n
   TRANS_ID 314874\r\n
   \r\n
   XACTKEY 321321\r\n
   OPERATIONS 1\r\n
   WARNING ""\r\n
   OPERATION_TYPE "UPDATE"\r\n
   \r\n
   TBL Entities\r\n
   Entities_Id 1\r\n
   Entities_ShortName "FOO"\r\n
   Dataservers_Id 1\r\n
   Cities_Id 9621\r\n
   EOT\r\n
   \r\n
   TBL Users\r\n
   Users_Id 6541\r\n
   Users_ShortName "FB"\r\n
   UsersGrp_Id 81815\r\n
   Users_Name "FOO BAR"\r\n
   OriginalId 0\r\n
   EOT

## Example flow
A project interchange file for WMBv6.1 can be found in the project folder in this repo. It contains the message set, message flow and a test client configuration that works with the default configuration created by Message Broker Toolkit.

I configure my input node like this
* **Parse Timing** is set to **Complete**
* **Validate** is set to **Content and Value**
* **Failure Action** is set to **Exception** but you might want to try using Exception List that way you can catch multiple errors
# Deciding on how to model the message
## Desired logical tree
The first step in message modelling is to get an idea of how you want the logical tree to look like when the input data has been parsed. For this example we state that we want a tree that looks something like the tree below.

```
MRM
|
+--+ header
|  +-- file = # FILE 0128_154659_ins.00175150.00233289
|  +-- action = ACTION "ins"
|  +-- tx = TRANS_ID 314874
|  +-- key = XACTKEY 321321
|  +-- operations = OPERATIONS 1
|  +-- warning = WARNING ""
|  +-- type = OPERATION_TYPE "UPDATE"
|
+--+ tables
   |
   +--+ table
   |  +-- name = Entities
   |  +--+ column
   |  |  +-- name = Entities_Id
   |  |  +-- value = 1
   |  |
   |  +--+ column
   |  |  +-- name = Entities_ShortName
   |  |  +-- value = "FOO"
   |  |
   |  +--+ column
   |  |  +-- name = Dataservers_Id
   |  |  +-- value = 1
   |  |
   |  +--+ column
   |     +-- name = Cities_Id
   |     +-- value = 9621
   |
   +--+ table
      +-- name = Users
      +--+ column
      |  +-- name = Users_Id
      |  +-- value = 6541
      |
      +--+ column
      |  +-- name = Users_ShortName
      |  +-- value = "FB"
      |
      +--+ column
      |  +-- name = UsersGrp_Id
      |  +-- value = 81815
      |
      +--+ column
      |  +-- name = Users_Name
      |  +-- value = "FOO BAR"
      |
      +--+ column
         +-- name = OriginalId
         +-- value = 0
```
## About "empty rows"
The "newline rows" (i.e. the rows in the format that doesn't contain any "data") in this format posed a
couple questions on how to be handled at first. I started off with simply ignoring them and letting the
MRM do the same. This was achieved by simply not modelling any specific elements for these "rows". By
using the settings described in the WMB InfoCenter under the topic [TDS Null handling options (ad06830)](http://publib.boulder.ibm.com/infocenter/wmbhelp/v6r1m0/index.jsp?topic=%2Fcom.ibm.etools.mft.doc%2Fad06830_.htm) and specifically beneath the sub-heading _Handling missing fields in a delimited format_ we're able to simply suppress these "empty rows". While ignoring these rows might be fine when we received data in this format for processing within WMB it's **however not** a workable solution in the other scenario. These blank rows are actually part of the format and need to be represented in order for us to be able to construct new messages according to this format for propagation to other systems. To address this I first used specific _newline_ elements in the model in order to preserve these "empty rows". This is however a bit cumbersome and clutter the tree with elements to represent these lines isn't really that nice. This was quickly spotted by Steve Hanson while proof-reading this article and Steve then told me that these newlines could be consumed as group terminators. The construct on to which configure the group terminator could for example be a sequence which is the approach I've used in this example.
# Getting an idea of what the message set should look like
## Iteration one
With this input message it's obvious that the basis for our parsing is the newlines (\r\n). A good place to start would therefore be to simply getting the message split up in "rows", such as:

```
MRM
|
+-- row = # FILE 0128_154659_ins.00175150.00233289
+-- row = ACTION "ins"
+-- row = TRANS_ID 314874
+-- row = XACTKEY 321321
+-- row = OPERATIONS 1
+-- row = WARNING ""
+-- row = OPERATION_TYPE "UPDATE"
+-- row = TBL Entities
+-- row = Entities_Id 1
+-- row = Entities_ShortName "FOO"
+-- row = Dataservers_Id 1
+-- row = Cities_Id 9621
+-- row = EOT
+-- ...
```

Here the carriage return and line feed are "consumed" by being specified as the **delimiter for all elements**. The message set to achieve this tree is ridiculously simple. It merely consists of a complex type with one element. The important thing is to define the complex type as being "All Elements Delimited" and "Suppress Absent Element Delimiters" to "Never". The last setting is will tell the parser to continue past blank lines such as the ones between different TBL structures in our sample message. However please note that there are no "empty" row elements in our tree above that would represent these blank lines. This is fine for now but would be a problem in the final message set as discussed in the section above _About "empty rows"_. The element within our complex type of course needs to have "Max Occurs" set to -1 to allow any number of occurrences.
### 1.png
### 2.png
## Iteration two
The second thing I do is to fully model the one statically defined structure in our message, the header to give us a tree like this:

```
MRM
|
+--+ header
|  +-- file = # FILE 0128_154659_ins.00175150.00233289
|  +-- action = ACTION "ins"
|  +-- tx = TRANS_ID 314874
|  +-- key = XACTKEY 321321
|  +-- operations = OPERATIONS 1
|  +-- warning = WARNING ""
|  +-- type = OPERATION_TYPE "UPDATE"
|
+--+ rows
   +-- row = TBL Entities
   +-- row = Entities_Id 1
   +-- row = Entities_ShortName "FOO"
   +-- row = Dataservers_Id 1
   +-- row = Cities_Id 9621
   +-- row = EOT
   +-- ...
```
Note here how the "empty rows" within the header are being consumed by the group terminators configured on the sequences within the header type.
### 3.png
### 4.png
# Getting a handle on the TBL-structures
Now that we have the header sorted it's time for the juicy part of this message. Modelling the TBLstructures. The basis is that we can have any number of tables and that any given table can have any number of key-value column pairs. Tables are separated by an "empty row" and the message itself does not contain a trailing \r\n. As illustrated in the "Example section" section in the beginning of this article.

```
MRM
|
+--+ header
|  +-- file = # FILE 0128_154659_ins.00175150.00233289
|  +-- action = ACTION "ins"
|  +-- tx = TRANS_ID 314874
|  +-- key = XACTKEY 321321
|  +-- operations = OPERATIONS 1
|  +-- warning = WARNING ""
|  +-- type = OPERATION_TYPE "UPDATE"
|
+--+ tables
   |
   +--+ table
   |  +-- name = Entities
   |  +--+ column
   |  |  +-- name  = Entities_Id
   |  |  +-- value = 1
   |  |
   |  +--+ column
   |  |  +-- name  = Entities_ShortName
   |  |  +-- value = "FOO"
   |  |
   |  +--+ column
   |  |  +-- name  = Dataservers_Id
   |  |  +-- value = 1
   |  |
   |  +--+ column
   |     +-- name  = Cities_Id
   |     +-- value = 9621
   |
   ...
```
## Defining the structure
### 5.png
This is the outline of the complete structure. Fairly straightforward.
### 6.png
Occurrence is naturally configured on the repeating elements table and column.
## Configuring our message set
Simply defining the structure won't be enough. We must now define how WMB should determine where a tbl-structure starts, ends and how to split column names from their values. I've chosen to walk through the configuration "backwards" thus starting from the end of our structure with the _tColumn_ type and then moving "upwards" in the structure.
### 7.png
On the _tColumn_ type there's only significant configuration. The delimiter is set to space in order to split column names from their values.
### 8.png
On the _tTable_ type we're saying _All Elements Delimited_. Group indicator is "TBL<SP>" (<SP> in order to consumer the space character between TBL and the table name). Also we specify _Group Terminator_ to be "<CR><LF>EOT" to properly identify when a table ends. It's important to note that we don't specify "EOT<CR><LF>" since the last table in the message will not have a trailing \r\n.
### 9.png
Then on the _column_ element (type _tColumn_) we specify that the _Repeating Element Delimiter_ is "<CR><LF>"
### 10.png
When we then utilize the _tTable_ as the type for our _table_ we set the _Repeating Element Delimiter_ to "<CR><LF><CR><LF>". This will consume the \r\n trailing after the "EOT" used as _Group Terminator_ and the \r\n "empty row" that separates one table from another.
# The resulting parse tree
This is what the resulting parse tree looks like when we use parse the sample message used in this article together with the message set.