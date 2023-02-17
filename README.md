## Exploring 21,280 Global Cities: Applications of Big Open Data Processing System



## Query in Vector Representation



### Table Description

####  table: boundary

<ul>
  <li> <b>cityID</b>: ID of the city </li>
  <li> <b>polygon</b>: Polygon of the city stored as a list of (latitude, longitude)  </li>
  <li> <b>minlon, maxlon, minlat, maxlat</b>: Bounding box of the city polygon  </li>
</ul>


#### table: road

<ul>
  <li> <b>node_start_lat, node_start_lon</b>: latitude and longitude of the starting node </li>
  <li> <b>node_end_lat, node_end_lon</b>: latitude and longitude of the end node </li>
</ul>


####  table: light

<ul>
  <li> <b>lat</b>: latitude </li>
  <li> <b>lon</b>: longitude </li>
  <li> <b>light</b>: nighttime light intensity </li>
</ul>


####  table: builtup

<ul>
  <li> <b>lat</b>: latitude </li>
  <li> <b>lon</b>: longitude  </li>
  <li> <b>area</b>: built-up surface area  </li>
</ul>


#### table: population

<ul>
  <li> <b>lat</b>: latitude </li>
  <li> <b>lon</b>: longitude  </li>
  <li> <b>population</b>: population  </li>  
</ul>




### Query 21,280 Global Cities

#### Road Length

```sql
SELECT 
	cityID, SUM(distance) as distance_sum 
FROM
(
    SELECT
  		cityID, 
  		geo_distance(lon1, lat1, lon2, lat2) AS distance
    FROM 
  	( 
  		SELECT  /*+ mapjoin(boundary) */
      	boundary.cityID,
      	road.node_start_lon AS lon1, 
      	road.node_start_lat AS lat1, 
      	road.node_end_lon AS lon2, 
      	road.node_end_lat AS lat2,
        pnpoly(road.node_start_lon, road.node_start_lat, boundary.polygon) AS inPoly 
       
      FROM
      	boundary, road
    	WHERE
  			road.node_start_lon < boundary.maxlon	AND 
    		road.node_start_lon > boundary.minlon	AND 
    		road.node_start_lat < boundary.maxlat	AND 
    		road.node_start_lat > boundary.minlat	
     )
     WHERE inPoly = true
)
GROUP BY cityID
```



#### Nighttime Light

```sql
SELECT 
	cityID, SUM(light) as light_sum
FROM
(
    SELECT  /*+ mapjoin(boundary) */
  		boundary.cityID, 
  		light.light,
      pnpoly(light.lon, light.lat, boundary.polygon) AS inPoly 
    FROM
      boundary, light
    WHERE 
  		light.lon < boundary.maxlon	AND 
    	light.lon > boundary.minlon	AND 
    	light.lat < boundary.maxlat	AND 
    	light.lat > boundary.minlat	
)
WHERE inPoly = true
GROUP BY cityID
```



#### Built-up Surface Area

```sql
SELECT 
	cityID, SUM(area) as area_sum
FROM
(
    SELECT /*+ mapjoin(boundary) */
  		boundary.cityID, 
  		builtup.area,
      pnpoly(builtup.lon, builtup.lat, boundary.polygon) AS inPoly 
    FROM
      boundary, builtup
    WHERE 
  		builtup.lon < boundary.maxlon	AND 
    	builtup.lon > boundary.minlon	AND 
    	builtup.lat < boundary.maxlat	AND 
    	builtup.lat > boundary.minlat	
)
WHERE inPoly = true
GROUP BY cityID
```



#### Population

```sql
SELECT 
	cityID, SUM(population) as population_sum
FROM
(
    SELECT /*+ mapjoin(boundary) */
  		boundary.cityID, 
  		population.population,
      pnpoly(population.lon, population.lat, boundary.polygon) AS inPoly 
    FROM
      boundary, population
    WHERE
  		population.lon < boundary.maxlon	AND 
    	population.lon > boundary.minlon	AND 
    	population.lat < boundary.maxlat	AND 
    	population.lat > boundary.minlat	
      
)
WHERE inPoly = true
GROUP BY cityID
```





## Query in Raster Representation



### Table Description

latitude as the row index

longitude as the column index

#### table: boundaryRaster

<ul>
  <li> <b>cityID</b>: ID of the city </li>
  <li> <b>row, col</b>: the cell (row, col) within the city boundary  </li>
</ul>


#### table: roadRaster

<ul>
  <li> <b>start_row, start_col</b>: the cell index of the start node of the road edge </li>
  <li> <b>end_row, end_col</b>: the cell index of the end node of the road edge </li>
</ul>


#### table: lightRaster

<ul>
  <li> <b>row, col</b>: the cell index (row, col) </li>
  <li> <b>light</b>: nighttime light intensity </li>
</ul>


#### builtupRaster: built-up Surface

<ul>
  <li> <b>row, col</b>: the cell index (row, col) </li>
  <li> <b>area</b>: built-up surface area  </li>
</ul>


#### population: World Population

<ul>
  <li> <b>row, col</b>: the cell index (row, col) </li>
  <li> <b>population</b>: population  </li>
</ul>


#### 

### Query 21,280 Global Cities

#### Road Length

```sql
SELECT 
	cityID, SUM(distance) AS distance_sum
FROM
(
	SELECT 
  	b.cityID,
    geo_distance(r.start_col/240-180, r.start_row/240-90, r.end_col/240-180, r.end_row/240-90) AS distance   
  FROM 
  	boundaryRaster AS b, 
  	roadRaster AS r
  WHERE 
  	b.row = r.start_row AND 
    b.col = r.start_col
)
GROUP BY cityID
```

####  Nighttime Light

```sql
SELECT 
	cityID, SUM(light) AS light_sum
FROM
(
	SELECT 
  	boundaryRaster.cityID,
		lightRaster.light
  FROM 
  	boundaryRaster, 
  	lightRaster
  WHERE 
  	boundaryRaster.row = lightRaster.row AND 
  	boundaryRaster.col = lightRaster.col  
)
group by cityID
```

####  Built-up Surface Area

```sql
SELECT 
	cityID, SUM(area) AS area_sum
FROM
(
	SELECT 
  	boundaryRaster.cityID,
		builtupRaster.area
  FROM 
  	boundaryRaster, 
  	builtupRaster
  WHERE 
  	boundaryRaster.row = builtupRaster.row AND 
  	boundaryRaster.col = builtupRaster.col  
)
group by cityID
```

#### Population

```sql
SELECT 
	cityID, SUM(population) AS population_sum
FROM
(
	SELECT 
  	boundaryRaster.cityID,
		populationRaster.population
  FROM 
  	boundaryRaster, 
  	populationRaster
  WHERE 
  	boundaryRaster.row = populationRaster.row AND 
  	boundaryRaster.col = populationRaster.col  
)
group by cityID
```

####  

