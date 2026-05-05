---
title: "Using Apache Sedona with AWS Glue to process billions of daily points from a geospatial dataset"
url: "https://aws.amazon.com/blogs/big-data/using-apache-sedona-with-aws-glue-to-process-billions-of-daily-points-from-a-geospatial-dataset/"
date: "Wed, 22 Apr 2026 15:42:28 +0000"
author: "Ruan Roloff"
feed_url: "https://aws.amazon.com/blogs/big-data/feed/"
---
<p>Data strategy can use geospatial data to provide organizations with insights for decision-making and operational optimization. By incorporating geospatial data (such as GPS coordinates, points, polygons and geographic boundaries), businesses can uncover patterns, trends, and relationships that might otherwise remain hidden across multiple industries, from aviation and transportation to environmental studies and urban planning. Processing and analyzing this geospatial data at scale can be challenging, especially when dealing with billions of daily observations.</p> 
<p>In this post, we explore how to use <a href="https://sedona.apache.org/latest/" rel="noopener noreferrer" target="_blank">Apache Sedona</a> with <a href="https://aws.amazon.com/glue/" rel="noopener noreferrer" target="_blank">AWS Glue</a> to process and analyze massive geospatial datasets.</p> 
<h2>Introduction to geospatial data</h2> 
<p>Geospatial data is information that has a geographic component. It describes objects, events, or phenomena along with their location on the Earth’s surface. This data includes coordinates (latitude and longitude), shapes (points, lines, polygons), and associated attributes (such as the name of a city or the type of road).</p> 
<p>Key types of geospatial geometries (and examples of each in parentheses) include:</p> 
<ul> 
 <li><strong>Point –</strong> Represents a single coordinate (a weather station).</li> 
 <li><strong>MultiPoint –</strong> A collection of points (bus stops in a city).</li> 
 <li><strong>LineString –</strong> A series of points connected in a line (a river or a flight path).</li> 
 <li><strong>MultiLineString –</strong> Multiple lines (multiple flight routes).</li> 
 <li><strong>Polygon –</strong> A closed area (the boundary of a city).</li> 
 <li><strong>MultiPolygon –</strong> Multiple polygons (national parks in a country).</li> 
</ul> 
<p>Geospatial datasets come in different formats, each designed to store and represent different types of geographic information. Common formats for geospatial data are vector formats (Shapefile, GeoJSON), raster formats (GeoTIFF, ESRI Grid), GPS formats (GPX, NMEA), web formats (WMS, GeoRSS) among others.</p> 
<h2>Core concepts of Apache Sedona</h2> 
<p><a href="https://sedona.apache.org/" rel="noopener noreferrer" target="_blank">Apache Sedona</a> is an open-source computing framework for processing large-scale geospatial data. Built on top of <a href="https://spark.apache.org/" rel="noopener noreferrer" target="_blank">Apache Spark</a>, Sedona extends Spark’s capabilities to handle spatial operations efficiently. At its core, Sedona introduces several key concepts that enable distributed spatial processing. These include Spatial Resilient Distributed Datasets (SRDDs), which allow for the distribution of spatial data across a cluster, and Spatial SQL, which provides a familiar SQL-like interface for spatial queries. Some of the core capabilities of Apache Sedona are:</p> 
<ul> 
 <li>Efficient spatial data types like points, lines and polygons.</li> 
 <li>Spatial operations and functions such as <code>ST_Contains</code> (check if point is inside of a polygon), <code>ST_Intersects</code> (check if point is inside of a polygon), <code>ST_H3CellIDs</code> (geospatial indexing system developed by Uber, return the <a href="https://h3geo.org/" rel="noopener noreferrer" target="_blank">H3</a> cell ID(s) that contain the given point at the specified resolution).</li> 
 <li>Spatial joins to combine different spatial datasets.</li> 
 <li>Integration with Spark SQL (geospatial functions to run spatial SQL queries).</li> 
 <li>Spatial indexing techniques, such as quad-trees and R-trees, to optimize query performance.</li> 
