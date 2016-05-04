
# Graph hack 2016 @ GraphConnect Europe
## Using Neo4j with transport data

### Team: Crash Dodgers: Adam Hill & Amy McQuillan
### Concept: Combine Santander Bike data with cyclist accident data to find dangerous places to hire bikes in London

First up lets find all the Santander bike stations in London
* Cycle hire updates with all stations are available from the TfL API here: https://tfl.gov.uk/tfl/syndication/feeds/cycle-hire/livecyclehireupdates.xml
* To make my life easier used http://codebeautify.org/xmltojson to convert to JSON and save the file locally


```python
import pandas as pd
import numpy as np
import json
```


```python
stations = json.load(open('./GraphHackData/bikeStation.json', 'r'))
stationsDF = pd.DataFrame(stations['stations']['station'])
stationsDF.tail()
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>id</th>
      <th>installDate</th>
      <th>installed</th>
      <th>lat</th>
      <th>locked</th>
      <th>long</th>
      <th>name</th>
      <th>nbBikes</th>
      <th>nbDocks</th>
      <th>nbEmptyDocks</th>
      <th>removalDate</th>
      <th>temporary</th>
      <th>terminalName</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>754</th>
      <td>794</td>
      <td>1456404240000</td>
      <td>true</td>
      <td>51.474567</td>
      <td>false</td>
      <td>-0.12458</td>
      <td>Lansdowne Way Bus Garage, Stockwell</td>
      <td>15</td>
      <td>28</td>
      <td>13</td>
      <td></td>
      <td>false</td>
      <td>300204</td>
    </tr>
    <tr>
      <th>755</th>
      <td>795</td>
      <td>1456744740000</td>
      <td>true</td>
      <td>51.527566</td>
      <td>false</td>
      <td>-0.13484927</td>
      <td>Melton Street, Euston</td>
      <td>5</td>
      <td>28</td>
      <td>23</td>
      <td></td>
      <td>false</td>
      <td>300203</td>
    </tr>
    <tr>
      <th>756</th>
      <td>800</td>
      <td>1457107140000</td>
      <td>true</td>
      <td>51.4811219398</td>
      <td>false</td>
      <td>-0.149035374873</td>
      <td>Sopwith Way, Battersea Park</td>
      <td>23</td>
      <td>30</td>
      <td>7</td>
      <td></td>
      <td>false</td>
      <td>300248</td>
    </tr>
    <tr>
      <th>757</th>
      <td>801</td>
      <td></td>
      <td>true</td>
      <td>51.5052241745</td>
      <td>false</td>
      <td>-0.0980318118664</td>
      <td>Lavington Street, Bankside</td>
      <td>26</td>
      <td>29</td>
      <td>3</td>
      <td></td>
      <td>false</td>
      <td>300208</td>
    </tr>
    <tr>
      <th>758</th>
      <td>804</td>
      <td></td>
      <td>true</td>
      <td>51.5346677396</td>
      <td>false</td>
      <td>-0.125078652873</td>
      <td>Good's Way, King's Cross</td>
      <td>17</td>
      <td>27</td>
      <td>10</td>
      <td></td>
      <td>false</td>
      <td>300243</td>
    </tr>
  </tbody>
</table>
</div>



#### Let's store these stations as the first nodes of our graph


```python
from py2neo import Graph
from py2neo import Node, Relationship
```


```python
graph = Graph("http://neo4j:password@localhost:7474/db/data")
```


```python
#Loop over all bike stations and store their details in Neo4j
for r, data in stationsDF.iterrows():
    tempNode = Node("Bike_station", a_Id = np.int(data['id']))
    tempNode['latitude'] = np.float(data.lat)
    tempNode['longitude'] = np.float(data.long)
    tempNode['name'] = data['name']
    tempNode['installDate'] = data.installDate
    tempNode['num_docks'] = np.int(data.nbDocks)
    graph.create(tempNode)
