# Solr

 - Quick introduction to Solr
 - Steps to configure and setup Solr in IOP
 - How it integrates with Hadoop HDFS
 - Show how Solr works with an example

## What is Solr? 

Solr is a enterprise search server with REST-like interface. It manages documents such as JSON, XML, CSV, or binary files by indexing them to provide the capability to search those documents in a fraction of the time. Queries, including updates and other functions, can be managed through HTTP requests to receive results. 

Some features of Solr: 
* advanced full text search capabilities
* optimized for high volume traffic
* based on standard interfaces (xml, json, HTTP)
* admin interface
* easy monitoring of metric data via JMX
* scalable and fault tolerant via zookeeper
* flexible and adaptable
* near real-time indexing via Lucene
* extensible plugin architecture

Below we will provide a simple proof of concept in which the entire contents of wikipedia (14,000,000 documents) are indexed for query. This data can be found at the [wikipedia](https://en.wikipedia.org/wiki/Wikipedia:Database_download) data dump. The file used here is the pages-articles.xml.bz2 file- 13gb compressed, 61gb uncompressed. 

## How to Install Solr on CentOS

Solr is installed as part of the IBM open platform package (this document will make use of IOP 4.1.0.0). The version included in IOP is Solr 5.1.0. This version may be preferable for enterprise search versus standalone Solr as it is automatically configured for use with the Hadoop file system. 

A guide to set up one node IOP can be found [here](http://cleverowl.uk/2015/10/22/installation-of-ibm-open-platform-with-apache-hadoop-version-4-1/). 

As of this writing, Solr does not have a full admin interface through Ambari.

![Ambari interface](https://raw.githubusercontent.com/kdallas2/solr-tools/master/iop41_install_14.png)

What it does have is an interface for starting and stopping Solr only. Note, if Solr is instead started through terminal with "bin/solr start" it will not be preconfigured for index storage or data retrieval from HDFS. Starting the service through Ambari's interface will set directory factory to HdfsDirectoryFactory automatically. 

Once the Solr service has been started, the web interface can be accessed through localhost:8983 (default port number). 

## Configuring Solr to Index Wikipedia

To start an index one must create a "core". In the web interface, this can be done through Core Admin. The core will represent the structure and configuration of your document index. Each core has four main configuration files that are required to run, and a fifth we will use to help us index wikipedia. These files are: 
* solr.xml
* core.properties
* solrconfig.xml
* schema.xml
* data-config.xml (technically optional but provides the import handling function for our wikipedia xml file)

#### solr.xml

This is the highest level document that the Solr instance looks for. The contents can technically be as simple as:

    <?xml version="1.0" encoding="UTF-8" ?>
    <solr><solrcloud /></solr>

When the core is created through Core Admin, a default solr.xml will be created with some additional properties for use with HDFS, including a shard factory and solrcloud configurations.

Once the Solr instance finds solr.xml, it will begin core-autodiscovery, which searches directories beneath the one containing solr.xml for the file called core.properties, which declares the directory containing configuration files for that particular core. 

#### core.properties

A simple Java properties file. May contain some pointers to the names of your config files, but for the most part the default values are sufficient (and do not need to be declared in the file). The only line to be included for our purposes is to name our core:

    name=Wiki

The rest of the files (solrconfig.xml, schema.xml, data-config.xml, etc.) are located in a sub-directory to the directory containing core.properties, named conf. 

#### solrconfig.xml

This xml file contains the most information for the actually configuration of the core. 3 additions to be made to the file automatically generated. Include:

    <lib dir="${solr.install.dir:../../../..}/dist/" regex="solr-dataimporthandler-.*\.jar" />
    
For our data import.

Make sure the following is set, instead of "ManagedIndexSchemaFactory"; will provide more control over our schema.

    <schemaFactory class="ClassicIndexSchemaFactory"/>
    
The last is:

    <requestHandler name="/dihupdate" class="org.apache.solr.handler.dataimport.DataImportHandler" startup="lazy">
      <lst name="defaults">
	        <str name="config">data-config.xml</str>
     	</lst>
    </requestHandler>
    
Declares our request handler and makes it available for HTTP requests.

#### schema.xml

This file determines how we will treat our data. Besides the default information included, this chunk needs to be added:

    <field name="_version_" type="long" indexed="true" stored="true"/>
    <field name="id"        type="string"  indexed="true" stored="true" required="true"/>
    <field name="title"     type="string"  indexed="true" stored="true"/>
    <field name="revision"  type="int"     indexed="false" stored="false"/>
    <field name="user"      type="string"  indexed="false" stored="true"/>
    <field name="userId"    type="int"     indexed="false" stored="false"/>
    <field name="text"      type="text_en" indexed="true"  stored="false"/>
    <field name="timestamp" type="date"    indexed="false" stored="true"/>
    <field name="titleText" type="text_en" indexed="true"  stored="true"/>
    <uniqueKey>id</uniqueKey>
    <copyField source="title" dest="titleText"/>
    
All other field and otherwise redundant tags can be removed.

#### data-config.xml

As we defined our data import handler in the solrconfig.xml file, this is where it points to. 

    <?xml version="1.0" encoding="UTF-8" ?>
    <dataConfig>
        <dataSource type="FileDataSource" encoding="UTF-8" />
        <document>
        <entity name="page"
                processor="XPathEntityProcessor"
                stream="true"
                forEach="/mediawiki/page/"
                url="D:\Data\Wiki\pages-articles.xml"
                transformer="RegexTransformer,DateFormatTransformer"
                >
            <field column="id"        xpath="/mediawiki/page/id" />
            <field column="title"     xpath="/mediawiki/page/title" />
            <field column="revision"  xpath="/mediawiki/page/revision/id" />
            <field column="user"      xpath="/mediawiki/page/revision/contributor/username" />
            <field column="userId"    xpath="/mediawiki/page/revision/contributor/id" />
            <field column="text"      xpath="/mediawiki/page/revision/text" />
            <field column="timestamp" xpath="/mediawiki/page/revision/timestamp" dateTimeFormat="yyyy-MM-dd'T'hh:mm:ss'Z'" />
            <field column="$skipDoc"  regex="^#REDIRECT .*" replaceWith="true" sourceColName="text" />
        </entity>
        </document>
    </dataConfig>
    
Notice the fields defined here were all defined earlier in schema.xml and they make extensive use of xpath to navigate the wiki dump file.

The last field $skipDoc skips those that are simply redirect documents in wikipedia.


#### Index pages-articles.xml

A data import can now be run. The request can be made through accessing URL localhost:8983/solr/Wiki/dihupdate?command=full-import

Once the command is run, it will begin indexing. You can check the status by accessing localhost:8983/solr/Wiki/dihupdate

Indexing 14,000,000 documents took approximately 1.5 hours on a quad core CPU with 24gb RAM and 7200rpm USB hdd. 


#### Using the index

Our index may now be used to search the contents of wikipedia. If we make a simple query by the web interface Query page, and change *.* to text:"Royal Robbins", we can find every article with a reference to that famous mountain climber in the text. The search takes approximately 1 second. 

![Royal Robbins search](https://raw.githubusercontent.com/kdallas2/solr-tools/master/royal.png)


## Conclusion

Solr is a powerful tool to index and search big data repositories, particularly as implemented through IOP. It is maintainable and scalable through sharding and usage of HDFS as well as easy implementation of search characteristics and modules. It is reliable and fault tolerant through replication. Accessibility is excellent through Lucene real-time indexing and the REST-like interface. One concern is security, although Kerberos security may be implemented to require authorization to use the interface or submit requests.

Thank you
