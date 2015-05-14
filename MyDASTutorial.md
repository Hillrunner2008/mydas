# MyDAS 1.6 Tutorial #

## First Part: Getting MyDas to work and checking the example data source ##

### Download and Configure the Template Project ###

  * **Download** [MyDasTemplate-1.6.5.zip](http://mydas.googlecode.com/files/MyDasTemplate-1.6.5.zip) from the Downloads page and unzip in an appropriate location.  You now have a template project that you can configure in your favorite IDE to develop your own DAS data sources.
  * **Edit the location for data caching** in the file `[MyDasTemplate]/src/main/resources/oscache.properties`
    * In the `oscache.properties` file, set the property `cache.path` to a valid, writable folder on your filesystem.

  * Assuming [Maven2](http://maven.apache.org/download.html) and [Java](http://java.sun.com/javase/downloads/index.jsp) are correctly configured, you should now be able to compile successfully using Maven2 as follows:
    * cd to the `[MyDasTemplate]` folder where you should find the `pom.xml` file.
    * Run the command `mvn clean jetty:run` to run the server using Jetty.

### The Server is Running ###

  * Once Jetty is up and running, navigating to `http://localhost:8080/das/sources` in your browser should return a XML listing the DAS sources of this server, similar to:
```
<?xml version="1.0" standalone="no"?>
<SOURCES>
  <SOURCE	uri="template" 
		doc_href="http://code.google.com/p/mydas/" 
		title="template" description="Template Annotation Data Source">
    <MAINTAINER email="template@template.com" />
    <VERSION uri="template" created="2010-03-17">
      <COORDINATES	uri="http://uritocoordinatesystem.com" 
			source="typeOfReference" 
			authority="organisationResponsible" 
			test_range="P00280">UniProt,Protein Sequence
      </COORDINATES>
      <CAPABILITY	type="das1:capability_name" 
			query_uri="http://actualDASURLusedtoexecuteacommand.com" />
    </VERSION>
    <PROPERTY name="templateexamplekey" value="templateexamplevalue" />

  </SOURCE>
  <SOURCE	uri="examplegff" 
		doc_href="http://mydas.googlecode.com/svn/trunk/mydas/MyDasTemplate/src/main/resources/fickett-tss.gff" 
		title="fickett-tss" 
		description="GFF2 Example Annotation Data Source">
    <MAINTAINER email="gsalazar@ebi.ac.uk" />
    <VERSION uri="examplegff" created="2010-03-19">
      <COORDINATES	uri="http://www.notARealCoordinateSystem.com"
			source="-" 
			authority="unknown" 
			taxid="0000" 
			test_range="L47615.1">TEST INFO
      </COORDINATES>
      <CAPABILITY	type="das1:sources" 
			query_uri="http://localhost:8080/das/examplegff" />
      <CAPABILITY	type="das1:types" 
			query_uri="http://localhost:8080/das/examplegff/types" />
      <CAPABILITY	type="das1:features" 
			query_uri="http://localhost:8080/das/examplegff/features?segment=U29912.1" />
      <CAPABILITY	type="das1:feature-by-id" 
			query_uri="http://localhost:8080/das/examplegff/features?feature_id=GFF_feature_1" />
      <CAPABILITY	type="das1:stylesheet" 
			query_uri="http://localhost:8080/das/examplegff/stylesheet" />
      <CAPABILITY	type="das1:unknown-feature" />
    </VERSION>
    <PROPERTY name="gff_file" value="fickett-tss.gff" />
  </SOURCE>
</SOURCES>
```

  * The DAS 'sources' command for this server show us two data sources named 'template' and 'examplegff'.
    * The data source 'template' is a blank project that can be use as an start point to create a new database.
    * The second data source 'examplegff' is a ready-to-use data source that load a gff file and uses its content as the segments and features that can be queried.
      * From the response of the command sources is possible to known that the examplegff source has implemented the following capabilities: sources, types, features, feature-by-id, stylesheet and unknown-feature.
      * That response is also providing information about the coordinate system used. In this case as the data is an example this information points to a dummy coordinate sysem.
      * This source has an extra property which specifies the gff file that is been loaded to be publish as a DAS source. in this case the [fickett-tss.gff](http://mydas.googlecode.com/svn/trunk/mydas/MyDasTemplate/src/main/webapp/fickett-tss.gff)

### Exploring the Example Data Source ###
Now that the data source is working we can start using the available capabilities. Try the following commands and compare the results with the original file ([fickett-tss.gff](http://mydas.googlecode.com/svn/trunk/mydas/MyDasTemplate/src/main/webapp/fickett-tss.gff)):
  * Source: http://localhost:8080/das/examplegff
  * Types: http://localhost:8080/das/examplegff/types
  * Features: http://localhost:8080/das/examplegff/features?segment=L47615.1
  * Feature-by-id: http://localhost:8080/das/examplegff/features?feature_id=GFF_feature_1
  * Stylesheet: http://localhost:8080/das/examplegff/stylesheet
  * Unknown-feature: http://localhost:8080/das/examplegff/features?feature_id=xxxxx


### Reconfiguring the Example Data Source ###
As you can see the gff file used in this example is quite simple with very few annotations, the next tasks will allow you to reconfigure this data source to use data from a real biological source.
  * Get the gff file. In this example we will use the mitocondrial DNA annotations of the C. elegans given that the implementation of this data source is running in memory and using a Full genome or even a full chromosome can be too demanding for the resources of your local Machine.(More sophisticated implementations can be created for this goal).
    * Go to [ftp://ftp.sanger.ac.uk/pub/wormbase/releases/WS220/genomes/c_elegans/genome_feature_tables/GFF2](ftp://ftp.sanger.ac.uk/pub/wormbase/releases/WS220/genomes/c_elegans/genome_feature_tables/GFF2)
    * Save the file CHROMOSOME\_MtDNA.gff in the folder `[MyDasTemplate]/src/main/webapp/`
  * Stop the jetty server by pressing 'ctrl+c' on the terminal that is running.
  * Open the file `[MyDasTemplate]/src/main/webapp/MydasServerConfig.xml` with your favorite editor.
  * Locate the data source with uri = "examplegff"
  * Change the value of the attribute email in the tag `<maintainer>` for your own email
  * Change the value of the attribute uri in the tag `<coordinates>` for `http://www.dasregistry.org/coordsys/CS_DS52`
  * Change the value of the attribute source in the tag `<coordinates>` for `Chromosome`
  * Change the value of the attribute authority in the tag `<coordinates>` for `CEL`
  * Change the value of the attribute taxid in the tag `<coordinates>` for `6239`
  * Change the value of the attribute test\_range in the tag `<coordinates>` for `CHROMOSOME_MtDNA`
  * Change the content of the the tag `<coordinates>` for `CEL_160,Chromosome,Caenorhabditis elegans`
  * Look for the line `<property key="gff_file" value="fickett-tss.gff" />` and change it for `<property key="gff_file" value="CHROMOSOME_MtDNA.gff" />`
  * Save the changes and start the jetty servers by the maven command at the console: `mvn clean jetty:run`
  * Try the following DAS commands:
    * Source: http://localhost:8080/das/examplegff
    * Types: http://localhost:8080/das/examplegff/types
    * Feature-by-id: http://localhost:8080/das/examplegff/features?feature_id=GFF_feature_1
    * Features: http://localhost:8080/das/examplegff/features?segment=CHROMOSOME_MtDNA - Note that this can take some minutes depending of your system, because is basically asking for the whole data source.


**1st Tutorial(Beginners level)** | [2nd Tutotial(Intermediate level) >](MyDASTutorial2ndPart.md) | [3rd Tutorial(Advance level) >>](MyDASTutorial3rdPart.md)