</ul> 
<p>For more information about the functions available in Apache Sedona, visit the official Sedona <a href="https://sedona.apache.org/1.8.0/api/sql/Function/" rel="noopener noreferrer" target="_blank">Functions</a> documentation.</p> 
<h2>Use case</h2> 
<p>This use case consists of a global air traffic visualization and analysis platform that processes and displays real-time or historical aircraft tracking data on an interactive world map. Using unique aircraft identifiers from the International Civic Aviation Organization (ICAO), the system ingests trajectory records containing information such as geographic position (latitude and longitude), altitude, speed, and flight direction, then transforms this raw data into two complementary visual layers. The Flight Tracks Layer plots the routes traveled by each aircraft individually, allowing for the analysis of specific trajectories and navigation patterns. The Flight Density Layer uses hexagonal spatial indexing (H3) to aggregate and identify regions of higher air traffic concentration worldwide, revealing busy air corridors, aviation hubs, and high-density flight zones.</p> 
<p>The dataset used for this use case is <a href="https://www.adsb.lol/docs/open-data/historical/" rel="noopener noreferrer" target="_blank">historical flight tracker data</a> from <a href="https://www.adsb.lol/" rel="noopener noreferrer" target="_blank">ADSB.lol</a>. ADSB.lol provides unfiltered flight tracker with a focus on open data. Data is also freely available via the API. The data contains a file per aircraft, a JSON gzip file containing the data for that aircraft for the day.</p> 
<p>This is a JSON trace file format sample:</p> 
<div class="hide-language"> 
 <pre><code class="lang-typescript">{
    icao: "0123ac", // hex id of the aircraft
    timestamp: 1609275898.495, // unix timestamp in seconds since epoch (1970)
    trace: [
        [ seconds after timestamp,
            lat,
            lon,
            altitude in ft or "ground" or null,
            ground speed in knots or null,
            track in degrees or null, (if altitude == "ground", this will be true heading instead of track)
            flags as a bitfield: (use bitwise and to extract data)
                (flags &amp; 1 &gt; 0): position is stale (no position received for 20 seconds before this one)
                (flags &amp; 2 &gt; 0): start of a new leg (tries to detect a separation point between landing and takeoff that separates flights)
                (flags &amp; 4 &gt; 0): vertical rate is geometric and not barometric
                (flags &amp; 8 &gt; 0): altitude is geometric and not barometric
             ,
            vertical rate in fpm or null,
            aircraft object with extra details or null,
            type / source of this position or null,
            geometric altitude or null,
            geometric vertical rate or null,
            indicated airspeed or null,
            roll angle or null
        ],
    ]
}</code></pre> 
</div> 
<p>For this use case, this is a simplified schema of the dataset after processing:</p> 
<ul> 
 <li><code>icao -</code> Unique aircraft identifier</li> 
 <li><code>timestamp -</code> Epoch timestamp of the observation (converted to readable format)</li> 
 <li><code>trace.lat / trace.lon -</code> Latitude and longitude of the aircraft</li> 
 <li><code>trace.altitude -</code> Aircraft altitude</li> 
 <li><code>trace.ground_speed -</code> Ground speed</li> 
 <li><code>geometry -</code> Geospatial geometry of the observation point (<code>Point</code>)</li> 
</ul> 
<h2>Solution overview</h2> 
<p>This solution enables aircraft tracking and analysis. The data can be visualized on maps and used for aviation management and safety applications. The process begins with data acquisition, extracting the compressed JSON files from TAR archives, then transforms this raw data into geospatial objects, aggregating them into H3 cells for efficient analysis. The processed data schema includes ICAO aircraft identifiers, timestamps, latitude/longitude coordinates, and derived fields such as H3 cell identifiers and point counts per cell. This structure allows detailed tracking of individual flights and aggregate analysis of traffic patterns. For visualization, you can generate density maps using the H3 grid system and create visual representations of individual flight tracks. The architecture data flow is as follows:</p> 
<ul> 
 <li><strong>Data ingestion –</strong> Aircraft observation data stored as JSON compressed files in <a href="https://aws.amazon.com/s3/" rel="noopener noreferrer" target="_blank">Amazon Simple Storage Service</a> (Amazon S3).</li> 
 <li><strong>Data processing –</strong> AWS Glue jobs using Apache Sedona for geospatial processing.</li> 
 <li><strong>Data visualization –</strong> Spark SQL with Sedona’s spatial functions to extract insights and export data to visualize the information in a map on Kepler.gl.</li> 
