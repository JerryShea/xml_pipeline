## Solution
### Components
#### Receiver
Receives XML, assigns and ID, and persists to disk together with received timestamp. Core processing does not generate any garbage and streams XML
straight out to disk. Very low latency and very high throughput.
This job of this component is to "land" incoming XML to disk and make it available for downstream processing pipelines.
#### Normaliser
Tails persisted XML file, loads XML, extracts configured subset of individual tag values from the XML, and persists to disk in normalised file. Three versions have been implemented:

* JAXB implementation converts the XML to a Java object in memory - this takes the most up-front time but is suitable for very complex and very numerous extractions of tags from XML
* XOM implementation uses XOM to allow execute configured xpaths against XML
* Custom implementation that makes use of a low latency XML parser and cut-down support for an xpath subset. Low garbage and considerably lower latency and higher throughout than JAXB and XOM impls.
#### Indexer
Tails normalised file and builds an in-memory index of every normalised value.
#### Queryer
Allows basic queries to be executed suchs as:

* load xml or normalised data by ID
* find xml or normalised data by indexed values
#### ContinuousQuery
Continuous queries e.g. count of received messages in last 10 seconds, alert where a trade's attributes have changed
#### DBWriter
Not implemented yet. Will build on JAXB indexer and write XML to relational tables.
#### Downstream analytics
Not implemented yet.

### Performance
Test harness uses approx 110 example FpMLs from FpML website, some of which have been increased in size, and values substituted values e.g. tradeId, tradeDate etc. Average size of each XML file is approx 29K. 

* Test scenario is to send 1M FpML files, with a total size of approx 29GB. These XMLs are generated using the test harness and passed to Receiver. All components are run simultaneously
* Normaliser tails received XML file and normalises using custom implementation before writing to normalised file
* Indexer tails normalised file and builds in-memory index
* During the test run the Queryer is run to repeatedly execute the same queries: 
    * get all of a particular type 
    * get all on tradeDate of X 
    * get normalised data by random ID 
    * get xml by random ID 
    * get count.

#### Soak test
To test what happens when you throw as much XML in the front only constrained by write speed to disk.
Not entirely realistic as super-fast no garbage XML generation component actually embedded in receiver for maximum speed.

* MacBook Pro 4 cores and SSD:
    * Total test execution time 980 seconds
        * Receiver completed in 495 seconds, persisting 1M messages at 2,000 msgs/sec, each message average #chars 28,858
        * Normaliser completed in 960 seconds, reading, normalising and persisting 1M messages at 1,000 msgs/sec
        * Indexer completes processing in 980 seconds, reading and indexing 1M messages
        * Queryer executes during entire test run, executing 3,000 queries, returning a total of 453MB of data (average 180 rows per query), in an average of 312 ms per query


