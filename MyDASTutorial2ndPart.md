# MyDAS 1.6 Tutorial #

## Second Part: Developing your own Data Source ##
Now that you have a real DAS source working on MyDAS lets get closer to a more interesting example: what if you have your data in an in-house format and therefore the templates/examples don't apply for your data source?

Following this tutorial you will parse an in-house formated file, publish its information as a data source in MyDAS using the DAS command feature. You can used the IDE of your choice but the tutorial is done in such a way you can use just a terminal and a text editor.

### Example File ###
The file of this example was built using information from the ensembl database homo\_sapiens\_core\_56\_37a. The file gives positional information of genes, transcripts and exons in the chromosome 5 organized in a separated by pipes `|` like this:
```
|chr|gene_id|gene_start|gene_end|trascript_id|transcript_start|transcript_end|exon_id|exon_start|exon_end|
```
You can see a sample of this file below.
```
|5|ENSG00000153404|140373|190087|ENST00000398036|140373|157131|ENSE00001648483|156888|157131|
|5|ENSG00000153404|140373|190087|ENST00000398036|140373|157131|ENSE00001135995|156186|156325|
|5|ENSG00000153404|140373|190087|ENST00000398036|140373|157131|ENSE00001136002|155460|155558|
|5|ENSG00000153404|140373|190087|ENST00000398036|140373|157131|ENSE00001136007|154990|155106|
|5|ENSG00000153404|140373|190087|ENST00000398036|140373|157131|ENSE00001136016|151628|151714|
|5|ENSG00000153404|140373|190087|ENST00000398036|140373|157131|ENSE00001136025|144942|145035|
|5|ENSG00000153404|140373|190087|ENST00000398036|140373|157131|ENSE00001136029|143495|143618|
```


### Analyzing the Data ###
Firstly, we know that our file just contains information for one segment. The chromosome 5, however we can create our source to support files with information from more than one chromosome.
We also know from this file the information that our data source has to provide, and from there that there will be just three types of features in this data source:
  * genes: The file gives information about the start and end of the genes plus its ID. From there we have another type in our data source:
```
<TYPE id="Gene" cvId="SO:0000704">Gene</TYPE>
```
> But also we have the id field of each feature of this type and the tags start and end can be also deduced from there.
  * transcripts: As in the case of genes the information provided is the id, start and stop. So we will need a new DAS type:
```
<TYPE id="Transcript" cvId="SO:0000673">Transcript</TYPE>
```
  * exons: Same analysis as above:
```
<TYPE id="Exon" cvId="SO:0000147">Exon</TYPE>
```
Note that the numbers at cvId are valid Ontology Ids from the Sequence Ontology [SO](http://www.ebi.ac.uk/ontology-lookup/browse.do?ontName=SO).

The other information that we have implicit in this file is that there is a hierarchy in those components. A chromosome contains genes, a gene contains transcripts, an transcript contains exons.

For this demo we can assume some values for all the annotations, the method will be
```
<METHOD id="combinatorial analysis used in automatic assertion" cvId="ECO:0000213">combinatorial analysis used in automatic assertion</METHOD>
```
And given that we don't know the score, phase and orientation, we will assume those values as not applicable.

### Implementing the Data Source ###
Now lets start working.  The task of implementing the data source can be divided in three parts:
  1. Implement a parser of the file.
  1. Implement one of the MyDAS Data Sources Interfaces.
  1. Configure MyDAS with the new Data Source.

Something to consider first is the strategy to deal with the data. It means if  the file is going to be parsed completely and keep it in memory as a model (As the gff example) or if the file will be processed every time something is requested.

Pros and cons? the whole file in memory will response quicker but the bigger the file, the more memory the system will require. Opposite situation with the other case; processing the file for each request doesn't require too much memory, but the response time will be proportional to the size of the file.

Maybe an intermediate approach with a preprocessing stage, and subsequents specialized processes per query, can be a better way, however it will require to focus in details that are not in the scope of this tutorial.

The approach to follow here will be parsing the file and keeping a model in memory. It is quite similar to the one used for the gff example, so you might one to have a look on the previous tutorial[(1st Tutorial)](MyDASTutorial.md) before to start coding.

So, lets start coding a parser for this file. The goal of the parser is to get the info loaded in memory using the model of MyDAS to represent segments and annotations.


#### Implement a parser of the file ####
  1. Create a file in `[MyDasTemplate]/src/main/java/uk/ac/ebi/mydas/examples/` called `SeparatedByPipesParser.java` and open it to edit with the editor of your choice(vi, write, eclipse, intelliJ, etc.)
  1. Add the empty class declaration with his package and the classes to import. You can add those imports now or whenever they are required:
```
 package uk.ac.ebi.mydas.examples;

import java.io.FileInputStream;
import java.io.FileNotFoundException;
import java.io.InputStream;
import java.util.ArrayList;
import java.util.Collection;
import java.util.HashSet;
import java.util.Scanner;

import uk.ac.ebi.mydas.exceptions.DataSourceException;
import uk.ac.ebi.mydas.model.*;


public class SeparatedByPipesParser {
}
```
  1. Define the attributes of the class, which in this case are:
    * scanner: Object used to process the file line by line.
    * segments: List of the parsed segments.
    * types: List of the types used in this data source. From a previous analysis: chromosome, gene, transcript, exon. Attributes for each of those types can be created to facilitate its use.
    * method: From the analysis of the file(above). we are going to use the same method for all the features.
> > The code for this will be something like:
```
	private Scanner scanner;
        private ArrayList<DasAnnotatedSegment> segments;
        private ArrayList<DasType> types;
        private DasType chromosomeType,geneType,transcriptType,exonType;
        private DasMethod method;
        private Collection<DasFeature> features=new ArrayList<DasFeature>();
        
```
  1. Code the constructor of the class. It should receive an InputStream, instantiate the scanner with the stream, create the empty lists for types and segments and create the types and method to use through the source.
```
	public SeparatedByPipesParser(InputStream gffdoc) throws DataSourceException{
                scanner= new Scanner(gffdoc);
                types= new ArrayList<DasType>();
                geneType= new DasType("Gene", null, "SO:0000704", "Gene");
                transcriptType= new DasType("Transcript", null, "SO:0000673", "Transcript");
                exonType= new DasType("Exon", null, "SO:0000147", "Exon");
                types.add(geneType);
                types.add(transcriptType);
                types.add(exonType);
               method = new DasMethod("combinatorial analysis used in automatic assertion","combinatorial analysis used in automatic assertion","ECO:0000213");
      
        }
```
  1. Now we have to go though the whole file line by line. To do that you can create the next method:
```
	public Collection<DasAnnotatedSegment> parse() throws Exception{
            segments=new ArrayList<DasAnnotatedSegment>();
                try {
                        //first use a Scanner to get each line
                        while ( scanner.hasNextLine() ){
                                processLine( scanner.nextLine() );
                        }
                } finally {
                        //ensure the underlying stream is always closed
                        scanner.close();
                }
                DasAnnotatedSegment segment=new DasAnnotatedSegment("5", 1,1000000000,"1.0", "5", features); 
                segments.add(segment);
                return segments;
        }
```
  1. As you can see the previous code makes use of the method `processLine()` which we haven't created, so thats our next step. A line of this file is a a set of fields divided by pipes (`|`) as is described before, so we can process a line by splitting it by pipes. The acquired data is used then, to call the methods to get/create the segment(chromosome) and features(gene, transcript or exon).
```
	private void processLine(String aLine) throws Exception{
                String[] parts = aLine.split("\\|");
                if (parts.length<11)
                        throw new Exception("Parsing Error: A line doesn't have the right number of fields ["+aLine+"]");
                
                DasFeature gene= this.getFeature(parts[2],parts[3],parts[4], geneType, null);
                //System.out.println("gene="+gene);
                DasFeature transcript= this.getFeature(parts[5],parts[6],parts[7],transcriptType, gene);
                gene.getParts().add(transcript.getFeatureId());
                //System.out.println("transcript="+transcript);
        DasFeature exon = this.getFeature(parts[8],parts[9],parts[10],exonType, transcript);
        transcript.getParts().add(exon.getFeatureId());
         //System.out.println("exon="+exon);
         
        }
```

  1. Now we need the methods to create a gene, transcript and exon. As they only differ in the DasType we can use the same method to create a feature. First look in our feature list for that we have already and if is found is returned, if not a new instance is created and added. The code for the feature method and the check if feature exists is below:
```
	private DasFeature getFeature(    String geneID,
                                                String start,
                                                String stop, DasType type, DasFeature parent) 
        throws DataSourceException {
            //return the feature if it already exists
                DasFeature feature=this.getFeatureWithId(geneID);
                if(feature!=null)return feature;
                int startI=0,stopI=0;
                try {
                        startI=Integer.parseInt(start);
                        stopI=Integer.parseInt(stop);
                }catch (NumberFormatException nfe){
                        throw new DataSourceException("PARSE ERROR: The coordinates for the gene "+geneID+" should be numeric",nfe);
                }
                DasFeatureOrientation orientation= DasFeatureOrientation.ORIENTATION_SENSE_STRAND;
		
		DasPhase phase= DasPhase.PHASE_READING_FRAME_0;
		Double score=0.0;
                HashSet parents=new HashSet();
                if(parent!=null){
                parents.add(parent.getFeatureId());
                }
                DasFeature gene=new DasFeature(geneID,geneID,type, method, startI, stopI , score,orientation,phase, null, null, null, parents, new HashSet());
                features.add(gene);
                return gene;
        }

        private DasFeature getFeatureWithId(String featureId){
            for(DasFeature feature:features){
                if(feature.getFeatureId().equals(featureId)){
                    return feature;
                }
            }
            return null;
        }
```
  1. The last method to add is `getTypes()` in order to return the private attribute types.
```
	public Collection<DasType> getTypes() {
		return types;
	}
```
#### Implement one of the MyDAS Data Sources Interfaces ####
For this example we are going to implement the AnnotationDataSource. Create a file in `[MyDasTemplate]/src/main/java/uk/ac/ebi/mydas/examples/` called `SeparatedByPipesDataSource.java`. The content of an empty class that implements this Interface is below, and it contains the classes to import, and the definition of the empty methods forced to be implemented by that interface.
```
package uk.ac.ebi.mydas.examples;

import java.io.FileInputStream;
import java.io.FileNotFoundException;
import java.net.URL;
import java.util.ArrayList;
import java.util.Collection;
import java.util.Collections;
import java.util.Map;

import javax.servlet.ServletContext;

import uk.ac.ebi.mydas.configuration.DataSourceConfiguration;
import uk.ac.ebi.mydas.controller.CacheManager;
import uk.ac.ebi.mydas.datasource.AnnotationDataSource;
import uk.ac.ebi.mydas.exceptions.BadReferenceObjectException;
import uk.ac.ebi.mydas.exceptions.DataSourceException;
import uk.ac.ebi.mydas.exceptions.UnimplementedFeatureException;
import uk.ac.ebi.mydas.extendedmodel.DasUnknownFeatureSegment;
import uk.ac.ebi.mydas.model.DasAnnotatedSegment;
import uk.ac.ebi.mydas.model.DasComponentFeature;
import uk.ac.ebi.mydas.model.DasFeature;
import uk.ac.ebi.mydas.model.DasType;
import uk.ac.ebi.mydas.model.DasEntryPoint;

import uk.ac.ebi.mydas.configuration.PropertyType;
import uk.ac.ebi.mydas.model.*;



public class SeparatedByPipesDataSource implements AnnotationDataSource {

	public void init(ServletContext servletContext,
			Map<String, PropertyType> globalParameters,
			DataSourceConfiguration dataSourceConfig)
			throws DataSourceException {
	}

	public void destroy() {
	}

	public DasAnnotatedSegment getFeatures(String segmentId, Integer maxbins)
			throws BadReferenceObjectException, DataSourceException {
		return null;
	}

	public Collection<DasAnnotatedSegment> getFeatures(
			Collection<String> featureIdCollection, Integer maxbins)
			throws UnimplementedFeatureException, DataSourceException {
		return null;
	}

	public URL getLinkURL(String field, String id)
			throws UnimplementedFeatureException, DataSourceException {
		return null;
	}

	public Integer getTotalCountForType(DasType type)
			throws DataSourceException {
		return null;
	}

	public Collection<DasType> getTypes() throws DataSourceException {
		return null;
	}

	public void registerCacheManager(CacheManager cacheManager) {
	}
	public Collection<DasEntryPoint> getEntryPoints(Integer start, Integer stop) 
			throws UnimplementedFeatureException, DataSourceException {
	}

	public String getEntryPointVersion() 
			throws UnimplementedFeatureException, DataSourceException {
	}

	public int getTotalEntryPoints() 
			throws UnimplementedFeatureException, DataSourceException {
	}
        @Override
        public DasAnnotatedSegment getFeatures(String segmentId, Integer maxbins,
                        Range rows) throws BadReferenceObjectException,
                        DataSourceException, UnimplementedFeatureException {
                
        }

        @Override
        public Collection<DasAnnotatedSegment> getFeatures(
                        Collection<String> featureIdCollection, Integer maxbins, Range rows)
                        throws UnimplementedFeatureException, DataSourceException {
                
        }

}

```
Firstly, we should define the attributes of the class. The first three attributes (`svCon`, `globalParameters` and `config`) are basically keeping the external variables acquired in the `init()` in the local scope for potential requests. The `cacheManager` has the same purpose but its value is registered in the method `registerCacheManager`. `path` is the string with the path of the file to extract all the info of the data source. Finally, `segments` and `types` are the collections that contain the information of the datasource in memory.
```
	private ServletContext svCon;
        private Map<String, PropertyType> globalParameters;
        private DataSourceConfiguration config;
        private CacheManager cacheManager = null;
        private String path;
        private Collection<DasAnnotatedSegment> segments;
        private Collection<DasType> types;
```

Now lets go top down implementing all the methods of this Data Source.
  * **init()**: This method is called when the data source is loaded, so the goal of this method should be to prepare the data source to receive the queries; in our case it means to parse the file and keep the model in memory. We assume that the file to be loaded is going to be the data source property `pipes_file` in the configuration file.

> The parameters of this method have to be save as attributes of this class(you have to create those attributes). The parser is now invoked and the model(segments and types) is also saved as parameter. Exceptions are catch and throw in case something goes wrong.
```
	public void init(ServletContext servletContext,
                        Map<String, PropertyType> globalParameters,
                        DataSourceConfiguration dataSourceConfig)
                        throws DataSourceException {
                this.svCon = servletContext;
                this.globalParameters = globalParameters;
                this.config = dataSourceConfig;
                path = config.getDataSourceProperties().get("pipes_file").getValue();
                try {
                        SeparatedByPipesParser parser = new SeparatedByPipesParser(new FileInputStream(servletContext.getRealPath(path)));
                        segments = parser.parse();
                        types = parser.getTypes();
                } catch (FileNotFoundException e) {
                        throw new DataSourceException("The data source cannot be loaded. The file couldn't be oppened",e);
                } catch (Exception e) {
                        throw new DataSourceException("The data source cannot be loaded because of parsing problems",e);
                }
        }
```
  * **destroy()**: the goal of this method is to liberate the resources of the system whenever the data source is turned off. In our case there is not too much to do, maybe be sure the attributes of this class are null. which is not completely necessary because thats something java is going to do later anyway, however in other cases this method can be important to close databases connections or network sockets.
```
	public void destroy() {
		this.svCon=null;
		this.globalParameters=null;
		this.config=null;
		this.path=null;
		this.segments=null;
		this.types=null;
		this.cacheManager = null;
	}
```
  * **getFeatures()**: The first method to get Features, queries for the features of a segment id, giving that we already have an array of segments this method just requires to go though this array looking for the segment with the same id. In case the id is not in the array an exception should be thrown it, to let !MyDAS report this in the appropriate way.
> The attribute maxbins is ignored for now, this implies that our data source is not implementing that capability.
```
	public DasAnnotatedSegment getFeatures(String segmentId, Integer maxbins)
                        throws BadReferenceObjectException, DataSourceException {
                for(DasAnnotatedSegment segment:segments){
                        if (segment.getSegmentId().equals(segmentId))
                                return segment;
                }
                throw new BadReferenceObjectException("The id is not in the file", segmentId);
        }
```
  * **getFeatures()**: The second method to get features queries by the feature id. (a set of those to be more specific). to do that we loop though all the ids of the query looking on each segment for features with those Ids. If we found any, we create a copy of the segment, that contains just that feature. If the the feature is not found in any of the segments an `DasUnknownFeatureSegment` is added to the response array.
```
	public Collection<DasAnnotatedSegment> getFeatures(
                        Collection<String> featureIdCollection, Integer maxbins)
                        throws UnimplementedFeatureException, DataSourceException {
                Collection<DasAnnotatedSegment> segmentsResponse =new ArrayList<DasAnnotatedSegment>();
                for (String featureId:featureIdCollection){
                        boolean found=false;
                        for (DasAnnotatedSegment segment:segments){
                                for(DasFeature feature:segment.getFeatures())
                                        if(feature.getFeatureId().equals(featureId)){
                                                segmentsResponse.add(new DasAnnotatedSegment(   segment.getSegmentId(),
                                                                                                segment.getStartCoordinate(),
                                                                                                segment.getStopCoordinate(),
                                                                                                segment.getVersion(), 
                                                                                                segment.getSegmentLabel(),
                                                                                                Collections.singleton(feature)));
                                                found=true;
                                                break;
                                        }
                        }
                        if(!found)
                                segmentsResponse.add(new DasUnknownFeatureSegment(featureId));
                }
                return segmentsResponse;
        }
```

  * **getLinkURL()**: This method has been deprecated in DAS 1.6, so we won't implement it, and just throw an exception indicatiing that is not implemented.
```
	public URL getLinkURL(String field, String id)
			throws UnimplementedFeatureException, DataSourceException {
		throw new UnimplementedFeatureException("No implemented");
	}
```
  * **getTotalCountForType()**: This method is a helper function for the types command to be able to answer how many times a type has been used in this data source. in order to implement we have to go through all the features in all the segments counting how many times the queried type is there.
```
	public Integer getTotalCountForType(DasType type)
                        throws DataSourceException {
                int count=0;
                for (DasAnnotatedSegment segment:segments){
                        for(DasFeature feature:segment.getFeatures()){
                                if(type.getId().equals(feature.getType().getId()))
                                        count++;
}
}
                return count;
        }
```
  * **getTypes()**: This is really simple cause we already have this info in the model so we just have to return this array:
```
	public Collection<DasType> getTypes() throws DataSourceException {
		return types;
	}
```
  * **registerCacheManager()**: The mydas DAS server implements caching within the server.  This method passes your datasource a reference to a `uk.ac.ebi.mydas.controller.CacheManager` object.
```
	public void registerCacheManager(CacheManager cacheManager) {
		this.cacheManager = cacheManager;
	}
```
  * **Entry point methods**: The entry points capability is not going to be developed in this tutorial, therefore all the related methods should throw the correspondent exception like that:
```

	public Collection<DasEntryPoint> getEntryPoints(Integer start, Integer stop) throws UnimplementedFeatureException, DataSourceException {
		throw new UnimplementedFeatureException("No implemented");
	}
	
	public String getEntryPointVersion() throws UnimplementedFeatureException, DataSourceException {
		throw new UnimplementedFeatureException("No implemented");
	}
	
	public int getTotalEntryPoints() throws UnimplementedFeatureException, DataSourceException {
		throw new UnimplementedFeatureException("No implemented");
	}

```

 **rows methods**: The rows capability (a DAS 1.6E paginated response of the features command e.g. rows=1-10) is also not going to be developed in this tutorial, therefore the following method should throw the correspondent exception:

```

	@Override
        public DasAnnotatedSegment getFeatures(String segmentId, Integer maxbins,
                        Range rows) throws BadReferenceObjectException,
                        DataSourceException, UnimplementedFeatureException {
                throw new UnimplementedFeatureException("No implemented");
        }

        @Override
        public Collection<DasAnnotatedSegment> getFeatures(
                        Collection<String> featureIdCollection, Integer maxbins, Range rows)
                        throws UnimplementedFeatureException, DataSourceException {
                throw new UnimplementedFeatureException("No implemented");
        }


```

#### Configure MyDAS with the new Data Source ####

  * Get the file. In this example we will use the file located at http://mydas.googlecode.com/files/testGenes.txt. Download it and saved in the folder `[MyDasTemplate]/src/main/webapp/`
  * Stop the jetty server(if running) by pressing 'ctrl+c' on the terminal that is running.
  * Open the file `[MyDasTemplate]/src/main/webapp/MydasServerConfig.xml` with your favorite editor.
  * The recently created data source is very similar to the GFF example data source, So, copy and paste the configuration for the data source with uri = "examplegff", it should look like:
```
        <datasource uri="examplegff"
                    title="fickett-tss"
                    description="GFF2 Example Annotation Data Source"
                    doc_href="http://mydas.googlecode.com/svn/trunk/mydas/MyDasTemplate/src/main/resources/fickett-tss.gff"
                    mapmaster="http://mymapmaster.com">
			<maintainer email="youremail@server.com" />

			<version uri="examplegff" created="2010-03-19">
	            <class>uk.ac.ebi.mydas.examples.GFFFileDataSource</class>
				<coordinates 	uri="http://www.dasregistry.org/coordsys/CS_DS52"
								source="Gene_ID"
								authority="Ensembl"
								taxid="6239"
								test_range="ENSG00000160916">Ensembl,CEL_160,Chromosome,Caenorhabditis elegans</coordinates>
				<capability	type="das1:sources"
						query_uri="http://localhost:8080/das/examplegff" />
				<capability	type="das1:types"
						query_uri="http://localhost:8080/das/examplegff/types" />
				<capability	type="das1:features"
						query_uri="http://localhost:8080/das/examplegff/features" />
				<capability	type="das1:feature-by-id"
						query_uri="http://localhost:8080/das/examplegff/features" />
				<capability	type="das1:stylesheet"
						query_uri="http://localhost:8080/das/examplegff/stylesheet" />
				<capability	type="das1:unknown-feature"	/>
	    <property key="gff_file" value="CHROMOSOME_MtDNA.gff" />
			</version>

            <dna-command-enabled>true</dna-command-enabled>
            <features-strictly-enclosed>true</features-strictly-enclosed>
            <use-feature-id-for-feature-label>true</use-feature-id-for-feature-label>
            <include-types-with-zero-count>true</include-types-with-zero-count>
        </datasource>

```
  * Lets edit all those values:
    * Tag `<datasource>`, attribute `uri`: examplepipes
    * Tag `<datasource>`, attribute `title`: separatedByPipes
    * Tag `<datasource>`, attribute `description`: Data source testing a new parser
    * Tag `<datasource>`, attribute `doc_href`: http://mydas.googlecode.com
    * Tag `<maintainer>`, attribute `email`: `[Your email]`
    * Tag `<version>`, attribute `uri`: examplepipes
    * Tag `<version>`, attribute `created`: `Today`
    * Tag `<class>`, content: uk.ac.ebi.mydas.examples.SeparatedByPipesDataSource
    * Tag `<coordinates>`, attribute `uri`: `http://www.dasregistry.org/coordsys/CS_DS311`
    * Tag `<coordinates>`, attribute `source`: Chromosome
    * Tag `<coordinates>`, attribute `taxid`: 9606
    * Tag `<coordinates>`, attribute `Authority`: GRCh
    * Tag `<coordinates>`, content: GRCh\_37,Chromosome,Homo sapiens
    * Tag `<coordinates>`, attribute `test_range`: 5
    * Tag `<property>`, attribute `key`: pipes\_file
    * Tag `<property>`, attribute `value`: testGenes.txt
    * All query attributes of the capabilities elements should have an url starting for `http://localhost:8080/das/examplepipes/` instead of `http://localhost:8080/das/examplegff/`

If everything goes right you have created a new data source from scratch, so now you can start again the server using maven to compile the new files and using the new configuration.

You can check that the information display for MyDas corresponds to the one in the file. For instance you can try to go to the following links:
  * http://localhost:8080/das/examplepipes/features?segment=5
  * http://localhost:8080/das/examplepipes/features?feature_id=ENSG00000228416
  * http://localhost:8080/das/examplepipes/features?feature_id=ENST00000413363
  * http://localhost:8080/das/examplepipes/features?feature_id=ENSE00001747247

The final files can be directly download from:
  * http://mydas.googlecode.com/svn/trunk/mydas/MyDasTemplate/src/main/java/uk/ac/ebi/mydas/examples/SeparatedByPipesDataSource.java
  * http://mydas.googlecode.com/svn/trunk/mydas/MyDasTemplate/src/main/java/uk/ac/ebi/mydas/examples/SeparatedByPipesParser.java


[<<1st Tutorial(Beginners level)](MyDASTutorial.md) | **2nd Tutotial(Intermediate level)** | [3rd Tutorial(Advance level) >>](MyDASTutorial3rdPart.md)