</ul> 
<p>The following figure illustrates this solution.</p> 
<p><img alt="AWS architecture diagram showing a geospatial data processing pipeline." class="alignnone size-full wp-image-90098" height="728" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/04/13/BDB-5249-geospatial-1_v2.png" width="761" /></p> 
<h3>Prerequisites</h3> 
<p>You will need the following for this solution:</p> 
<ul> 
 <li>An <a href="https://aws.amazon.com/resources/create-account/" rel="noopener noreferrer" target="_blank">AWS Account</a> and a user with AWS Console access.</li> 
 <li>Access to a Linux terminal and the <a href="https://docs.aws.amazon.com/cli/latest/userguide/getting-started-quickstart.html" rel="noopener noreferrer" target="_blank">AWS Command Line Interface</a> (AWS CLI).</li> 
 <li>An <a href="https://docs.aws.amazon.com/glue/latest/dg/create-an-iam-role.html" rel="noopener noreferrer" target="_blank">IAM role for AWS Glue</a> with list, read, and write permissions for Amazon S3 buckets.</li> 
 <li>An Amazon S3 Bucket for flight files. For this example, name the bucket <code>blog-sedona-nessie-&lt;account_number&gt;-&lt;aws_region&gt;</code>, using your account number and region.</li> 
 <li>An Amazon S3 bucket for artifacts and Sedona libraries. For this example, name the bucket <code>blog-sedona-artifacts-&lt;account_number&gt;-&lt;aws_region&gt;</code>, using your account number and region.</li> 
 <li>Download a day of historical data from <a href="https://www.adsb.lol/docs/open-data/historical/" rel="noopener noreferrer" target="_blank">ADSB.lol</a>. In our examples, we used <a href="https://github.com/adsblol/globe_history_2025/releases/download/v2025.05.29-planes-readsb-prod-0tmp/v2025.05.29-planes-readsb-prod-0tmp.tar.aa" rel="noopener noreferrer" target="_blank">v2025.05.29-planes-readsb-prod-0tmp.tar.aa</a> and <a href="https://github.com/adsblol/globe_history_2025/releases/download/v2025.05.29-planes-readsb-prod-0tmp/v2025.05.29-planes-readsb-prod-0tmp.tar.ab" rel="noopener noreferrer" target="_blank">v2025.05.29-planes-readsb-prod-0tmp.tar.ab</a>.</li> 
 <li>Download the Apache Sedona libraries. The example was created using <a href="https://repo1.maven.org/maven2/org/apache/sedona/sedona-spark-shaded-3.5_2.12/1.7.1/sedona-spark-shaded-3.5_2.12-1.7.1.jar" rel="noopener noreferrer" target="_blank">sedona-spark-shaded-3.5_2.12-1.7.1.jar</a> and <a href="https://repo1.maven.org/maven2/org/datasyslab/geotools-wrapper/1.7.1-28.5/geotools-wrapper-1.7.1-28.5.jar" rel="noopener noreferrer" target="_blank">geotools-wrapper-1.7.1-28.5.jar</a>.</li> 
 <li>Download the <a href="https://github.com/aws-samples/sample-blog-geospacial-lake-on-aws-with-aws-dataservices/blob/main/src/glue_scripts/process_sedona_geo_track.py" rel="noopener noreferrer" target="_blank">AWS Glue script</a> from AWS Sample to process the geospatial data.</li> 
 <li>Review the <a href="https://docs.aws.amazon.com/glue/latest/dg/security.html" rel="noopener noreferrer" target="_blank">AWS Glue security best practices</a>, especially IAM least-privilege, encryption for sensitive data at rest and in transit, and configuring VPC Endpoints to prevent data from routing through the public internet.</li> 
</ul> 
<h2>Solution walkthrough</h2> 
<p>From now on, executing the next steps will incur costs on AWS. This step-by-step walkthrough demonstrates an approach to processing and analyzing large-scale geospatial flight data using Apache Sedona and Uber’s H3 spatial indexing system, using AWS Glue for distributed processing and Apache Sedona for efficient geospatial computations. It explains how to ingest raw flight data, transform it using Sedona’s geospatial functions, and index it with H3 for optimized spatial queries. Finally, it also demonstrates how to visualize the data using Kepler.gl. For data processing, it is possible to use both Glue scripts and <a href="https://sedona.apache.org/latest/setup/glue/" rel="noopener noreferrer" target="_blank">Glue notebooks</a>. In this post, we will focus only on Glue scripts.</p> 
<h3>Upload the Apache Sedona libraries to Amazon S3</h3> 
<ol> 
 <li>Open your OS terminal command line.</li> 
 <li>Create a folder to download the Sedona libraries and name it <strong>jar</strong>. <pre><code class="lang-bash">
	# Create a directory for the Sedona libraries (JARs files)
	mkdir jar
	# Go to the folder JARs folder
	cd jar
	</code></pre> </li> 
 <li>Download the Apache Sedona libraries. <pre><code class="lang-bash">
	# Download required Sedona libraries (JARs files)
	wget https://repo1.maven.org/maven2/org/apache/sedona/sedona-spark-shaded-3.5_2.12/1.7.1/sedona-spark-shaded-3.5_2.12-1.7.1.jar
	wget https://repo1.maven.org/maven2/org/datasyslab/geotools-wrapper/1.7.1-28.5/geotools-wrapper-1.7.1-28.5.jar
	</code></pre> </li> 
 <li>Upload the Sedona libraries (JARs files) to Amazon S3. In this example, we use the S3 path <code>s3://aws-blog-post-sedona-artifacts/jar/</code>. <pre><code class="lang-bash">
	# Upload the JARs files to Amazon S3 bucket
	aws s3 cp . s3://blog-sedona-artifacts-&lt;account_number&gt;-&lt;aws_region&gt;/jar/ --recursive
	</code></pre> </li> 
 <li>Your Amazon S3 folder should now look similar to the following image:</li> 