```

### Traffic accidents in the UK
We look at data from 2014 regarding traffic accidents across the UK from here https://data.gov.uk/dataset/road-accidents-safety-data . We used the following files:
* 2014 Road Safety - Accidents 2014
* 2014 Road Safety - Vehicles 2014
* 2014 Road Safety - Casualties 2014
* Lookup up tables for variables
Local copies were downloaded and stored in ./GraphHackData



```python
accidents = pd.read_csv('./GraphHackData/DfTRoadSafety_Accidents_2014.csv')
vehicles = pd.read_csv('./GraphHackData/DfTRoadSafety_Vehicles_2014.csv')
casualties = pd.read_csv('./GraphHackData/DfTRoadSafety_Casualties_2014.csv')
```

Some strange characters are in some of the column names so let's trip them out and then then merge accidents and casualties on Accident_Index


```python
accidents = accidents.rename(columns={'﻿Accident_Index': 'Accidents_Index'})
vehicles = vehicles.rename(columns={'﻿Accident_Index': 'Accidents_Index'})
casualties = casualties.rename(columns={'﻿Accident_Index': 'Accidents_Index'})
```


```python
accidentsDF = pd.merge(accidents, casualties, on='Accidents_Index')
```


```python
accidentsDF.head()
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Accidents_Index</th>
      <th>Location_Easting_OSGR</th>
      <th>Location_Northing_OSGR</th>
      <th>Longitude</th>
      <th>Latitude</th>
      <th>Police_Force</th>
      <th>Accident_Severity</th>
      <th>Number_of_Vehicles</th>
      <th>Number_of_Casualties</th>
      <th>Date</th>
      <th>...</th>
      <th>Age_of_Casualty</th>
      <th>Age_Band_of_Casualty</th>
      <th>Casualty_Severity</th>
      <th>Pedestrian_Location</th>
      <th>Pedestrian_Movement</th>
      <th>Car_Passenger</th>
      <th>Bus_or_Coach_Passenger</th>
      <th>Pedestrian_Road_Maintenance_Worker</th>
      <th>Casualty_Type</th>
      <th>Casualty_Home_Area_Type</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>201401BS70001</td>
      <td>524600</td>
      <td>179020</td>
      <td>-0.206443</td>
      <td>51.496345</td>
      <td>1</td>
      <td>3</td>
      <td>2</td>
      <td>1</td>
      <td>09/01/2014</td>
      <td>...</td>
      <td>49</td>
      <td>8</td>
      <td>3</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>8</td>
      <td>1</td>
    </tr>
    <tr>
      <th>1</th>
      <td>201401BS70002</td>
      <td>525780</td>
      <td>178290</td>
      <td>-0.189713</td>
      <td>51.489523</td>
      <td>1</td>
      <td>3</td>
      <td>2</td>
      <td>1</td>
      <td>20/01/2014</td>
      <td>...</td>
      <td>27</td>
      <td>6</td>
      <td>3</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>1</td>
      <td>-1</td>
    </tr>
    <tr>
      <th>2</th>
      <td>201401BS70003</td>
      <td>526880</td>
      <td>178430</td>
      <td>-0.173827</td>
      <td>51.490536</td>
      <td>1</td>
      <td>3</td>
      <td>2</td>
      <td>1</td>
      <td>21/01/2014</td>
      <td>...</td>
      <td>27</td>
      <td>6</td>
      <td>3</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>3</td>
      <td>1</td>
    </tr>
    <tr>
      <th>3</th>
      <td>201401BS70004</td>
      <td>525580</td>
      <td>179080</td>
      <td>-0.192311</td>
      <td>51.496668</td>
      <td>1</td>
      <td>3</td>
      <td>1</td>
      <td>1</td>
      <td>15/01/2014</td>
      <td>...</td>
      <td>31</td>
      <td>6</td>
      <td>3</td>
      <td>1</td>
      <td>1</td>
      <td>0</td>
      <td>0</td>
      <td>2</td>
      <td>0</td>
      <td>1</td>
    </tr>
    <tr>
      <th>4</th>
      <td>201401BS70006</td>
      <td>527040</td>
      <td>179030</td>
      <td>-0.171308</td>
      <td>51.495892</td>
      <td>1</td>
      <td>3</td>
      <td>2</td>
      <td>1</td>
      <td>09/01/2014</td>
      <td>...</td>
      <td>32</td>
      <td>6</td>
      <td>3</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>9</td>
      <td>1</td>
    </tr>
  </tbody>
</table>
<p>5 rows × 46 columns</p>
</div>



### Which accidents are in London?
Santander bikes are only available in London so we also need to be able to filter by whether an accident is in London. Accidents are all assigned to LSOA (Lower Layer Super Output Area).

We identify all the LSOAs in London using this ref: http://data.london.gov.uk/dataset/lsoa-atlas


```python
london = pd.read_excel('./GraphHackData/lsoa-data.xls', sheet='iadatasheet1', skiprows=2)
lsoa = set(london.Codes)
```


```python
#Add a boolean column to the accidents dataframe to describe if in London
accidentsDF['in_London'] = accidentsDF.LSOA_of_Accident_Location.map(lambda x: x in lsoa)
```

