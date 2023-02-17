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
	cityID, SUM(distance) as distance_sum /* sum all the edge distance */
FROM
(
    SELECT /* calculate the distance for a road edge that is within the city boundary */
  		cityID, 
  		geo_distance(lon1, lat1, lon2, lat2) AS distance
    FROM 
  	( /* retrieve the edges with starting node inside city boundary */
  		SELECT
      	boundary.cityID
      	road.node_start_lon AS lon1, 
      	road.node_start_lat AS lat1, 
      	road.node_end_lon AS lon2, 
      	road.node_end_lat AS lat2,
        pnpoly(road.node_start_lon, road.node_start_lat, boundary.polygon) AS inPoly 
        /* if the start node of the edge is in the polygon, we treat this edge in the polygon */
      FROM
      	boundary, road
    	WHERE /*bounding box filter*/
  			road.node_start_lon < boundary.maxlon	AND 
    		road.node_start_lon > boundary.minlon	AND 
    		road.node_start_lat < boundary.maxlat	AND 
    		road.node_start_lat > boundary.minlat	AND
      	inPoly = TRUE
     )
)
GROUP BY cityID;
```



#### Nighttime Light

```sql
SELECT 
	cityID, SUM(light) as light_sum
FROM
(
    SELECT 
  		boundary.cityID, 
  		light.light,
      pnpoly(light.lon, light.lat, boundary.polygon) AS inPoly 
    FROM
      boundary, light
    WHERE /*bounding box filter*/
  		light.lon < boundary.maxlon	AND 
    	light.lon > boundary.minlon	AND 
    	light.lat < boundary.maxlat	AND 
    	light.lat > boundary.minlat	AND 
      inPoly = TRUE
)
GROUP BY cityID;
```



#### Built-up Surface Area

```sql
SELECT 
	cityID, SUM(area) as area_sum
FROM
(
    SELECT 
  		boundary.cityID, 
  		builtup.area,
      pnpoly(builtup.lon, builtup.lat, boundary.polygon) AS inPoly 
    FROM
      boundary, builtup
    WHERE /*bounding box filter*/
  		builtup.lon < boundary.maxlon	AND 
    	builtup.lon > boundary.minlon	AND 
    	builtup.lat < boundary.maxlat	AND 
    	builtup.lat > boundary.minlat	AND 
      inPoly = TRUE
)
GROUP BY cityID;
```



#### Population

```sql
SELECT 
	cityID, SUM(population) as population_sum
FROM
(
    SELECT 
  		boundary.cityID, 
  		population.population,
      pnpoly(population.lon, population.lat, boundary.polygon) AS inPoly 
    FROM
      boundary, population
    WHERE /*bounding box filter*/
  		population.lon < boundary.maxlon	AND 
    	population.lon > boundary.minlon	AND 
    	population.lat < boundary.maxlat	AND 
    	population.lat > boundary.minlat	AND 
      inPoly = TRUE
)
GROUP BY cityID;
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
  	/* if the start node of the edge is in the polygon, we treat this edge in the polygon */
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

