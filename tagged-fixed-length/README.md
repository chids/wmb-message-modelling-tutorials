# Introduction
This article takes simple by example approach to message modelling within WebSphere Message Broker. It does so by showing, step by step, how to model and implement a message set for tagged fixed length data within the TDS MRM domain.
## Input
### About
The input message is a consists of rows delimited by carriage return (_ASCII decimal 13, hexadecimal 0D_) and line feed (_ASCII decimal 10, hexadecimal 0A_). The data on each row is fixed length and each row starts with a two character tag that identifies whether the row is a header (10) or, an optional, related data (11) row. One logical record is therefore always a header row which sometimes also has a data row.
### Example message
For all data samples in this article, carriage return and line feed are marked as \r and \n respectively.
#### Raw input message
`10AABB\r\n10CCDD\r\n11EEFF\r\n10GGHH\r\n11XXYY\r\n10KKLL\r\n`
#### Input message formatted for readability
```
10AABB\r\n
10CCDD\r\n
11EEFF\r\n
10GGHH\r\n
11XXYY\r\n
10KKLL\r\n
```
### Example flow
A project interchange file for WMBv6.1 can be found in the _project_ directory in this repository. It contains the message set, message flow and a test client configuration that works with the default configuration created by Message Broker Toolkit.

# Deciding on how to model the message

## Desired logical tree
The first step in message modelling is to get an idea of how you want the logical tree to look like when the input data has been parsed. For this example we state that we want a tree that looks something like the tree below. Note that leading tags as well as trailing carriage returns and new lines have disappeared from the tree. The tags are used to determine whether a given row is of the type Header or Optional and the trailing carriage return and newline are used to separate records from each other.
```
MRM
|
+-+ Data
| +-+ Header
| +- First = AA
| +- Second = BB
|
+-+ Data
| +-+ Header
| | +- First = CC
| | +- Second = DD
| |
| +-+ Optional
| +- First = EE
| +- Second = FF
|
+-+ Data
| +- Header
| | +- First = GG
| | +- Second = HH
| |
| +- Optional
| +- First = XX
| +- Second = YY
|
+-+ Data
+-+ Header
+- First = KK
+- Second = LL
```
## Getting an idea of what the message set should look like
To allow us to get an initial idea of how the message set should be modelled in order to provide us with the logical tree described above I usually take the sample data at hand an look at it in different ways. It usually ends with me re-formatting it to better reflect my view of the data.

One such thing that I think is obvious in the input message is that an optional row, tagged 11, always belongs to a header row, tagged 10. So we could say that a logical data record consists of one _Header_ section and optionally one _Optional_ section. Therefore I would start of with re-formatting the input message to reflect that view. Note that we _do not manipulate_ the data in any way - it's simply formatted in a different way.
### Input message
```
10AABB\r\n
10CCDD\r\n
11EEFF\r\n
10GGHH\r\n
11XXYY\r\n
10KKLL\r\n
```
### Input message re-formatted to reflect our view of the data
```
10AABB\r\n
10CCDD\r\n11EEFF\r\n
10GGHH\r\n11XXYY\r\n
10KKLL\r\n
```
This work with reformatting and viewing data in different ways tends to help me get a better idea of how the message set should be configure in order to allow the MRM to parse the data and give me the logical tree that I want.
### Iteration one
```
MRM
|
+-+ Data = 10AABB
|
+-+ Data = 10CCDD
|
+-+ Data = 11EEFF
|
+-+ Data = 10GGHH
|
+-+ Data = 11XXYY
|
+-+ Data = 10KKLL\r\n
```
Here the carriage return and line feed that distinguishes one _Header_ and _Optional_ combinations from another are "consumed" by being specified as the **delimiter for all elements**.
### Iteration two
```
MRM
|
+-+ Data
| +-+ Header = AABB
|
+-+ Data
| +-+ Header = CCDD
| +-+ Optional = EEFF
|
+-+ Data
| +-+ Header = GGHH
| +-+ Optional = XXYY
|
+-+ Data
+-+ Header = KKLL
```
Note here how the leading tags have "disappeared". That's because we use them to determine whether a record (as delimited by carriage return and line feed) is of type _Header_ or of type _Optional_. Also the last carriage return and line feed between _Header_ and _Optional_ sections are "consumed" since that's the **delimiter** for the sections within a _Data_ element.
# The initial model
## Defining the structure

![](https://github.com/chids/archived-wmb-message-modelling-tutorials/raw/master/tagged-fixed-length/img/1.PNG)

This is our outline of the structure. Identifying a container (_Data_) and sections (_Header_ and _Optional_) as well as data elements (_First_ and _Second_).

![](https://github.com/chids/archived-wmb-message-modelling-tutorials/raw/master/tagged-fixed-length/img/2.PNG)

Note here that the Data element ha its occurrence set to between zero and infinite

![](https://github.com/chids/archived-wmb-message-modelling-tutorials/raw/master/tagged-fixed-length/img/3.PNG)

We then describe the _Header_ section and it's elements (_First_ and _Second_) that will actually contain data from the input message.

![](https://github.com/chids/archived-wmb-message-modelling-tutorials/raw/master/tagged-fixed-length/img/4.PNG)

The same goes for the _Optional_ section it's elements (_First_ and _Second_) that will, again, actually contain data from the input message.

![](https://github.com/chids/archived-wmb-message-modelling-tutorials/raw/master/tagged-fixed-length/img/5.PNG)

Note here that the _Optional_ section is marked as having an occurrence of either zero or one, thus making it optional.
## Making it parse
Simply defining the structure won't be enough. We must now define how WMB should determine where
one container (_Data_) starts and what bits and bytes in our input message that actually defines a section such as _Header_ and _Optional_ not to mention the actual data elements _First_ and _Second_.
### Root level (the message type) 
![](https://github.com/chids/archived-wmb-message-modelling-tutorials/raw/master/tagged-fixed-length/img/6.PNG)

We start of by specifying that all elements on the root level are delimited and we then specify carriage return and line feed as the separators.
### The container (Data)
![](https://github.com/chids/archived-wmb-message-modelling-tutorials/raw/master/tagged-fixed-length/img/7.PNG)

When then specify that the elements in the _Data_ containter is tagged delimited and that the delimiter between multiple data sections are also carriage return and line feed. Tags to identify a given data section are then specified on each sub-section as displayed further down.
### The mandatory section (Header)
![](https://github.com/chids/archived-wmb-message-modelling-tutorials/raw/master/tagged-fixed-length/img/8.PNG)

Here we specify that children of this element are fixed length. Each child element then specifies the length of it's data.
## Defining the tags
To allow the MRM to determine whether an record is either a mandatory (Header) section or an optional (Optional section we use the leading tags of each line. Since the containing element (Data) is specified as Tagged Delimited (as shown above) we only need to specify the actual tags on the sections.
### Specifying the tag on the mandatory _Header_ section
![](https://github.com/chids/archived-wmb-message-modelling-tutorials/raw/master/tagged-fixed-length/img/9.PNG)
### Specifying the tag on the optional _Optional_ section
![](https://github.com/chids/archived-wmb-message-modelling-tutorials/raw/master/tagged-fixed-length/img/10.PNG)
## The resulting parse tree
This is what the resulting parse tree looks like when we use parse the sample message used in this article together with the message set.
![](https://github.com/chids/archived-wmb-message-modelling-tutorials/raw/master/tagged-fixed-length/img/11.PNG)