From the DoT lookup table we identify that any casuality listed as 1 is a cyclist and hence we can now find all accidents in London in 2014 where the casuality was a cyclist


```python
cyclingAccidents = accidentsDF[(accidentsDF['in_London'] == True) & (accidentsDF.Casualty_Type == 1)]
```


```python
#Example incident
example = cyclingAccidents.loc[59,]
example
```




    Accidents_Index                                201401BS70065
    Location_Easting_OSGR                                 526610
    Location_Northing_OSGR                                177280
    Longitude                                          -0.178126
    Latitude                                             51.4803
    Police_Force                                               1
    Accident_Severity                                          3
    Number_of_Vehicles                                         2
    Number_of_Casualties                                       1
    Date                                              08/02/2014
    Day_of_Week                                                7
    Time                                                   18:20
    Local_Authority_(District)                                12
    Local_Authority_(Highway)                          E09000020
    1st_Road_Class                                             3
    1st_Road_Number                                         3220
    Road_Type                                                  6
    Speed_limit                                               30
    Junction_Detail                                            0
    Junction_Control                                          -1
    2nd_Road_Class                                            -1
    2nd_Road_Number                                            0
    Pedestrian_Crossing-Human_Control                          0
    Pedestrian_Crossing-Physical_Facilities                    5
    Light_Conditions                                           4
    Weather_Conditions                                         2
    Road_Surface_Conditions                                    2
    Special_Conditions_at_Site                                 0
    Carriageway_Hazards                                        0
    Urban_or_Rural_Area                                        1
    Did_Police_Officer_Attend_Scene_of_Accident                2
    LSOA_of_Accident_Location                          E01002840
    Vehicle_Reference                                          2
    Casualty_Reference                                         1
    Casualty_Class                                             1
    Sex_of_Casualty                                            1
    Age_of_Casualty                                           32
    Age_Band_of_Casualty                                       6
    Casualty_Severity                                          3
    Pedestrian_Location                                        0
    Pedestrian_Movement                                        0
    Car_Passenger                                              0
    Bus_or_Coach_Passenger                                     0
    Pedestrian_Road_Maintenance_Worker                         0
    Casualty_Type                                              1
    Casualty_Home_Area_Type                                   -1
    in_London                                               True
    Name: 59, dtype: object



Let's see what the two nearest Santander bike stations are to this accident ...


```python
query = "MATCH (b:Bike_station) WITH b, distance(point(b), point({{latitude:{0}, longitude:{1}}})) AS dist RETURN b.a_Id AS station_id, b.name AS station_name, dist ORDER BY dist LIMIT 2".format(example.Latitude, example.Longitude)
cypher = graph.cypher
result = cypher.execute(query)
```


```python
result
```




       | station_id | station_name                    | dist             
    ---+------------+---------------------------------+-------------------
     1 |        746 | Lots Road, West Chelsea         | 99.27002411121134
     2 |        649 | World's End Place, West Chelsea |  227.456404989975



### So first piece of insight appears to be not to cylce at "World's End"!

Before generating the graph of the accidents we need to convert many of the numerical classifications into their human readable form to make things easier to interpret


```python
#Conversion for some of the accident variables to huamn readable form

roadClass = {1: "Motorway",
             2: "A(M)",
             3: "A",
             4: "B",
             5: "C",
             6: "Unclassified"}

dow = {1: "Sunday",
       2: "Monday",
       3: "Tuesday",
       4: "Wednesday",
       5: "Thursday",
       6: "Friday",
       7: "Saturday"}
       
lightConditions = {1: "Daylight",
                   4: "Darkness: lights lit",
                   5: "Darkness: lights unlit",
                   6: "Darkness: no lighting",
                   7: "Darkness: lighting unknown",
                   -1: "Data missing"}

weatherConditions = {1:"Fine no high winds",
                     2:"Raining no high winds",
                     3:"Snowing no high winds",
                     4:"Fine + high winds",
                     5:"Raining + high winds",
                     6:"Snowing + high winds",
                     7:"Fog or mist",
                     8:"Other",
                     9:"Unknown",
                     -1:"Data missing"}
       
roadConditions = {1: "Dry",
                  2: "Wet or damp",
                  3: "Snow",
                  4: "Frost or ice",
                  5: "Flood over 3cm deep",
                  6: "Oil or diesel",
                  7: "Mud",
                 -1: "Data missing"}

gender = {1: "Male",
          2: "Female",
          3: "Not known",
          -1: "Data missing"}

severity ={1: "Fatal",
           2: "Serious",
           3: "Slight"}
```

