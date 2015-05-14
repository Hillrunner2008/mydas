# MyDas Tutorial - Reference Data Sources #

In this tutorial we will explain how to create a reference data source.This tutorial assumes that you are familiarized with the basic concepts of implementing data sources in MyDas. It will reuse the same example as in the [Basic Tutorial](MyDASTutorial.md).


## A Reference Server ##

MyDas also allows to implement a data source that provides the sequence as a reference for other data sources, including itself because in MyDas a reference server it is by inheritance an annotation data source.

in the [Basic Tutorial](MyDASTutorial.md) we used the datasource [GFFFileDataSource.java](http://mydas.googlecode.com/svn/trunk/mydas/MyDasTemplate/src/main/java/uk/ac/ebi/mydas/examples/GFFFileDataSource.java), which used the parser [GFF2Parser.java](http://mydas.googlecode.com/svn/trunk/mydas/MyDasTemplate/src/main/java/uk/ac/ebi/mydas/examples/GFF2Parser.java) to process a GFF file and distribute it through DAS. Here we will extend that data source to provide a reference for the annotated sequences.

### The Parser ###

The sequences will be provided in a !Fasta file and therefore we have to start by processing that kind of files. In order to do that lets create a parser class for that kind of file. First create a file called FastaParser in the same package than the rest of the tutorial classes. This parser will read the file and for each sequence it will create an instance of DasSequence, and the set will be store as a hash table using the Ids as the keys, to make easy any future access.

The constructor of the class is going to receive and InputStream with the content of the file and the file name for versioning purposes, it will also initializes a file Scanner to read the file and the HashMap to store the sequences:
```
	private Scanner scanner;
	private Map<String,DasSequence> sequences;
	private String fileName;
	public FastaParser(InputStream fastaDoc, String fileName){
		this.fileName=fileName;
		scanner= new Scanner(fastaDoc);
		sequences =new HashMap<String,DasSequence>();
	}
```
To start the parsing process creates a method like this:
```
	public Map<String,DasSequence> parse() throws Exception{
		this.processLineByLine();
		return sequences;
	}
```
The `processLineByLine` just uses the scanner to go line by line calling `processLine` with the content of each line:
```
	private final void processLineByLine() throws Exception{
		try {
			while ( scanner.hasNextLine() ){
				processLine( scanner.nextLine() );
			}
		} finally {
			scanner.close();
		}
	}
```
The key of this parsing is in the `processLine` method. Here we classify he lines into 2, the ones starting by `>` and the rest. if is not starting by `>` means that the line is part of the current sequence, so we just concat it to a private String called `currentSequence`.
In other case, there are also 2 cases: The first `>` of the whole file or subsequent ones, for the first sequence we just store in memory the currentHeader and initialize the currentSequence to the empty String. ad then every time it founds a new sequence(line starting by`>`) it uses the temporary header and sequence to create a DasSequence and added to the HashMap:
```
	private String currentSequence=null;
	private String currentHeader=null;
	private void processLine(String aLine) throws Exception{
		if (aLine.startsWith(">")){
			if (currentSequence!=null){
				String id=currentHeader.split(" ")[0];
				sequences.put(id,new DasSequence(id, currentSequence, 1, fileName, currentHeader));
			}
			currentHeader=aLine.substring(1).trim();
			currentSequence="";
		}else 
			currentSequence+=aLine.trim();
	}
```

After the whole file has been process the last sequence and header are still in memory and haven't been added to the HashMap, therefore the `processLineByLine` have to be edited to add this sequence:
```
	private final void processLineByLine() throws Exception{
		try {
			while ( scanner.hasNextLine() ){
				processLine( scanner.nextLine() );
			}
			String id=currentHeader.split(" ")[0];
			sequences.put(id,new DasSequence(id, currentSequence, 1, fileName, currentHeader));
		} finally {
			scanner.close();
		}
	}
```

### The Data Source ###

Copy the [Basic Tutorial](MyDASTutorial.md) we used the datasource [GFFFileDataSource.java](http://mydas.googlecode.com/svn/trunk/mydas/MyDasTemplate/src/main/java/uk/ac/ebi/mydas/examples/GFFFileDataSource.java) in the same `examples` folder and name it `GFFFileReferenceDataSource.java`. Change the !Interfase to implements from `AnnotationDataSource` to `ReferenceDataSource`. That change force us to implement the method `getSequence` but to be able to return the sequence we have to have process the fasta file first, so at the end of the `init` method we have to add a code like:
```
		try {
			FastaParser parser2 = new FastaParser(new FileInputStream(servletContext.getRealPath(path2)),path2);
			sequences = parser2.parse();
		} catch (FileNotFoundException e) {
			throw new DataSourceException("The reference data source cannot be loaded. The fasta file couldn't be oppened",e);
		} catch (Exception e) {
			throw new DataSourceException("The reference data source cannot be loaded because of parsing problems with the fasta file",e);
		}
```
this is assuming that the path of the fasta file is been provided in the same way that the gff file, i.e. a property in the configuration file whilst defining the data source. So lets add the corresponding data source to the config file:
```
        <datasource uri="referencegff"
                    title="fickett-tss"
                    description="GFF2 Example Annotation Data Source"
                    doc_href="http://mydas.googlecode.com/svn/trunk/mydas/MyDasTemplate/src/main/resources/fickett-tss.gff"
                    mapmaster="http://mapmaster_referencegff.com">
			<maintainer email="gsalazar@ebi.ac.uk" />

			<version uri="referencegff" created="2010-03-19">
	            <class>uk.ac.ebi.mydas.examples.GFFFileReferenceDataSource</class>
				<coordinates 	uri="http://www.notARealCoordinateSystem.com"
								source="-"
								authority="unknown"
								taxid="0000"
								test_range="L47615.1">TEST INFO</coordinates>
				<capability type="das1:sources"		 	query_uri="http://localhost:8080/das/examplegff" />
				<capability type="das1:types"			query_uri="http://localhost:8080/das/examplegff/types" />
				<capability type="das1:features"		query_uri="http://localhost:8080/das/examplegff/features" />
				<capability type="das1:feature-by-id"	query_uri="http://localhost:8080/das/examplegff/features" />
				<capability type="das1:stylesheet"		query_uri="http://localhost:8080/das/examplegff/stylesheet" />
				<capability type="das1:sequence"		query_uri="http://localhost:8080/das/examplegff/sequence" />
				<capability type="das1:unknown-feature"	/>
				<capability     type="das1:entry_points"        />
				<property key="gff_file" value="fickett-tss.gff" visibility="false"/>
				<property key="fasta_file" value="fickett.fasta" visibility="false"/>
			</version>

            <dna-command-enabled>true</dna-command-enabled>
            <use-feature-id-for-feature-label>true</use-feature-id-for-feature-label>
            <include-types-with-zero-count>true</include-types-with-zero-count>
        </datasource>
```
The fickett.fasta file canbe download from [Here](http://mydas.googlecode.com/svn/trunk/mydas/MyDasTemplate/src/main/webapp/fickett.fasta) and should be saved in the `[MyDasTemplate]/src/main/webapp/` folder.

Now we can implement the `getSequence` method. Here we just made use of the HashMap `sequences` loaded in the `init` method and we query it using the parameter `segmentId` if there is not a key with that id an exception will be thrown, otherwise the sequence is returned.
```
	private Map<String,DasSequence> sequences;
	public DasSequence getSequence(String segmentId) throws BadReferenceObjectException, DataSourceException {
		DasSequence seq=sequences.get(segmentId);
		if (seq==null)
			throw new BadReferenceObjectException("",segmentId);
		return seq;
	}
```

Taking advantage of having process the fasta file we can also provide the entry point capability:
```
	public Collection<DasEntryPoint> getEntryPoints(Integer start, Integer stop) throws DataSourceException {
		ArrayList<DasEntryPoint> entryPoints = new ArrayList<DasEntryPoint>();
		for (String id:sequences.keySet()) {
			DasSequence seq= sequences.get(id);
			entryPoints.add(
					new DasEntryPoint(
							id, seq.getStopCoordinate(), seq.getStartCoordinate(), "DNA", "1.0",
							null,
							seq.getLabel(), false));
		}
		if ((start != null) && (stop != null)) 
			return entryPoints.subList(start, stop);

		return entryPoints;
	}
	public int getTotalEntryPoints() throws DataSourceException {
		return sequences.size();
	}
	public String getEntryPointVersion() throws DataSourceException {
		return "1.0"; 
	}
```

Now you can check that everything is working by running the jetty server again, and checking the next URLs:
  * http://localhost:8080/das/referencegff/sequence?segment=AF107208.1
  * http://localhost:8080/das/referencegff/sequence?segment=U54701.1:10,20
  * http://localhost:8080/das/referencegff/entry_points
  * http://localhost:8080/das/referencegff/entry_points?rows=2-5