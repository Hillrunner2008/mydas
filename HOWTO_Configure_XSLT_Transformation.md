[<- How to build sequence assemblies](HOWTO_Build_Sequence_Assemblies.md) | [Index](HELP_INDEX.md)

# Configuring XSLT Transformation #

The MyDas DAS server includes a simple DAS client that allows the DAS response (i.e. raw XML) to be transformed into a well formatted, human readable HTML page when viewed in a modern internet browser.


# Details #

The primary purpose of a DAS server is, of course, to serve well formatted XML according to the DAS specification.  However, by including an _XML procressing instruction_ at the top of the XML file, which includes a URL pointing to the location of an XSLT document, it is possible to render the contents of the XML in an HTML page with whatever layout and format is desired by the page designer.

A typical processing instruction might look like this (example using a relative URL to the XSLT file):

```
<?xml-stylesheet href="../web/features.xsl" type="text/xsl"?> 
```

The processing instruction will be ignored by normal DAS clients, which will just receive the expected XML file.

The current version of the MyDAS server includes configuration allowing different XSLT transformations to be applied to the XML served for different XML commands.  These configurations are all optional, so it is possible to set up a MyDas DAS server without any of these transformations being available.