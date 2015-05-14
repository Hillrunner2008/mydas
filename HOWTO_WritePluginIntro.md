[<- Getting Started: Obtaining and Installing the MyDas code](ObtainingMyDas.md) | [Index](HELP_INDEX.md) | [Implementing a DataSource ->](HOWTO_Implement_A_DataSource.md)

# Getting Started: Selecting the best Interface to Implement #

The [DAS 1.53 Specification](http://biodas.org/documents/spec.html) describes two different kinds of DAS server:
  * annotation servers which are able to provide feature annotation on nucleic acid or protein sequences;
  * reference servers can provide annotation and also sequence information for proteins and nucleic acids.

Annotation servers are expected to refer to a specific reference server for sequence information.

This API allows the development of both annotation and reference data sources.  It is possible to create one mydas server with multiple data sources that might be a mixture of annotation and reference sources.

This page describes how to get started with implementing a data source, specifically by selecting the correct interface to implement.

# Details #

The data source interfaces are located in the `uk.ac.ebi.mydas.datasource` package.  There are four of these that should be selected from as described in the following diagram:

![http://mydas.googlecode.com/svn/trunk/mydas/server_core/src/main/java/uk/ac/ebi/mydas/datasource/Selecting-an-interface.png](http://mydas.googlecode.com/svn/trunk/mydas/server_core/src/main/java/uk/ac/ebi/mydas/datasource/Selecting-an-interface.png)