</ol> 
<p><img alt="Amazon S3 console screenshot displaying the jar folder contents in blog-sedona-artifacts bucket." class="alignnone size-full wp-image-90099" height="919" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/04/10/BDB-5249-geospatial-2.jpg" width="2560" /></p> 
<h3>Download and upload the geospatial data to Amazon S3</h3> 
<ol> 
 <li>Open your OS terminal command line.</li> 
 <li>Create a folder to download the flight files and name it <strong>adsb_dataset</strong>. <pre><code class="lang-bash">		# Create a directory for download the geospatial flight files
		mkdir adsb_dataset
		# Go to the folder for geospatial flight files
		cd adsb_dataset
	</code></pre> </li> 
 <li>Download the flight files data from <a href="https://github.com/adsblol/globe_history_2025/releases" rel="noopener noreferrer" target="_blank">adsblol GitHub repository</a>. <pre><code class="lang-bash">	# Download the geospatial flight files in the folder created
	wget https://github.com/adsblol/globe_history_2025/releases/download/v2025.05.29-planes-readsb-prod-0tmp/v2025.05.29-planes-readsb-prod-0tmp.tar.aa
	wget https://github.com/adsblol/globe_history_2025/releases/download/v2025.05.29-planes-readsb-prod-0tmp/v2025.05.29-planes-readsb-prod-0tmp.tar.ab
	</code></pre> </li> 
 <li>Extract the flight files. <pre><code class="lang-bash">	# Combine the two the tar files together
	cat v2025.05.29* &gt;&gt; combined.tar
	# Extract the json flight files from the tar file
	tar xf combined.tar
	</code></pre> </li> 
 <li>Copy the flight files to Amazon S3. In this case, we are using the S3 folder: <code>s3://blog-sedona-nessie-&lt;account_number&gt;-&lt;aws_region&gt;/raw/adsb-2025-05-28/traces/</code>. <pre><code class="lang-bash">	# Copy the json flight files to Amazon S3
	aws s3 cp ./traces/ s3://blog-sedona-nessie-&lt;account_number&gt;-&lt;aws_region&gt;/raw/adsb-2025-05-28/traces/ --recursive
	</code></pre> </li> 
 <li>Your Amazon S3 folder should now look similar to the following image.</li> 
</ol> 
<p><img alt="Amazon S3 console showing JSON trace files in the path raw/adsb-2025-05-28/traces/00/." class="alignnone size-full wp-image-90100" height="1096" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/04/10/BDB-5249-geospatial-3-scaled.jpg" width="2560" /></p> 
<h3>Create an AWS Glue job and set up the job</h3> 
<p>Now, we are ready to define the AWS Glue job using Apache Sedona to read the geospatial data files. To create a Glue job:</p> 
<ol> 
 <li>Open the <a href="https://console.aws.amazon.com/glue/" rel="noopener noreferrer" target="_blank">AWS Glue console</a>.</li> 
 <li>On the <strong>Notebooks</strong> page, choose <strong>Script editor</strong>.</li> 
</ol> 
<p><img alt="AWS Glue Studio jobs creation interface showing three job creation methods: Visual ETL with data flow interface, Notebook for interactive coding, and Script editor for code authoring" class="alignnone size-full wp-image-90101" height="800" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/04/10/BDB-5249-geospatial-4-scaled.jpg" width="2560" /></p> 
<ol start="3"> 
 <li>On the Script screen, for the engine, choose <strong>Spark</strong>, then select the option <strong>Upload script</strong>.</li> 
 <li>Choose <strong>Choose file</strong>. Find the <code>process_sedona_geo_track.py</code> file, then choose <strong>Create script</strong>.</li> 
</ol> 
<p><img alt="Script creation dialog box with Spark engine selected. Upload script option is active, showing successfully uploaded file process_sedona_geo_track.py." class="alignnone size-full wp-image-90102" height="730" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/04/10/BDB-5249-geospatial-5.jpg" width="1602" /></p> 
<ol start="5"> 
 <li>Rename the job from <strong>Untitled</strong> to <strong>process_sedona_geo_track</strong>.</li> 
 <li>Choose <strong>Save</strong>.</li> 
 <li>Now, let’s set up the AWS Glue job. Choose <strong>Job Details.</strong></li> 
 <li>Choose the <strong>IAM Role</strong> created to be used with Glue. For this example, we use <strong>blog-glue</strong>.</li> 
 <li>Set the <strong>Glue version</strong> to <strong>Glue 5.0</strong> and the Worker type as needed. For this example, <strong>G.1X</strong> is sufficient, but we use <strong>G.2X</strong> to speed up processing.</li> 
</ol> 
<p><img alt="AWS Glue job details configuration page for process_sedona_geo_track." class="alignnone size-full wp-image-90103" height="1050" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/04/10/BDB-5249-geospatial-6.jpg" width="2182" /></p> 
<ol start="10"> 
 <li>Now, let’s import the libraries for Apache Sedona.</li> 
 <li>In the <strong>Dependent JARs path</strong>, type the path of the JAR files for Apache Sedona that you uploaded in the preceding steps. For this example, we used <code>s3://blog-sedona-artifacts-&lt;account_number&gt;-&lt;aws_region&gt;/jar/sedona-spark-shaded-3.5_2.12-1.7.1.jar,s3://blog-sedona-artifacts-&lt;account_number&gt;-&lt;aws_region&gt;/jar/geotools-wrapper-1.7.1-28.5.jar</code></li> 
 <li>In <strong>Additional Python modules path</strong>, enter the modules for Apache Sedona: <strong>apache-sedona==1.7.1,geopandas==0.13.2,shapely==2.0.1,pyproj==3.6.0,fiona==1.9.5,rtree==1.2.0</strong></li> 
