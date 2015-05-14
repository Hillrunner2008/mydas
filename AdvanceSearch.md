# Advance Search #
one of the features of the  [DAS1.6E](http://www.biodas.org/wiki/DAS1.6E#DAS_search) is the possibility of executing elaborated searches through a data source. The specification includes the query language and the responses. The query is similar to and the responses follow the same structure as the standard DAS feature command/response.


# Indexing #
MyDas implements the advance search by using [LUCENE](http://lucene.apache.org/java/docs/index.html), this search engine works by creating an indexed file to allow rapid acces to the information. MyDas allows the creation of the index as another DAS command, although some configuration steps are necessary and the indexing process can take several hours depending of how many features the data source contains.

The first step is to create a password/keyphrase in order to avoid undesired runs of the indexing process. To do this open the configuration file MyDasServerConfig.xml and edit/create a new global property with key equal to `keyphrase` and value is your chosen keyword, for instance `confirmation`.
The path where the index is going to be created has to be indicated in the same way, where the key is now `indexerpath` and value is a writeble directory in your system.
```

<mydasserver xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
 xsi:noNamespaceSchemaLocation="http://mydas.googlecode.com/svn/trunk/mydas/server_core/src/main/resources/mydasserverconfig.xsd">
    <!-- The global element holds details of configuration common to the entire server. -->
    <global>
        <property key="keyphrase" value="confirmation"/>
        <property key="indexerpath" value="/tmp"/>

...

```

Now as you might have more than one datasource in the same MyDas server, you now have to indicate in the same file which data source will support the Advance Search capability, so i order to do this, you just have to add the following capability element to the selected data source:
```
<capability type="das1:advanced-search" />
```
So, for instance the data source configuration of the other capabilities [Tutorial](MyDasTutorialOtherCapabilities.md) will look like this one when the new capability is added:
```
        <datasource uri="ensemblTest"
                    title="Ensembl Test"
                    description="Data source test using the ensmbl database"
                    doc_href="http://mydas.googlecode.com"
                    mapmaster="http://mapmaster.esemblTest">
			<maintainer email="gsalazar@ebi.ac.uk" />

			<version uri="ensemblTest" created="2010-04-01">
	            <class>uk.ac.ebi.mydas.examples.EnsemblTestDataSource</class>
				<coordinates 	uri="http://www.dasregistry.org/dasregistry/coordsys/CS_DS94"
								source="Gene_ID"
								authority="Ensembl"
								taxid="9606"
								test_range="ENSG00000160916">Ensembl,Gene_ID,Homo sapiens</coordinates>
				<capability type="das1:sources"		 	query_uri="http://localhost:8080/das/ensemblTest" />
				<capability type="das1:types"			query_uri="http://localhost:8080/das/ensemblTest/types" />
				<capability type="das1:features"		query_uri="http://localhost:8080/das/ensemblTest/features" />
				<capability type="das1:feature-by-id"	query_uri="http://localhost:8080/das/ensemblTest/features" />
				<capability type="das1:stylesheet"		query_uri="http://localhost:8080/das/ensemblTest/stylesheet" />
				<capability type="das1:unknown-feature"	/>
				<capability     type="das1:maxbins"     />
				<capability     type="das1:entry_points"        />
				<capability type="das1:advanced-search" />
			</version>

            <dna-command-enabled>true</dna-command-enabled>
            <use-feature-id-for-feature-label>true</use-feature-id-for-feature-label>
            <include-types-with-zero-count>true</include-types-with-zero-count>
        </datasource>
```
To start the indexing process the command `indexer` has to be invoked as an URL including the defined keyphrase, so for example for a localhost server, the URL might look like:
```
http://localhost:8080/das/indexer?keyphrase=confirmation
```

**Warning**: This process  can take from minutes to several hours depending of how much information your data source has.
**Warning**: Please notice that the queries are done over the index and not over the original data, which implies that if your data change you will have to run the indexing process again to make sure the DAS source is up to date.


# Querying #
Once the index is ready it is possible to execute advance queries to your data source. As described in the   [specs](http://www.biodas.org/wiki/DAS1.6E#DAS_search), the `query` is an extra parameter of the DAS command `features`, so you can submit queries to the data source as the ones below:
```
http://localhost:8080/das/test/features?query=featureLabel:"one Feature"

http://localhost:8080/das/test/features?query=typeCvId:CV\:00001 AND featureLabel:"one Feature"

http://localhost:8080/das/test/features?query=(typeCvId:CV\:00001 AND featureLabel:"one Feature") OR typeId:twoFeatureTypeIdOne
```