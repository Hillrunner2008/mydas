# MyDas Tutorial - Other Capabilities #

In this tutorial we will explain how to implements other capabilities included in DAS but that are optional, and because of that have been ignored in the other tutorials. This tutorial assumes that you have completed the [Advanced Tutorial](MyDASTutorial3rdPart.md) some of the methods developed there are going to be reused here.

To start lets edit the configuration file to indicate that the ensemblTest data source implements `entry_points` and `maxbins` capabilities:
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
			</version>

            <dna-command-enabled>true</dna-command-enabled>
            <use-feature-id-for-feature-label>true</use-feature-id-for-feature-label>
            <include-types-with-zero-count>true</include-types-with-zero-count>
        </datasource>
```

## `entry points` Capability ##

Some times it is important to provide the ids of the segments that a data source hava annotations about. This is specially remarkable for reference servers, in order to bring an easy way to let the user/client to know what can be found in this data source. Usually the entry points are the Uniprot ids for protein data sources and chromosomes/contigs/genes for genomic data sources.

Developing an annotation data source implies to implement the methods `getEntryPoints`, `getEntryPointVersion` and `getTotalEntryPoints` as seen here:

```
	public Collection<DasEntryPoint> getEntryPoints(Integer start, Integer stop) 
		throws UnimplementedFeatureException, DataSourceException {
	}
	public String getEntryPointVersion() 
		throws UnimplementedFeatureException, DataSourceException {
	}
	public int getTotalEntryPoints() 
		throws UnimplementedFeatureException, DataSourceException {
	}
```

In the previous tutorial we have just throw the corresponding exception to inform that this capability have not been implemented. Here we will see that this implementation can be very simple. To start we can use the version of the database we are using as the version for the entry points, in other cases it may be use the date of publication, etc. so, in the `EnsemblTestManager.java` file we should put the database as a private String attribute and provide a public getter method for it:
```
	private String database = "homo_sapiens_core_56_37a";

	public String getDatabase(){
		return database;
	}
```
And in the file `EnsemblTestDataSource.java` the `getEntryPointVersion` method should be overwritten as:
```
	public String getEntryPointVersion() throws UnimplementedFeatureException, DataSourceException {
		return ensembl.getDatabase();
	}
```

The entry points of our data source are the chromosomes available. Besides the regular chromosomes(1-22 + X + Y) Ensembl database includes other structures of this level, for instance the mitochondrial chromosome. To get this list we have to query the `seq_region` in its chromosomal scope which is the coordinate system 2. The SQL is:
```
SELECT * FROM seq_region WHERE seq_region.coord_system_id =2;
```
This SQL will be located in the class `EnsemblTestManager.java` in a method called `getEntryPoints`. We will create a private attribute to store in memory the list of entry points, in this way the query will be executed just once, and the next time an entry point request is received can be obtained from memory. The code to add is:
```
	private Collection<DasEntryPoint> entryPoints=null;

	public Collection<DasEntryPoint> getEntryPoints() throws DataSourceException{
		if (entryPoints!=null){
			if ((start != null) && (stop != null)) 
				return ((ArrayList<DasEntryPoint>)entryPoints).subList(start, stop);
			return entryPoints;
		}
		String sql="";
		sql="SELECT * FROM seq_region WHERE seq_region.coord_system_id =2;";
		try {
			Statement s = connection.createStatement ();
			s.executeQuery (sql);
			ResultSet rs = s.getResultSet ();
			entryPoints = new ArrayList<DasEntryPoint>();
			while (rs.next ()) {
				entryPoints.add(new DasEntryPoint(
					rs.getString("name"), 
					1, rs.getInt("length"), 
					"Chromosome", 
					getDatabase(), 
					DasEntryPointOrientation.NO_INTRINSIC_ORIENTATION, 
					"Chromosome", 
					true));
			}
			rs.close ();
			s.close ();
		} catch (SQLException e) {
			throw new DataSourceException("Problems executing the sql query",e);
		}
		if ((start != null) && (stop != null)) 
			return ((ArrayList<DasEntryPoint>)entryPoints).subList(start, stop);
		return entryPoints;
	}
```
Instances of the entry points are being created at the same moment to added in the list in the line:
```
entryPoints.add(new DasEntryPoint(
	rs.getString("name"), 
	1, rs.getInt("length"), 
	"Chromosome", 
	getDatabase(), 
	DasEntryPointOrientation.NO_INTRINSIC_ORIENTATION, 
	"Chromosome", 
	true));