</ol> 
<p><img alt="ob libraries configuration section showing Dependent JARs path pointing to S3 bucket." class="alignnone size-full wp-image-90104" height="956" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/04/10/BDB-5249-geospatial-7.jpg" width="2016" /></p> 
<ol start="13"> 
 <li>In the <strong>Job parameters</strong> section, in the <strong>Key</strong> field, type <strong> —BUCKET_NAME</strong>. For its <strong>Value</strong>, enter your bucket name. In this example, ours is <code>blog-sedona-nessie-&lt;account_number&gt;-&lt;aws_region&gt;</code>.</li> 
</ol> 
<p><img alt="ob parameters configuration interface showing key-value pair with --BUCKET_NAME parameter." class="alignnone size-full wp-image-90105" height="229" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/04/10/BDB-5249-geospatial-8.jpg" width="704" /></p> 
<ol start="14"> 
 <li>Choose <strong>Save</strong>.</li> 
</ol> 
<h3>Processing the geospatial flights data</h3> 
<p>Before we run the job, let’s understand how the code works. First, import the Apache Sedona libraries:</p> 
<div class="hide-language"> 
 <pre><code class="lang-python">import json 
import gzip 
from sedona.spark import SedonaContext</code></pre> 
</div> 
<p>Next, initialize the Sedona context using an existing Spark session:</p> 
<div class="hide-language"> 
 <pre><code class="lang-python">sedona = SedonaContext.create(spark)</code></pre> 
</div> 
<p>After that, create a function for handling compressed JSON data:</p> 
<div class="hide-language"> 
 <pre><code class="lang-python">def parse_gzip_json(byte_content):
        try:
            decompressed = gzip.decompress(byte_content)
            return json.loads(decompressed.decode('utf-8'))
        except Exception as e:
            print(f"Error during gzip parse: {str(e)}")
            return None</code></pre> 
</div> 
<p>Add a function to transform raw tracking data into a structured format suitable for a valid coordinates process:</p> 
<div class="hide-language"> 
 <pre><code class="lang-python">def flatten_records(json_obj):
    records = []
    if "trace" in json_obj and isinstance(json_obj["trace"], list):
        for point in json_obj["trace"]:
            if len(point) &gt;= 3:
                lat, lon = float(point[1]), float(point[2])
                if -90 &lt;= lat &lt;= 90 and -180 &lt;= lon &lt;= 180:
                    records.append(Row(
                        icao=json_obj.get("icao", None),
                        timestamp=json_obj.get("timestamp", None),
                        lat=lat,
                        lon=lon
                    ))
    return records</code></pre> 
</div> 
<p>The <code>flat_rdd</code> variable applies these functions to the structured data from the original gzipped JSON. Each element in this RDD is a Row object representing a single data point from an aircraft’s trace, with fields for ICAO, timestamp, latitude, and longitude.</p> 
<div class="hide-language"> 
 <pre><code class="lang-python">flat_rdd = raw_rdd.map(lambda x: parse_gzip_json(x[1])).filter(lambda x: x is not None).flatMap(flatten_records)</code></pre> 
</div> 
<p>The ADSB trace files contain a deeply nested JSON structure where the trace field holds an array of mixed-type arrays, compressed in Gzip format. For this specific case, developing a UDF represented one of the most practical and efficient solutions. Since Gzip is a non-splittable format, Spark is unable to parallelize processing, constraining both methods to a single worker per file and processing the data multiple times across JVM decompression, full JSON parsing, and subsequent re-parsing operations. The UDF bypasses all of this by reading raw bytes and doing everything in a single Python pass: decompress → parse → extract → validate, returning only the small set of needed fields directly to Spark.</p> 
<p>The Spark SQL query processes geographic trace data using the H3 hexagonal grid system, converting point data into a regularized hexagonal grid that can help identify areas of high point density. A <a href="https://h3geo.org/docs/core-library/restable/#average-area-in-km2" rel="noopener noreferrer" target="_blank">resolution</a> of 5 was adopted, producing hexagons of approximately 253 km² (roughly the same size as the city of Edinburgh, Scotland, which is approximately 264 km²), for its ability to effectively capture route density patterns at the city and metropolitan level.</p> 
<div class="hide-language"> 
 <pre><code class="lang-sql">h3_traces_df = spark.sql("""
WITH base_h3 AS (
    SELECT
        ST_H3CellIDs(geometry, 5, false)[0] AS h3_index,
        lat,
        lon
    FROM traces
)
SELECT
    COUNT(*) AS num, -- Count points in each H3 cell
    h3_index,
    AVG(lon) AS center_lon,
    AVG(lat) AS center_lat
FROM base_h3
GROUP BY h3_index
""")
</code></pre> 
</div> 
<p>Finally, this code prepares the datasets for visualization purposes. The first dataset is based on the aircraft unique identifier. The complete dataset for a single day can contain more than 80 million data points. A random sampling rate of 0.1% was applied, which proves sufficient to illustrate route density patterns without overwhelming the Kepler.gl browser renderer. The second dataset aggregates trace points into hexagonal spatial cells (result from the query above).</p> 
<div class="hide-language"> 
 <pre><code class="lang-python">points_viz_sampled = df_points.select(
    col("icao"), # Aircraft unique identifier (24-bit address)
    col("timestamp").cast("double").alias("timestamp"),
    col("lat").cast("double").alias("lat"),
    col("lon").cast("double").alias("lon")
).sample(False, 0.001)

