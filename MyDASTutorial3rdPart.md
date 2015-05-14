# MyDAS 1.6 Tutorial #

## Third Part: Developing a Data Source from a database ##

Now that you have the basis to create a data source in MyDAS, let start with a real  scenario in bioinformatics. Well. actually we are going to use the same kind of data of the previous tutorial, but now we are going to take the information from the original source: Ensembl.

Ensembl uses MySQL relational databases to store its information in the server ensembldb.ensembl.org, which you can access as user `anonymous`. [http://www.ensembl.org/info/data/mysql.html ]

For this example we are going to use the database `homo_sapiens_core_56_37a`. This database contains assembly information the generate top-level sequences like chromosomes, the non-redundant set of Ensembl gene, transcript and protein models annotated, Microarray probe set annotation and more information about the human genome.

To simplify our example we will use the same subset of our previous tutorial. It means, Chromosome, Genes(id,start and end), Transcripts(id,start and end) and Exons(id,start and end). The SQL query to get this info from the data base is:
```
SELECT 
 sr.name,
 gsi.stable_id,
 g.seq_region_start,
 g.seq_region_end,
 tsi.stable_id,
 t.seq_region_start,
 t.seq_region_end,
 esi.stable_id,
 e.seq_region_start,
 e.seq_region_end 
FROM 
 seq_region sr,
 gene_stable_id gsi,
 gene g,
 transcript t,
 transcript_stable_id tsi,
 exon_transcript et,
 exon e,
 exon_stable_id esi 
WHERE 
 gsi.gene_id = g.gene_id and 
 g.gene_id = t.gene_id and 
 t.transcript_id = tsi.transcript_id and 
 t.transcript_id = et.transcript_id and 
 et.exon_id = e.exon_id and 
 e.exon_id = esi.exon_id and 
 g.seq_region_id = sr.seq_region_id 
 and sr.coord_system_id = 2
```

That query will give us 982190 rows, the file used in the previous example was a subset of the first 1000 rows of this query and just to have a rough idea that file occupied 98Kb, that implies that the whole database in that file can be approximately 100Mb. And therefore an approach of putting the whole model in memory and quering it from there is not adequate, moreover, the fact that counting with the possibility of having programmatic access to it, is something to take advantage of.

### Database Manager ###

Now lets create a java class to deal with all the transactions to the Ensembl database.
  1. We require the MySQL driver for java. We can add this dependency to the pom file of our maven project, and maven will deal to get the files. This can be done by editing the file `[MyDasTemplate]/pom.xml` and adding the following code nested to the tag `<dependencies>`:
```
		<dependency>
			<groupId>mysql</groupId>
			<artifactId>mysql-connector-java</artifactId>
			<version>5.1.12</version>
		</dependency>
```
  1. In the folder `[MyDasTemplate]/src/main/java/uk/ac/ebi/mydas/examples/` create a file called: `EnsemblTestManager.java`.
  1. The initial content of this file is the empty class with the package definition, the packages to import and the attributes of this class.
```
package uk.ac.ebi.mydas.examples;

import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.sql.Statement;
import java.util.ArrayList;
import java.util.Collection;
import java.util.HashSet;

import uk.ac.ebi.mydas.exceptions.BadReferenceObjectException;
import uk.ac.ebi.mydas.exceptions.DataSourceException;
import uk.ac.ebi.mydas.model.*;

public class EnsemblTestManager {
        private ArrayList<DasType> types;
        private DasType geneType,transcriptType,exonType;
        private DasMethod method;
        private Connection connection;
```
  1. The constructor should connect to the database and let it open for next queries. As in the previous tutorial we are going create the types here, because we already know them.
```
	public EnsemblTestManager() throws DataSourceException{
                //Initialize types
                geneType= new DasType("Gene", null, "SO:0000704", "Gene");
                transcriptType= new DasType("Transcript", null, "SO:0000673", "Transcript");
                exonType= new DasType("Exon", null, "SO:0000147", "Exon");
                types= new ArrayList<DasType>();
                types.add(geneType);
                types.add(transcriptType);
                types.add(exonType);
                method = new DasMethod("combinatorial analysis used in automatic assertion","combinatorial analysis used in automatic assertion","ECO:0000213");


                connection = null;
                String userName = "anonymous";
                String password = "";
                String url = "jdbc:mysql://ensembldb.ensembl.org:5306/homo_sapiens_core_56_37a";
                try {
                        Class.forName ("com.mysql.jdbc.Driver").newInstance ();
                        connection = DriverManager.getConnection (url, userName, password);
                } catch (InstantiationException e) {
                        throw new DataSourceException("Problems loading the MySql driver",e);
                } catch (IllegalAccessException e) {
                        throw new DataSourceException("Problems loading the MySql driver",e);
                } catch (ClassNotFoundException e) {
                        throw new DataSourceException("Problems loading the MySql driver",e);
                } catch (SQLException e) {
                        throw new DataSourceException("Problems conecting to the ensembl database",e);
                }
        }  
```
  1. As this connection cannot be open forever we should provide a method to close it
```
	public void close(){
		try{
			connection.close ();
		} catch (Exception e) { /* ignore close errors */ }

	}
```
  1. To facilitate the work when we will implementing the data source the method of this class will return objects following the model of MyDAS and the strategy to do this is very similar to the one in [MyDASTutorial2ndPart](MyDASTutorial2ndPart.md), the difference is that now the info is been recover from a database query:
```
	public Collection<DasAnnotatedSegment> getSubmodelBySQL(String sql) 
                        throws DataSourceException{
                Collection<DasAnnotatedSegment> segments=null;
              
                        try {
                                Statement s = connection.createStatement ();
                                s.executeQuery (sql);
                                ResultSet rs = s.getResultSet ();
                                while (rs.next ()) {
                                        if (segments==null)
                                                segments= new ArrayList<DasAnnotatedSegment>();
                                        DasAnnotatedSegment segment = this.getSegment(segments,rs.getString ("chr"));
                                        DasFeature gene= this.getFeature( rs.getString ("gene_id"),
                                                                                rs.getInt("gene_start"),
                                                                                rs.getInt("gene_end"),
                                                                                geneType, null, segment);
                                        
                                        DasFeature transcript= this.getFeature(     rs.getString ("trascript_id"), 
                                                                                                rs.getInt("transcript_start"), 
                                                                                                rs.getInt("transcript_end"), 
                                                                                                transcriptType,gene, segment);
                                        gene.getParts().add(transcript.getFeatureId());
                                        DasFeature exon=this.getFeature(   rs.getString ("exon_id"), 
                                                        rs.getInt("exon_start"), 
                                                        rs.getInt("exon_end"), 
                                                        transcriptType, transcript, segment);
                                     
                                          transcript.getParts().add(exon.getFeatureId());
                                }
                                rs.close ();
                                s.close ();
                        } catch (SQLException e) {
                                throw new DataSourceException("Problems executing the sql query",e);
                        }
                return segments;
        }
 
```
  1. The method `getFeature()` is similar to [MyDASTutorial2ndPart](MyDASTutorial2ndPart.md), it's easier because here we will receive the start and end as integers but slightly more complicated as we have to take into account multiple segments (not just Chromosome 5) and so we pass the segment in as a parameter to both the getFeature and getFeatureWithId method:
```
	private DasFeature getFeature(    String featureId,
                                                int start,
                                                int stop, DasType type, DasFeature parent, DasAnnotatedSegment segment) 
        throws DataSourceException {
            //return the feature if it already exists
                DasFeature feature=this.getFeatureWithId(featureId, segment);
                if(feature!=null)return feature;
           
                
                DasFeatureOrientation orientation= DasFeatureOrientation.ORIENTATION_SENSE_STRAND;
		
		DasPhase phase= DasPhase.PHASE_NOT_APPLICABLE;
		Double score=null;
                HashSet parents=new HashSet();
                if(parent!=null){
                parents.add(parent.getFeatureId());
                }
                DasFeature gene=new DasFeature(featureId,featureId,type, method, start, stop , score,orientation,phase, null, null, null, parents, new HashSet());
               // System.out.println("adding gene:"+gene);
                segment.getFeatures().add(gene);
                return gene;
        }

private DasFeature getFeatureWithId(String featureId, DasAnnotatedSegment segment){
            for(DasFeature feature:segment.getFeatures()){
                if(feature.getFeatureId().equals(featureId)){
                    return feature;
                }
            }
            return null;
        }
```
  1. The method `getSegment()` has a modification to be able to capture the length of the segment from the database. To do this we created a private method to query the length for the specific segment.
```

	private DasAnnotatedSegment getSegment(Collection<DasAnnotatedSegment> segments,String segmentId) throws DataSourceException {
                for (DasAnnotatedSegment segment:segments)
                        if (segment.getSegmentId().equals(segmentId))
                                return segment;
                Integer length = this.getSegmentLength(segmentId);
                DasAnnotatedSegment newSegment = new DasAnnotatedSegment(segmentId,1,length,"1.0",segmentId, new ArrayList<DasFeature>());
                segments.add(newSegment);
                return newSegment;
        }
        private Integer getSegmentLength(String segmentId){
                String sql="SELECT length FROM seq_region WHERE name='"+segmentId+"' and coord_system_id=2";
                try{
                        Statement s = connection.createStatement ();
                        s.executeQuery (sql);
                        ResultSet rs = s.getResultSet ();
                        while (rs.next ()) 
                                return rs.getInt ("length");
                } catch (SQLException e) {
                        return 1;
                }
                return 1;
        }

```
  1. Now we have to built the SQL for the different queries. For the one that queries a segment we will allow to specify a start and end point to get the features of a part of a chromosome and not necessarily the whole chromosome, to do this, we add an extra condition to the `WHERE` clause of the original SQL. In the case that the request is for the whole chromosome, we define a condition where the `start` and `end` parameters should be equal to -1.
```
	public DasAnnotatedSegment getSubmodelBySegmentId(String segmentId, int start, int stop) 
                        throws DataSourceException, BadReferenceObjectException{
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
                " e.seq_region_end AS exon_end " +
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
                        sql += " and g.seq_region_start<="+stop+" and g.seq_region_end>="+start;
                Collection<DasAnnotatedSegment>segments= getSubmodelBySQL(sql);
                if (segments!=null && segments.size()>0)
                        return segments.iterator().next();
                else
                        throw new BadReferenceObjectException("Unknown Chromosome", segmentId);
        }
```
  1. A similar method to get features by their id is required. In this case the addition to the SQL should be looking for the id in 3 different fields(for gene, transcript and exon). the method should be something like:
```
	public Collection<DasAnnotatedSegment> getSubmodelByFeatureId(Collection<String> featureIdCollection) throws DataSourceException{
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
                " e.seq_region_end AS exon_end " +
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
                " sr.coord_system_id = 2 and (";
                String or ="";
                for (String featureId:featureIdCollection){
                        sql +=  or +" gsi.stable_id = '"+featureId + 
                                "' or tsi.stable_id = '"+featureId + 
                                "' or esi.stable_id = '"+featureId+ "'" ;
                        or ="or";
                }
                sql += ")";
                return getSubmodelBySQL(sql);
        }
```
  1. As in the previous tutorial the list of types is predefined, and therefore, the method to get types is as simple as returning that array:
```
	public ArrayList<DasType> getTypes() {
		return types;
	}
```
  1. Finally we require to count how many annotations of a specific type are. For that reason we query the database counting for entryes of different ids for each type:
```
	public Integer getTotalCountForType(String typeId)
                        throws DataSourceException {
                String sql="";
                Integer count =0;
                if (typeId.equalsIgnoreCase("Gene"))
                        sql="SELECT count(stable_id) as num FROM gene_stable_id;";
                else if (typeId.equalsIgnoreCase("Transcript"))
                        sql="SELECT count(stable_id) as num FROM transcript_stable_id;";
                else if (typeId.equalsIgnoreCase("Exon"))
                        sql="SELECT count(stable_id) as num FROM exon_stable_id;";
                try {
                        Statement s = connection.createStatement ();
                        s.executeQuery (sql);
                        ResultSet rs = s.getResultSet ();
                        if (rs.next ()) {
                                count=rs.getInt("num");
                        }
                        rs.close ();
                        s.close ();
                } catch (SQLException e) {
                        throw new DataSourceException("Problems executing the sql query",e);
                }
                return count;
        }
```

### The Data Source ###
You can follow similar steps to create the data source as in [MyDASTutorial2ndPart](MyDASTutorial2ndPart.md). The main differences here is that the model is not generated at the moment of initialization of the data source; instead of that, a sub model is generated for every request. The other change is that now we are going to implement the MyDAS interface `RangeHandlingAnnotationDataSource`, to be able to deal with start and end positions of the segment.
  1. In the folder `[MyDasTemplate]/src/main/java/uk/ac/ebi/mydas/examples/` create a file called: `EnsemblTestDataSource.java`.
  1. The empty implementation of the interface `RangeHandlingAnnotationDataSource` should be something like:
```
package uk.ac.ebi.mydas.examples;

import java.net.URL;
import java.util.ArrayList;
import java.util.Collection;
import java.util.Collections;
import java.util.Map;

import javax.servlet.ServletContext;

import uk.ac.ebi.mydas.configuration.DataSourceConfiguration;
import uk.ac.ebi.mydas.configuration.PropertyType;
import uk.ac.ebi.mydas.controller.CacheManager;
import uk.ac.ebi.mydas.datasource.RangeHandlingAnnotationDataSource;
import uk.ac.ebi.mydas.exceptions.BadReferenceObjectException;
import uk.ac.ebi.mydas.exceptions.CoordinateErrorException;
import uk.ac.ebi.mydas.exceptions.DataSourceException;
import uk.ac.ebi.mydas.exceptions.UnimplementedFeatureException;
import uk.ac.ebi.mydas.extendedmodel.DasUnknownFeatureSegment;
import uk.ac.ebi.mydas.model.DasAnnotatedSegment;
import uk.ac.ebi.mydas.model.DasComponentFeature;
import uk.ac.ebi.mydas.model.DasFeature;
import uk.ac.ebi.mydas.model.DasType;
import uk.ac.ebi.mydas.model.DasEntryPoint;
import uk.ac.ebi.mydas.model.Range;
import uk.ac.ebi.mydas.configuration.PropertyType;


public class EnsemblTestDataSource implements RangeHandlingAnnotationDataSource{
        CacheManager cacheManager = null;
        ServletContext svCon;
        Map<String, PropertyType> globalParameters;
        DataSourceConfiguration config;
        EnsemblTestManager ensembl;

	public void init(ServletContext servletContext,
			Map<String, PropertyType> globalParameters,
			DataSourceConfiguration dataSourceConfig)
			throws DataSourceException {
	}

	public void destroy() {
	}

	public DasAnnotatedSegment getFeatures(String segmentId, int start, int stop, Integer maxbins) 
			throws BadReferenceObjectException, CoordinateErrorException, DataSourceException {
	}

	public DasAnnotatedSegment getFeatures(String segmentId, Integer maxbins) 
			throws BadReferenceObjectException, DataSourceException {
		return null;
	}

	public Collection<DasAnnotatedSegment> getFeatures(Collection<String> featureIdCollection, Integer maxbins)
			throws UnimplementedFeatureException, DataSourceException {
		return null;
	}

	public Collection<DasType> getTypes() throws DataSourceException {
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

	public void registerCacheManager(CacheManager cacheManager) {
	}

}
```
  1. The constructor should create an instance of the manager and save locally the parameters:
```
	public void init(ServletContext servletContext,
                        Map<String, PropertyType> globalParameters,
                        DataSourceConfiguration dataSourceConfig)
                        throws DataSourceException {
                this.svCon = servletContext;
                this.globalParameters = globalParameters;
                this.config = dataSourceConfig;
                ensembl=new EnsemblTestManager();
        }
```
  1. The destroy method will close the database connection of the ensemble manager:
```
	public void destroy() {
		ensembl.close();
	}
```
  1. The response for a features command that does specify the start and end of the segment will return the submodel generated by the ensembleManager class:
```
	public DasAnnotatedSegment getFeatures(String segmentId, int start, int stop, Integer maxbins) 
                        throws BadReferenceObjectException, CoordinateErrorException, DataSourceException {
                return ensembl.getSubmodelBySegmentId(segmentId, start, stop);
        }
```
  1. In the case that method does not specifythe coordinates, we can call the same method but given the value of -1 to those parameters
```
	public DasAnnotatedSegment getFeatures(String segmentId, Integer maxbins) 
                        throws BadReferenceObjectException, DataSourceException {
                return ensembl.getSubmodelBySegmentId(segmentId, -1, -1);
        }
```
  1. In the case that the features are requested by feature\_id, the submodel is requested to the ensembl and then processed in the same way of the previous tutorial([MyDASTutorial2ndPart](MyDASTutorial2ndPart.md)):
```
	public Collection<DasAnnotatedSegment> getFeatures(Collection<String> featureIdCollection, Integer maxbins)
                        throws UnimplementedFeatureException, DataSourceException {
                Collection<DasAnnotatedSegment> segments = ensembl.getSubmodelByFeatureId(featureIdCollection);
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
  1. The last methods have nothing special and follow the same reasoning that the previous tutorial([MyDASTutorial2ndPart](MyDASTutorial2ndPart.md)):
```
	public Collection<DasType> getTypes() throws DataSourceException {
                return ensembl.getTypes();
        }
        public URL getLinkURL(String field, String id)
                        throws UnimplementedFeatureException, DataSourceException {
                throw new UnimplementedFeatureException("Not implemented");
        }

        public Integer getTotalCountForType(DasType type)
                        throws DataSourceException {
                return ensembl.getTotalCountForType(type.getId());
        }
        public void registerCacheManager(CacheManager cacheManager) {
                this.cacheManager = cacheManager;
        }
        public Collection<DasEntryPoint> getEntryPoints(Integer start, Integer stop) throws UnimplementedFeatureException, DataSourceException {
                throw new UnimplementedFeatureException("Not implemented");
        }
        
        public String getEntryPointVersion() throws UnimplementedFeatureException, DataSourceException {
                throw new UnimplementedFeatureException("Not implemented");
        }
        
        public int getTotalEntryPoints() throws UnimplementedFeatureException, DataSourceException {
                throw new UnimplementedFeatureException("Not implemented");
        }
      
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
                throw new UnimplementedFeatureException("Not implemented");
        }
        @Override
        public DasAnnotatedSegment getFeatures(
                        String featureId, int start, int stop, Integer maxbins, Range rows)
                        throws UnimplementedFeatureException, DataSourceException {
                throw new UnimplementedFeatureException("Not implemented");
        }