```
We are calling the constructor of the DasEntryPoint providing the next values:
  * segmentId -> `rs.getString("name")` : value obtained from the DB for the name of the chromosome.
  * startCoordinate -> 1 : we assume all the chromosomes starts in the position 1. It might be a false assumption in the case of the clone chromosomes or cases like that.
  * stopCoordinate -> `rs.getInt("length")` : given the previous assumption the stop coordinate is equivalent to the length of the chromosome.
  * type -> `"Chromosome"` : We are not specifying the types of chromosomes that we are returning
  * version -> `getDatabase()` : Re-using the method to get the database that we are using which implies the version of Ensemble release that we are using.
  * orientation -> `DasEntryPointOrientation.POSITIVE_ORIENTATION` : just to express that at chromosomal level the sequence is always given in the positive direction.
  * description -> `"is a chromosome"` : free description of the entry point, could be more elaborated and include the id and its coordinates.
  * hasSubparts -> `true` : just to recall the fact that when we define this data source we define the contigs as parts of the chromosome.

And then, in the data source we just have to call this method in the correspondent method:
```
	public Collection<DasEntryPoint> getEntryPoints(Integer start, Integer stop) throws UnimplementedFeatureException, DataSourceException {
		return ensembl.getEntryPoints(start,stop);
	}
```
The last method we have to implement is getTotalEntryPoints, which is specially important in data sources with many entry points because MyDas uses this method for validation of the range for the pagination of entry points. An easy way to implement this in our data source is:
```
	public int getTotalEntryPoints() throws UnimplementedFeatureException, DataSourceException {
		return ensembl.getEntryPoints(null,null).size();
	}
```
Now we can restart the server and check the capability working:
  * http://localhost:8080/das/ensemblTest/entry_points
  * http://localhost:8080/das/ensemblTest/entry_points?rows=10-15

## `maxbins` Capability ##

Often a DAS client is limited to display a number of features when in reality a segment can have a lot more; think for example, in a client displaying the chromosome 1 and wanted to show its genes as its annotations, there are around 4200 genes in this chromosome, a popular screen configuration have 1280x1024 pixeles, therefore it is physically impossible to draw all the genes in the same track for this chromosome.
The DAS capability **maxbins** allow to request a X number of the most representative features in a segment.
The problem is that just the owner of the data source can define what 'representative' means for their data. For instance in the case of the genes, a medical related database might want to prioritize genes connected with a disease, but a different data source can define its discriminating criterion in a very different way.

As seen in previous tutorials the methods to implement in MyDas in order to get the features of a segment are defined as:
```
	public DasAnnotatedSegment getFeatures(String segmentId, Integer maxbins)
		throws BadReferenceObjectException, DataSourceException {
	}

	public Collection<DasAnnotatedSegment> getFeatures(Collection<String> featureIdCollection, Integer maxbins)
		throws UnimplementedFeatureException, DataSourceException {
	}

```

In both cases there is an Integer attribute called `maxbins`, this value indicates how many features the client wants to receive for the requested segment, and again is decision of the provider how to choose the response set.

The arbitrary criterion chosen for this example is the size of the genes, to do this we will order the results of the SQL by the size wich is calculated as the difference between the start and stop coordinates in the SQL itself, so now the `getSubmodelBySegmentId` method will receive an extra parameter for maxbins and can be implemented as:
```
	public DasAnnotatedSegment getSubmodelBySegmentId(String segmentId, int start, int stop, int maxbins) throws DataSourceException, BadReferenceObjectException{
		String sql="SELECT " +
		" sr.name AS chr, " +
		" gsi.stable_id AS gene_id, " +
		" g.seq_region_start AS gene_start, " +
		" g.seq_region_end AS gene_end, " +
		" tsi.stable_id AS trascript_id, " +
		" t.seq_region_start AS transcript_start, " +
		" t.seq_region_end AS transcript_end, " +
		" esi.stable_id AS exon_id, " +
		" e.seq_region_start AS exon_start, " +
		" e.seq_region_end AS exon_end, " +
		"(g.seq_region_end-g.seq_region_start) AS size" +
		"FROM  " +
		" seq_region sr, " +
		" gene_stable_id gsi, " +
		" gene g, " +
		" transcript t, " +
		" transcript_stable_id tsi, " +
		" exon_transcript et, " +
		" exon e, " +
		" exon_stable_id esi  " +
		"WHERE  " +
		" gsi.gene_id = g.gene_id and " + 
		" g.gene_id = t.gene_id and  " +
		" t.transcript_id = tsi.transcript_id and " + 
		" t.transcript_id = et.transcript_id and  " +
		" et.exon_id = e.exon_id and  " +
		" e.exon_id = esi.exon_id and  " +
		" g.seq_region_id = sr.seq_region_id and " +
		" sr.coord_system_id = 2 and " +
		" sr.name ='"+segmentId+"' ";
		if (start!=-1 && stop!=-1)
			sql += " and g.seq_region_start>"+start+" and g.seq_region_end<"+stop;
		sql += " ORDER BY size";
		Collection<DasAnnotatedSegment>segments= getSubmodelBySQL(sql,maxbins);
		if (segments!=null && segments.size()>0)
			return segments.iterator().next();
		else
			throw new BadReferenceObjectException("Unknown Chromosome", segmentId);
	}