h3_viz_csv = h3_traces_df.select(
    col("num").alias("point_count"),
    col("h3_index").cast("string").alias("h3_index"),
    col("center_lon"),
    col("center_lat")
)</code></pre> 
</div> 
<p>Now that we understand the code, let’s run it.</p> 
<ol> 
 <li>Open the <a href="https://console.aws.amazon.com/glue/" rel="noopener noreferrer" target="_blank">AWS Glue console</a>.</li> 
 <li>On the <strong>ETL jobs &gt;&gt; Notebooks </strong>page, choose the job name <strong>process_sedona_geo_track</strong>.</li> 
 <li>Choose <strong>Run</strong>.</li> 
</ol> 
<p><img alt="Python script editor showing import statements for process_sedona_geo_track job." class="alignnone size-full wp-image-90106" height="417" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/04/10/BDB-5249-geospatial-9.jpg" width="1038" /></p> 
<ol start="4"> 
 <li>Now, it is possible to monitor the job by choosing the <strong>Runs</strong> tab.</li> 
 <li>It may take a few minutes to run the entire job. It took nearly 8 minutes to process approximately 2.50 GB (67,540 compressed files) with 20 DPUs. After the job is processed, you should see your job with the status <strong>Succeeded</strong>.</li> 
</ol> 
<p><img alt="Job runs monitoring dashboard showing successful execution on June 5, 2025, running from 12:28:03 to 12:36:37 with 8 minutes 19 seconds duration." class="alignnone size-full wp-image-90107" height="785" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/04/10/BDB-5249-geospatial-10.jpg" width="1253" /></p> 
<p>Now your data should be saved for a preview visualization demo in a folder named <code>s3://blog-sedona-nessie-&lt;account_number&gt;-&lt;aws_region&gt;/visualization/</code>.</p> 
<h3>Performance insights</h3> 
<p>The workload characterization of this job reveals a CPU-intensive profile, primarily because of the processing of small binary files with GZIP compression and subsequent JSON parsing. Given the inherent nature of this pipeline, which includes Python UDF serialization and partial single-partition write stages, linear scaling does not yield proportional performance gains. The following table presents an analysis of AWS Glue configurations, evaluating the trade-off between computational capacity, execution duration, and associated costs:</p> 
<table border="1px" cellpadding="10px" class="styled-table"> 
 <tbody> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><strong>Duration</strong></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><strong>Capacity (DPUs)</strong></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><strong>Worker type</strong></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><strong>Glue version</strong></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><strong>Estimated Cost*</strong></td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">10 m 7 s</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">32 DPUs</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">G.1X</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">5</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><strong>$2.34</strong></td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">11 m 50 s</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">10 DPUs</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">G.1X</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">5</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><strong>$0.88</strong></td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">19 m 7 s</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">4 DPUs</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">G.1X</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">5</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><strong>$0.59</strong></td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">8 m 19 s</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">20 DPUs</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">G.2X</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">5</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><strong>$1.32</strong></td> 
  </tr> 
 </tbody> 