```

### MyDAS configuration ###
For this we can make a copy of the configuration created in [MyDASTutorial2ndPart](MyDASTutorial2ndPart.md) and change some values for the new Data Source.
  1. Copy and paste the data source configuration of the data source with uri equal to `"examplepipes"`
```

        <datasource uri="examplepipes"
                    title="separatedByPipes"
                    description="Data source testing a new parser"
                    doc_href="http://mydas.googlecode.com"
                    mapmaster="http://mymapmaster.com">
			<maintainer email="gsalazar@ebi.ac.uk" />

			<version uri="examplepipes" created="2010-03-19">
	            <class>uk.ac.ebi.mydas.examples.SeparatedByPipesDataSource</class>
				<coordinates 	uri="CS_DS94"
								source="Gene_ID"
								authority="Ensembl"
								taxid="9606"
								test_range="ENSG00000160916">Ensembl,Gene_ID,Homo sapiens</coordinates>
				<capability	type="das1:sources"
						query_uri="http://localhost:8080/das/examplepipes" />
				<capability	type="das1:types"
						query_uri="http://localhost:8080/das/examplepipes/types" />
				<capability	type="das1:features"
						query_uri="http://localhost:8080/das/examplepipes/features" />
				<capability	type="das1:feature-by-id"
						query_uri="http://localhost:8080/das/examplepipes/features" />
				<capability	type="das1:stylesheet"
						query_uri="http://localhost:8080/das/examplegff/stylesheet" />
				<capability	type="das1:unknown-feature"	/>
			<property key="pipes_file" value="testGenes.txt" />
			</version>

            <dna-command-enabled>true</dna-command-enabled>
            <features-strictly-enclosed>true</features-strictly-enclosed>
            <use-feature-id-for-feature-label>true</use-feature-id-for-feature-label>
            <include-types-with-zero-count>true</include-types-with-zero-count>
        </datasource>
