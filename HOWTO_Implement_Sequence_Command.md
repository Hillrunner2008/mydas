[<- How to Implement a DataSource](HOWTO_Implement_A_DataSource.md) | [Index](HELP_INDEX.md) | [Implementing the features command ->](HOWTO_Implement_Features_Command.md)

# Implementing the sequence command (DAS Reference Server Sources Only) #

This page describes how to implement the sequence command.  This is based upon the assumption that you are implementing a DAS reference server and are therefore implementing either the `ReferenceDataSource` interface or the `RangeHandlingReferenceDataSource` interface.  (See [HOWTO\_WritePluginIntro](HOWTO_WritePluginIntro.md) to find out which of these interfaces you should use).


# Details #

The `ReferenceDataSource` class defines the method:
`public DasSequence getSequence (String segmentReference) throws BadReferenceObjectException, DataSourceException` that returns a `DasSequence` object.  This `DasSequence` object is constructed with the complete sequence of the requested segment id.  The mydas servlet then does the work of returning only the requested sequence, where the request has specified restrictive coordinates.

If you are implementing a reference server data source that is able to restrict the sequence returned to specified coordinates, you may instead implement the `RangeHandlingReferenceDataSource` interface that also defines the method:
`public DasRestrictedSequence getSequence (String segmentReference, int start, int stop) throws CoordinateErrorException, BadReferenceObjectException, DataSourceException`.  This method returns a `DasRestrictedSequence` object.  The sequence included in this object includes only the requested sequence, not the entire sequence of the requested segment.

Both the `DasSequence` and the `DasRestrictedSequence` classes are illustrated in the following diagram:

![http://mydas.googlecode.com/svn/trunk/mydas/server_core/docs/UML/sequence-command-model-class-diagram.png](http://mydas.googlecode.com/svn/trunk/mydas/server_core/docs/UML/sequence-command-model-class-diagram.png)