### Now we are ready to generate accident nodes and map them to the two nearest bike stations
##### N.B. we will not map if the nearest bike station isn't within 2km of an accident as Santander bike stations are concentrated in the centre rather than across the whole of what is labelled London


```python
def genAccidentNodes(datum):
    """For a given row in the accidents dataframe construct the appropriate set of nodes and relationships"""
    accident = Node("Accident", a_Id = datum.Accidents_Index)
    accident['latitude'] = datum.Latitude
    accident['longitude'] = datum.Longitude
    accident['severity'] = severity[datum.Casualty_Severity]
    accident['severity_score'] = 4. - datum.Casualty_Severity
    accident['time'] = datum.Time
    graph.create(accident)
    date = graph.merge_one('Date', "value", datum.Date)
    date.properties['day_of_week'] = dow[datum.Day_of_Week]
    graph.push(date)
    rel_1 = Relationship(accident, "HAPPENED_ON", date)

    #Make vector of relationships to create
    relationships = []
    relationships.append(rel_1) 
    
    weatherCon = weatherConditions.get(datum.Weather_Conditions, "Data missing")
    if weatherCon != "Data missing":
        weather = graph.merge_one('Weather', "condition", weatherCon)
        relationships.append(Relationship(accident, "WITH", weather))
    lightCon = lightConditions.get(datum.Light_Conditions, "Data missing")
    if lightCon != "Data missing":
        light = graph.merge_one('Light', "condition", lightCon)
        relationships.append(Relationship(accident, "WITH", light))
    roadSurfaceCon = roadConditions.get(datum.Road_Surface_Conditions, "Data missing")
    if roadSurfaceCon != "Data missing":
        roadSurf = graph.merge_one('Road_surface', "condition", roadSurfaceCon)
        relationships.append(Relationship(accident, "WITH", roadSurf))
        
    speed = graph.merge_one("Speed_limit", "value", np.int(datum.Speed_limit))
    relationships.append(Relationship(accident, "WITH", speed))
    
    #And find the nearest bike stations
    query = "MATCH (b:Bike_station) WITH b, distance(point(b), point({{latitude:{0}, longitude:{1}}})) AS dist RETURN b.a_Id AS station, dist ORDER BY dist LIMIT 2".format(datum.Latitude, datum.Longitude)
    result = cypher.execute(query)
    #Only do this for bike sations where the nearest station is less than 2km away
    if result.records[0].dist <= 2000.:
        for i,rec in enumerate(result.records):
            bikeStation = graph.merge_one("Bike_station", "a_Id", rec.station)
            tempRel = Relationship(accident, "CLOSE_TO", bikeStation, distance=round(rec.dist,2), proximity=i+1)
            relationships.append(tempRel)
    
    graph.create(*relationships)
```

Let's run the function over all London cycling accidents


```python
output = cyclingAccidents.apply(genAccidentNodes, axis=1)
```
### Graphically exploring our new database

#### An accident node and associated properties including closest bike docking stations
![png](images/AccidentNode.png)
#### Which bike docking stations are linked to the fatal cycling accidents?
![png](images/Accident_Station_Nodes.png)
#### What was the speed limit on the roads with fatal cycling accidents?
![png](images/Accidents_Speed.png)

## The most dangerous bike docking stations to cycle between
We can query neo4j to count the number of accidents between bike stations


```python
query = """MATCH (b1:Bike_station)<-[:CLOSE_TO]-(a:Accident)-[:CLOSE_TO]->(b2:Bike_station) 
WITH b1, b2, COLLECT(DISTINCT a.a_Id) AS accidents
WHERE b1.a_Id < b2.a_Id
RETURN b1.name AS station1, b1.longitude AS lon1, b1.latitude AS lat1, 
       b2.name AS station2, b2.longitude AS lon2, b2.latitude AS lat2,
       size(accidents) AS num_accidents 
ORDER BY num_accidents DESC;"""
result = cypher.execute(query)
```

### The top 10 most dangerous bike station pairs are ...