```
For compatibility reasons with previous tutorials we can add a method with no `maxbins` attribute that will call the above method with a value that means that there is not maxbins value (-1)
```
	public DasAnnotatedSegment getSubmodelBySegmentId(String segmentId, int start, int stop) throws DataSourceException, BadReferenceObjectException{
		return getSubmodelBySegmentId(segmentId, start, stop,-1);
	}
```
Using this methods the results are going to be ordered, how ever we haven't filtered to use the desired number of features, in order to do that we can change the `getSubmodelBySQL` method by restricting the loop to the number of maxbins:
```
	public Collection<DasAnnotatedSegment> getSubmodelBySQL(String sql,int maxbins) throws DataSourceException{
		Collection<DasAnnotatedSegment> segments=null;
		try {
			Statement s = connection.createStatement ();
			s.executeQuery (sql);
			ResultSet rs = s.getResultSet ();
			DasComponentFeature previousGene=null;
			while (rs.next () && maxbins!=0) {
				if (segments==null)
					segments= new ArrayList<DasAnnotatedSegment>();
				DasAnnotatedSegment segment = this.getSegment(segments,rs.getString ("chr"));
				DasComponentFeature gene= this.getGene(rs.getString ("gene_id"),rs.getInt("gene_start"),rs.getInt("gene_end"),segment);
				if (previousGene!=gene){
					maxbins--;
					previousGene = gene;
				}

				DasComponentFeature transcript= this.getTranscript(rs.getString ("trascript_id"),rs.getInt("transcript_start"),rs.getInt("transcript_end"),gene);
				this.getExon(rs.getString ("exon_id"),rs.getInt("exon_start"),rs.getInt("exon_end"),transcript);
			}
			rs.close ();
			s.close ();
		} catch (SQLException e) {
			throw new DataSourceException("Problems executing the sql query",e);
		}
		return segments;
	}
```
We are here taking advantage of the selected value for no-maxbins which is -1 and in this loop will be negative without reaching the stopping condition (been equal to zero), however if a maxbins number is given it will be used a descending counter.

Now is just a matter of passing the maxbins parameter in the data source, so, open the file `EnsemblTestDataSource.java` and replace the `getFeatures` methods for:
```
	public DasAnnotatedSegment getFeatures(String segmentId, int start, int stop, Integer maxbins) 
	throws BadReferenceObjectException, CoordinateErrorException, DataSourceException {
		if (maxbins==null)
			maxbins=-1;
		return ensembl.getSubmodelBySegmentId(segmentId, start, stop,maxbins);
	}
	public DasAnnotatedSegment getFeatures(String segmentId, Integer maxbins) 
	throws BadReferenceObjectException, DataSourceException {
		if (maxbins==null)
			maxbins=-1;
		return ensembl.getSubmodelBySegmentId(segmentId, -1, -1,maxbins);
	}

```

So after restarting the server you will be able to query using maxbins like:
  * http://localhost:8080/das/ensemblTest/features?segment=5;maxbins=2
  * http://localhost:8080/das/ensemblTest/features?segment=5;maxbins=5
  * http://localhost:8080/das/ensemblTest/features?segment=5;maxbins=10