```

  * Edit the following fields:
    * Tag `<datasource>`, attribute `uri`: ensemblTest
    * Tag `<datasource>`, attribute `title`: Ensembl Test
    * Tag `<datasource>`, attribute `description`: Data source test using the ensmbl database
    * Tag `<datasource>`, attribute `doc_href`: http://www.ensembl.org/info/data/mysql.html
    * Tag `<maintainer>`, attribute `email`: `[Your email]`
    * Tag `<version>`, attribute `uri`: ensemblTest
    * Tag `<version>`, attribute `created`: 2010-04-07
    * Tag `<coordinates>`, attribute `url`: http://www.dasregistry.org/dasregistry/coordsys/CS_DS40
    * Tag `<coordinates>`, attribute `source`: Chromosome
    * Tag `<coordinates>`, attribute `version`: 37
    * Tag `<coordinates>`, attribute `test_range`: 5:11000000,12000000
    * Tag `<coordinates>`, value: Ensembl\_37a,Chromosome,Homo sapiens
    * Tag `<class>`, content: uk.ac.ebi.mydas.examples.EnsemblTestDataSource
    * All query attributes of the capabilities elements should have an url starting for `http://localhost:8080/das/ensemblTest`
  * Delete the tag `<property>`

Now is posible to check some of the responses of the server, try:
  * http://localhost:8080/das/ensemblTest/features?segment=5
  * http://localhost:8080/das/ensemblTest/features?segment=x
  * http://localhost:8080/das/ensemblTest/features?segment=5:113386,190088


[<<1st Tutorial(Beginners level)](MyDASTutorial.md) | [<2nd Tutotial(Intermediate level)](MyDASTutorial2ndPart.md) | **3rd Tutorial(Advance level)**