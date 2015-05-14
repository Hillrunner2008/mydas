|![http://mydas.googlecode.com/svn/trunk/mydas/server_core/src/main/resources/mydas_logo_small.png](http://mydas.googlecode.com/svn/trunk/mydas/server_core/src/main/resources/mydas_logo_small.png)|A Java Distributed Annotation System (DAS) Server with a Simple API for Developing DAS Sources|[![](http://www.ebi.ac.uk/inc/images/ebilogohome.gif)](http://www.ebi.ac.uk)|
|:--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|:---------------------------------------------------------------------------------------------|:---------------------------------------------------------------------------|

[Index](HELP_INDEX.md) | [Introduction: Obtaining and configuring the MyDas code to start building your data source ->](ObtainingMyDas.md)

### New Release - version 1.6.7 ###

A new version of MyDAS is now available.  In this version the old oscache framework code has been completely removed, as this was causing problems for DAS services with a heavy load.  The oscache project is no longer developed or supported and so it was felt appropriate to remove this code.

If you wish to upgrade to version 1.6.7, you should remove the following method from your  implementation of `uk.ac.ebi.mydas.datasource.AnnotationDataSource`:

```
// Remove your implementation of this method to upgrade to version 1.6.7
public void registerCacheManager(CacheManager cacheManager);
```

As a consequence, MyDAS no longer includes any embedded caching code.  We recommend the use of [ehcache web page caching](http://ehcache.org/documentation/recipes/pagecaching) which is extremely lightweight and reliable, requiring no code changes.

## Introduction ##

This project offers an easy-to-extend Java DAS server framework with several advantages:

  * Implementing data sources is very easy but also flexible and powerful
  * Data caching is built into the system, with access to the caching mechanism made available to the data sources.
  * All aspects of the server are highly configurable, including selecting options where the DAS 1.53 specification offers choices to the implementor.
  * The latest Java technologies have been used throughout the system to optimise performance and simplify data source development.
  * Wherever possible the same terminology is used in the API as in the DAS specification and XML - again, making data source development more easy.

_If you would like to see an example of MyDas in action, please take a look at the [UniProt DAS Annotation Server](http://www.ebi.ac.uk/das-srv/uniprot/das/dsn) at the EBI, which is a popular DAS Reference server with six data sources._

More details of the DAS protocol, DAS servers and DAS clients can be found at http://www.biodas.org/wiki/Main_Page.

If you want to start learning how to use MyDas, you can go to the [Tutorials](Tutorials.md) page.

**Implementations:**

The current version of this server is a complete implementation of http://www.biodas.org/wiki/DAS1.6. If you are interested in learning more about DAS 1.6, this specification is highly recommended as a concise and complete description of the DAS protocol.

The first version of this server was a complete implementation of [Distributed Sequence Annotation System (DAS) Version 1.53](http://biodas.org/documents/spec.html).

**Reference:**

Salazar GA, García LJ, Jones P, Jimenez RC, Quinn AF, Jenkinson AM, Mulder N, Martin M, Hunter S and Hermjakob H (2012) **MyDas, an Extensible Java DAS Server**. PLoS ONE 7(9): e44180. doi:10.1371/journal.pone.0044180

http://www.plosone.org/article/info%3Adoi%2F10.1371%2Fjournal.pone.0044180


**Related publications:**

Prlić A, Down TA, Kulesha E, Finn RD, Kähäri A, Hubbard TJ. **Integrating sequence and structural biology with DAS.** [BMC Bioinformatics.](http://dx.doi.org/10.1186/1471-2105-8-333) 2007 Sep 12;8:333  (PMID: 17850653)

All the source code is available from subversion. (Click on the 'Source' tab above to access this.)

Please see the [documentation for building a mydas data source plugin](HELP_INDEX.md) for details of how to start implementing a plugin (This documentation is a work in progress).

This code is being developed using the following platform, tools and APIs:

|Development OS|Both Windows Vista and Suse Linux 10.1| |
|:-------------|:-------------------------------------|:|
|IDE|IntelliJ IDEA Version 6|http://www.jetbrains.com/idea/|
|Sun Java SDK|jdk1.5.0\_11|http://java.sun.com|
|Build and Project Management Tool|Maven 2.0.6|http://maven.apache.org/|
|Test Framework|JUnit 4.3.1|http://www.junit.org/index.htm|
|Logging|Log4J 1.2.14|http://logging.apache.org/log4j/docs/|
|Logging|commons-logging 1.1|http://jakarta.apache.org/commons/logging/|
|XML Marshalling and Unmarshalling|MXP XMLPullParser API 1.1.3.4.M|http://www.extreme.indiana.edu/xgws/xsoap/xpp/mxp1/|
|Java Collections Framework Extensions|commons-collections 3.2|http://jakarta.apache.org/commons/collections/|

**Note that the Maven build tool downloads all jar dependencies for the project - it will not be necessary to download these individually.**
|![http://mydas.googlecode.com/svn/trunk/mydas/server_core/src/main/resources/mydas_logo.png](http://mydas.googlecode.com/svn/trunk/mydas/server_core/src/main/resources/mydas_logo.png)|MyDas is currently in the process of a major redevelopment to implement the new features in the DAS 1.6 specification, being led by Gustavo Adolfo Salazar Orejuela.  The original project to develop a 1.53 DAS server was led by [Philip Jones](http://www.ebi.ac.uk/Information/Staff/person_maint.php?s_person_id=471) with contributions from [Antony Quinn](http://www.ebi.ac.uk/Information/Staff/person_maint.php?s_person_id=479).  Both Philip and Antony are software developers in the [Proteomics Service Team](http://www.ebi.ac.uk/proteomics/index.html) at the [EMBL-European Bioinformatics Institute](http://www.ebi.ac.uk).  Philip Jones designed and developed the mydas DAS servlet and associated Java API and is responsible for the mydas project documentation.  Antony Quinn is developing a configurable XSLT transformation of the DAS XML to allow it to be viewed in a human-readable format in an internet browser : effectively a simple DAS client built into the DAS server.|[![](http://www.ebi.ac.uk/inc/images/ebilogohome.gif)](http://www.ebi.ac.uk)|
|:--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|:---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|:---------------------------------------------------------------------------|

###### Copyright 2007 Philip Jones, EMBL-European Bioinformatics Institute Licensed under the Apache License, Version 2.0 (the "License"); you may not use this file except in compliance with the License. You may obtain a copy of the License at http://www.apache.org/licenses/LICENSE-2.0 Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.  See the License for the specific language governing permissions and limitations under the License. ######