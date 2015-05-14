# Building the MyDas core application from source #

It is not necessary to follow the procedure on this page to set up your own DAS server using MyDas - you only need to obtain the MyDas jar file and follow the instructions on the ObtainingMyDas page.

However, if you wish to obtain the source code for the entire MyDas project and compile this from scratch, read on...

Maven 2 is a powerful project management, build and test tool that makes managing projects very easy.

Please take a look at the [Apache Maven Project Home Page](http://maven.apache.org/) for further details.  (This page includes a link to [download Maven 2](http://maven.apache.org/download.html)).  This project has been developed using Maven 2.0.6, so this or any later version should work well.

(You will also of course need Java 1.5 or later installed on your machine, with the JAVA\_HOME and PATH environment variables set correctly).


## Building the mydas Server Code ##

The project in subversion is split into two modules, each with their own Maven POM files.  The core project builds the mydas jar file and the example server project builds a complete, example war file that can be deployed to a servlet container.

To assist you with managing this build process, a parent Maven POM file exists in the folder above the two modules.  You may use this to build and test the entire project in one command.  This will compile the core jar, install it to your local Maven repository and then build the war file using this newly compiled jar.

Assuming that you have checked out the source code to a folder called `mydas` and have successfully installed both Java 1.5 and Maven2:

To confirm that your environment is set up correctly, you should be able to successfully run the following commands in any directory:

  * `java -version`
  * `mvn --version`

Once this is confirmed, follow the following to build the complete project:

  * cd to the `mydas` folder.
  * Check that this folder contains a file `pom.xml`.
  * Run the command `mvn install`.
  * This will do the following:
    * compile the mydas core jar file
    * run junit tests on the core jar
    * install the new jar to your local maven repository (by default, located in `HOME_DIRECTORY]/.mvn/repository/`)
    * compile the test data source and / or your new data source including the new core jar as a dependency
    * run any junit tests that have been included for the data sources
    * finally package the data sources in a war file, together with all required dependencies.

After you have done this, you may wish to run your new server using jetty.  To do this, follow these steps:

  * cd to the example\_server\_installation folder
  * Run the command `mvn jetty:run-war`.

The jetty server will be started, by default at localhost:8080.

You should the be able to open an internet browser and access your server, e.g. at:

http://localhost:8080/das/dsn

to view the XML listing the available data sources. (You may need to use the browser's 'View source' facility to view the XML, which may be transformed into an HTML tabulated view by the provided XSLT transformation.)  If instead a java exception is displayed in the browser, this indicates a fatal problem in one or more of the data sources or in the configuration XML file, that has prevented the mydas server from initializing correctly.