```python
df = pd.DataFrame(result.records, columns=result.columns)
df.head(10)
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>station1</th>
      <th>lon1</th>
      <th>lat1</th>
      <th>station2</th>
      <th>lon2</th>
      <th>lat2</th>
      <th>num_accidents</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>Clarence Walk, Stockwell</td>
      <td>-0.126994</td>
      <td>51.470733</td>
      <td>Binfield Road, Stockwell</td>
      <td>-0.122832</td>
      <td>51.472510</td>
      <td>61</td>
    </tr>
    <tr>
      <th>1</th>
      <td>Shoreditch Court, Haggerston</td>
      <td>-0.070329</td>
      <td>51.539084</td>
      <td>Haggerston Road, Haggerston</td>
      <td>-0.074285</td>
      <td>51.539329</td>
      <td>48</td>
    </tr>
    <tr>
      <th>2</th>
      <td>Islington Green, Angel</td>
      <td>-0.102758</td>
      <td>51.536384</td>
      <td>Charlotte Terrace, Angel</td>
      <td>-0.112721</td>
      <td>51.536392</td>
      <td>32</td>
    </tr>
    <tr>
      <th>3</th>
      <td>Ravenscourt Park Station, Hammersmith</td>
      <td>-0.236770</td>
      <td>51.494224</td>
      <td>Hammersmith Town Hall, Hammersmith</td>
      <td>-0.234094</td>
      <td>51.492637</td>
      <td>32</td>
    </tr>
    <tr>
      <th>4</th>
      <td>Bricklayers Arms, Borough</td>
      <td>-0.085814</td>
      <td>51.495061</td>
      <td>Rodney Road , Walworth</td>
      <td>-0.090221</td>
      <td>51.491485</td>
      <td>30</td>
    </tr>
    <tr>
      <th>5</th>
      <td>Napier Avenue, Millwall</td>
      <td>-0.021582</td>
      <td>51.487679</td>
      <td>Spindrift Avenue, Millwall</td>
      <td>-0.018716</td>
      <td>51.491090</td>
      <td>28</td>
    </tr>
    <tr>
      <th>6</th>
      <td>Ada Street, Hackney Central</td>
      <td>-0.060292</td>
      <td>51.535717</td>
      <td>Victoria Park Road, Hackney Central</td>
      <td>-0.054162</td>
      <td>51.536425</td>
      <td>26</td>
    </tr>
    <tr>
      <th>7</th>
      <td>Wandsworth Rd, Isley Court, Wandsworth Road</td>
      <td>-0.141813</td>
      <td>51.469260</td>
      <td>Heath Road, Battersea</td>
      <td>-0.146545</td>
      <td>51.468669</td>
      <td>24</td>
    </tr>
    <tr>
      <th>8</th>
      <td>Caldwell Street, Stockwell</td>
      <td>-0.116493</td>
      <td>51.477839</td>
      <td>Binfield Road, Stockwell</td>
      <td>-0.122832</td>
      <td>51.472510</td>
      <td>23</td>
    </tr>
    <tr>
      <th>9</th>
      <td>Stebondale Street, Cubitt Town</td>
      <td>-0.009205</td>
      <td>51.489096</td>
      <td>Saunders Ness Road, Cubitt Town</td>
      <td>-0.009001</td>
      <td>51.487129</td>
      <td>23</td>
    </tr>
  </tbody>
</table>
</div>



### Plotting everything out looks like this


```python
df['mean_longitude'] = (df.lon1+df.lon2)/2.
df['mean_latitude'] = (df.lat1+df.lat2)/2.
df2 = df[df.num_accidents > 10]
```


```python
%matplotlib inline
import geopandas as gpd
import matplotlib.pyplot as plt
import mplleaflet

plt.rcParams['figure.figsize'] = 14, 10
fig = plt.figure()
plt.plot(cyclingAccidents.Longitude, cyclingAccidents.Latitude, 'b.', alpha=0.4, label='cycling accidents (2014)')
plt.plot(stationsDF.long, stationsDF.lat, 'rs', alpha=0.5, label='docking stations')
plt.scatter(df2.mean_longitude, df2.mean_latitude, s=df2.num_accidents*25, c='yellow', alpha=0.5, label='Most accidents')
plt.xlim(-0.25, 0.)
plt.ylim(51.45, 51.55)
#plt.legend(loc='lower right')
```




    (51.45, 51.55)




![png](images/output_36_1.png)


### Can combine this with a map to get a better feel for where in London we are


```python
mplleaflet.display(fig=fig, tiles='osm')
```

![png](images/FinalLondonMap.png)




## Conclusions
### Some interesting first results but more work needed

It's great to see that the intial analysis worked and using Neo4j made our analysis easier and there is a lot more data stored in there that could be analysed at a later date. What at first appears surprising is that the "danger stations" that we have identified appear to gnerally bound the region that Santander bikes are available in, however, we cannot confirm these are correct without correcting for a couple of additional observational biases:

1. Normalise for the amount of journeys starting/ending at each station, i.e. do more accidents happen because more people are riding in these parts of London.
2. The density of bike docking stations is not uniform so those on the periphery may be being assigned more accidents based upon fewer stations to assign the accidents to.
