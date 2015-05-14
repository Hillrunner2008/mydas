[<- mydas Home Page](http://code.google.com/p/mydas/) | [Index](HELP_INDEX.md) | [Selecting an interface to implement ->](HOWTO_WritePluginIntro.md)


# Introduction #

This page describes how to set up the environment needed to create a MyDas DAS server and how to obtain the necessary source code and binaries for starting development.

# Details #

## Prerequisites ##

To create a MyDas server, you should ensure that you have the following installed and configured on your computer:

  * Sun Java SDK 1.5 (or later) available from http://java.sun.com
  * Ensure the JAVA\_HOME environment variable is set correctly and JAVA\_HOME/bin is included in your PATH environment variable.  Run `java -version` to ensure that the correct version of Java is available on the command line.
  * Maven 2 Project Management / Build tool available from http://maven.apache.org/
  * View the [Instructions for downloading and installing Maven 2](http://maven.apache.org/download.html#installation) to ensure that you have a complete setup.  Run `mvn --version` on the command line to ensure that Maven 2 is correctly installed.

## Obtaining and configuring the project source and dependencies. ##

Thanks to Maven 2, you do not need to worry about obtaining the dependencies for this project.

### The MyDas server core binary jar file ###

A previous version of this documentation indicated that you should install this jar into your local maven repository manually.  **This instruction is now deprecated and should not be done - the MyDas core jar is available directly from the public maven repository and will automatically be downloaded by maven when you build the template project, along with all of the core MyDas dependencies.**.

If you have manually installed the core jar and are experiencing problems with compilation as a consequence, please delete the following folder and its contents prior to building the template project:

`[HOME_FOLDER]/.m2/repository/uk/ac/ebi/mydas`

### Download and Configure the Template Project ###

  * **Download** `MyDasTemplate-N.N.N.zip` from the Downloads page and unzip in an appropriate location.  You now have a template project that you can configure in your favorite IDE to develop your own DAS data sources.  To get you started, so you can compile for the first time, do the following:

  * **DEPRECATED IN VERSION 1.6.7 - Edit the location for data caching** in the file `[MyDasTemplate]/src/main/resources/oscache.properties`
    * In the `oscache.properties` file, set the property `cache.path` to a valid, writable folder on your filesystem.

  * Assuming Maven2 and Java are correctly configured, you should now be able to compile successfully using Maven2 as follows:
    * cd to the [MyDasTemplate](MyDasTemplate.md) folder where you should find the `pom.xml` file.
    * Run the command `mvn clean package` to create a valid war file, **OR**
    * Run the command `mvn clean jetty:run` to run the server using Jetty on `http://localhost:8080/das`.  Once Jetty is up and running, navigating to this URL in your browser should return a page listing the single data source named 'template'.

  * If this is successful, you have a valid configuration that you can use as a basis for developing your DAS data source.  The files you are most likely to want to edit immediately include:
    * `[MyDasTemplate]/src/main/webapp/MydasServerConfig.xml` to configure your DAS server and DAS source.  (You should specify the fully-qualified class name for your data source in this file, if different from the TemplateAnnotationDataSource.java file provided in the template project).
    * `[MyDasTemplate]/src/main/java/uk/ac/ebi/mydas/template/TemplateAnnotationDataSource.java`.  You will probably wish to refactor this class to a different package and classname and then configure this in the MydasServerConfig.xml file mentioned above.  You can also change the interface that you are implementing as described on the HOWTO\_WritePluginIntro page.  You should then implement the required methods for your class.