</table> 
<p>*Estimated Cost = DPUs x Duration (hours) x $0.44 per DPU-hour (<code>us-east-1</code>)</p> 
<h2>Visualizing and analyzing geospatial data with Kepler.gl</h2> 
<p><a href="https://kepler.gl/" rel="noopener noreferrer" target="_blank">Kepler.gl</a> is an open-source geospatial analysis tool developed by <a href="https://www.uber.com/en-HK/blog/keplergl/" rel="noopener noreferrer" target="_blank">Uber</a> with code available at <a href="https://github.com/keplergl/kepler.gl" rel="noopener noreferrer" target="_blank">Github</a>. Kepler.gl is designed for large-scale data exploration and visualization, offering multiple map layers, including point, arc, heatmap, and 3D hexagon. It supports various file formats like CSV, GeoJSON, and KML. In this use case, we will use Kepler.gl to present interactive visualizations that illustrate flight patterns, routes, and densities across global airspace.</p> 
<h3>Downloading the geospatial files</h3> 
<p>Before we can view the graph, we will need to download the flight files to our local machine, unzip them, and rename them (to make it easier to identify the files).</p> 
<ol> 
 <li>Open your OS terminal command line.</li> 
 <li>Create the folders to download the data processed in the steps before. In this case, we create <strong>kepler</strong> and <strong>kepler_csv</strong>. <pre><code class="lang-bash">	#create kepler folders: first folder is to download the files,
	#second folder is to organize the files to use in the next step
	mkdir kepler
	mkdir kepler_csv
	</code></pre> </li> 
 <li>Replace the bracketed variables with your account and directory information, then download all the CSV files. <pre><code class="lang-bash">	#copy the files from Amazon S3 to local machine
	aws s3 cp s3://blog-sedona-nessie-&lt;account_number&gt;-&lt;aws_region&gt;/visualization/ /&lt;user_directory&gt;/kepler --recursive
	</code></pre> </li> 
 <li>Extract the files, rename them, and move them to another folder. <pre><code class="lang-bash">	# Extract the files processed by Spark and Sedona
	gzip -d ./kepler/kepler_h3_density/*.gz
	gzip -d ./kepler/kepler_track_points_sample/*.gz
	
	# Rename the Spark output files to more readable names
	cd ./kepler/kepler_h3_density/
	ls
	mv part-00000-*.csv kepler_h3_density.csv
	cd ..
	
	cd ./kepler/kepler_track_points_sample/
	ls
	mv part-00000-*.csv kepler_track_points_sample.csv
	cd ..
	
	# Ensure the output folder exists
	mkdir -p ../kepler_csv
	
	# Copy the renamed CSV files to the folder that will be used as input in kepler.gl
	cp ./kepler/kepler_h3_density/*.csv ../kepler_csv
	cp ./kepler/kepler_track_points_sample/*.csv ../kepler_csv
	</code></pre> </li> 
 <li>Your <strong>kepler_csv</strong> folder should look similar to the return of the command below. <pre><code class="lang-bash">	#list the files in the kepler_csv directory
	ls -l
	total 11684
	-rw-rw-r-- 1 ec2-user ec2-user 8630110 Jun 12 14:47 kepler_h3_density.csv
	-rw-rw-r-- 1 ec2-user ec2-user 3331763 Jun 12 14:47 kepler_track_points_sample.csv
	</code></pre> </li> 
</ol> 
<h3>Visualizing the data in a graph</h3> 
<p>Now that you have saved the data to your local machine, you can analyze the flight data through interactive map graphics. To import the data into the Kepler.gl web visualization tool:</p> 
<ol> 
 <li>Open the <a href="https://kepler.gl/demo" rel="noopener noreferrer" target="_blank">Kepler.gl Demo</a> web application.</li> 
 <li>Load data into Kepler.gl: 
  <ol type="a"> 
   <li>Choose <strong>Add Data</strong> in the left panel.</li> 
   <li>Drag and drop both CSV files (<code>flight_points</code> and <code>h3_density</code>) into the upload area.</li> 
   <li>Confirm that both datasets are loaded successfully.</li> 
  </ol> </li> 
 <li>Delete all layers.</li> 
 <li>Create the <strong>Flight Density Layer:</strong> 
  <ol type="a"> 
   <li>Choose <strong>Add Layer</strong> in the left panel.</li> 
   <li>In <strong>Basic</strong>, choose <strong>H3</strong> as the layer type, then add the following configuration: 
    <ol type="i"> 
     <li>Layer Name: <strong>Flight Density</strong></li> 
     <li>Data Source: <strong>kepler_h3_density.csv</strong></li> 
     <li>Hex ID: <strong>h3_index</strong></li> 
    </ol> </li> 
   <li>In the <strong>Fill Color</strong> section: 
    <ol type="i"> 
     <li>Color: <strong>point_count</strong></li> 
     <li>Color Scale: <strong>Quantile</strong>.</li> 
     <li>Color Range: Choose a blue/green gradient.</li> 
    </ol> </li> 
   <li>Set <strong>Opacity</strong> to <strong>0.7</strong>.</li> 
   <li>In the <strong>Coverage</strong> section, set it to <strong>0.9</strong>.</li> 
  </ol> </li> 
 <li>Create the <strong>Flight Tracks Layer:</strong> 
  <ol type="a"> 
   <li>Choose <strong>Add Layer</strong> in the left panel.</li> 
   <li>In <strong>Basic</strong>, choose <strong>Point</strong> as the layer type, then add the following configuration: 
    <ol type="i"> 
     <li>Layer Name: <strong>Flight Tracks</strong></li> 
     <li>Data Source: <strong>kepler_track_points_sample.csv</strong></li> 
     <li>Columns: 
      <ol> 
       <li>Latitude: <strong>lat</strong></li> 
       <li>Longitude: <strong>lon</strong></li> 
      </ol> </li> 
    </ol> </li> 
   <li>In the <strong>Fill Color </strong>section: 
    <ol type="i"> 
     <li>Solid Color: <strong>Orange</strong></li> 
     <li>Opacity: <strong>0.3</strong></li> 
    </ol> </li> 
   <li>Set the Point’s <strong>Radius</strong> to 1</li> 
  </ol> </li> 
 <li>The layers should look similar to the following figure.</li> 
</ol> 
<p><img alt="Kepler.gl layer configuration panel for Flight Density H3 layer using kepler_h3_density.csv data source." class="alignnone size-full wp-image-90108" height="1051" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/04/10/BDB-5249-geospatial-11.jpg" width="998" /></p> 
<ol start="7"> 
 <li>The graph visualization should now show flight density through color-coded hexagons, with individual flight tracks visible as orange points:</li> 
</ol> 
<p><img alt="Kepler.gl interactive map visualization displaying global flight density heatmap. High-density areas shown in yellow over North America, particularly the United States." class="alignnone size-full wp-image-90109" height="924" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/04/10/BDB-5249-geospatial-12.jpg" width="1897" /></p> 
<p>There you go! Now that you have knowledge about geospatial data and have created your first use case, take the opportunity to do some analysis and learn some interesting facts about flight patterns.</p> 
<p>It is possible to experiment with other interesting types of analysis in Kepler.gl, such as <a href="https://docs.kepler.gl/docs/user-guides/h-playback" rel="noopener noreferrer" target="_blank">Time Playback</a>.</p> 
<h2>Clean up</h2> 
<p>To clean up your resources, complete the following tasks:</p> 
<ol> 
 <li>Delete the AWS Glue job <code>process_sedona_geo_track</code>.</li> 
 <li><a href="https://docs.aws.amazon.com/AmazonS3/latest/userguide/empty-bucket.html" rel="noopener noreferrer" target="_blank">Delete content</a> from the Amazon S3 buckets: <code>blog-sedona-artifacts-&lt;account_number&gt;-&lt;aws_region&gt;</code> and <code>blog-sedona-nessie-&lt;account_number&gt;-&lt;aws_region&gt;</code>.</li> 
</ol> 
<h2>Conclusion</h2> 
<p>In this post, we showed how processing geospatial data can present significant challenges due to its complex nature (from big data to data structure format). For this use case of flight trackers, it involves vast amounts of information across multiple dimensions such as time, location, altitude, and flight paths, however, the combination of Spark’s distributed computing capabilities and Sedona’s optimized geospatial functions helps overcome those challenges. The spatial partitioning and indexing features of Sedona, coupled with Spark’s framework, enable us to perform complex spatial joins and proximity analyses efficiently, simplifying the overall data processing workflow.</p> 
<p>The serverless nature of AWS Glue eliminates the need for managing infrastructure while automatically scaling resources based on workload demands, making it an ideal platform for processing growing volumes of flight data. As the volume of flight data grows or as processing requirements fluctuate, with AWS Glue, you can quickly adjust resources to meet demand, ensuring optimal performance without the need for cluster management.</p> 
<p>By converting the processed results into CSV format and visualizing them in Kepler.gl, it is possible to create interactive visualizations that reveal patterns in flight paths, and you can efficiently analyze air traffic patterns, routes, and other insights. This end-to-end solution demonstrates how a modern data strategy in AWS with the support of open-source tools can transform raw geospatial data into actionable insights.</p> 
<hr style="width: 80%;" /> 
<h2>About the authors</h2> 
<footer> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="Ruan" class="alignnone size-full wp-image-90123" height="133" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/04/13/ruanroloff.jpeg" width="100" />
  </div> 
  <p><strong>Ruan Roloff</strong> is a Lead GTM Specialist Architect for Analytics and AI at AWS. During his time at AWS, he was responsible for the data journey and AI product strategy of customers across a range of industries, including finance, oil and gas, manufacturing, digital natives, public sector, and startups. He has helped these organizations achieve multi-million dollar use cases. Outside of work, Ruan likes to assemble and disassemble things, fish on the beach with friends, play SFII, and go hiking in the woods with his family.</p> 
 </div> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="Lucas" class="alignnone size-full wp-image-90122" height="133" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/04/13/lucasvitoreti.jpeg" width="100" />
  </div> 
  <p><strong>Lucas Vitoreti</strong> is a ProServe Data &amp; Analytics Specialist at AWS with 12+ years in the data domain. Architects and delivers solutions for data warehouses, lakes, lakehouses, and meshes, helping organizations transform their data strategies and achieve business outcomes. Expertise in scalable data architectures and guiding data-driven transformations. He balances professional life with weightlifting, music, and family time.</p> 
 </div> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="Denys" class="alignnone size-full wp-image-90121" height="133" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/04/13/denysgonzaga.jpeg" width="100" />
  </div> 
  <p><strong>Denys Gonzaga</strong> is a ProServe Consultant at AWS, he is an experienced professional with over 15 years of working across multiple technical domains, with a strong focus on development and data analytics. Throughout his career, he has successfully applied his skills in various industries, including aerospace, finance, telecommunications, and retail. Outside of AWS, Denys enjoys spending time with his family and playing video games.</p> 
 </div> 
</footer>
