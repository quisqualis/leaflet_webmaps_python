
# Creating Web Maps in Python with GeoPandas and Folium

## Introduction


In this post, I demonstrate the use of the Python package Folium to create a web map from a GeoDataFrame. Folium is built on the Leaflet javascript library, which is a great tool for creating interactive web maps. However, I use Python for all of my data wrangling and analytical tasks, so it's really nice to be able to have the web-mapping capabilities from within the same environment. The goal of this post is to demonstrate a workflow between GeoPandas and Folium that makes it really easy to create functional and visually appealing web maps in Python.

In this example, I plot the point locations of crimes in San Francisco, overlaid on a choropleth of census tract crime density. Viewing these two layers together on a web map creates a nice way to get an overall sense of crime distribution, while also being able to view individual crime information. As I demonstrate below, these Python packages provide a nice, clean, and customizable way of doing this.


```python
#Import the necessary Python moduless
import pandas as pd
import geopandas as gpd
import numpy as np
from geopandas.tools import sjoin
import folium
from folium.plugins import MarkerCluster
from folium.element import IFrame
import shapely
from shapely.geometry import Point
import unicodedata
import pysal as ps
```

## Data Prep
In this section I create two GeoDataFrames: one of crime points and one of census tract boundaries with crime densities. Both of these will then be plotted on a web map as separate layers.

### Read in Crime Data and Create a GeoDataFrame
First I read in a CSV file of San Francisco Police Incidents for the current year into a Pandas DataFrame. I downloaded the raw data from the San Francisco [Open Data Portal](https://data.sfgov.org/). Because there are so many crime incidents, I select a subset of the data: crimes in the "assault" category that were committed in the last 10 days. As shown below, this leaves me with 329 police incidents. 



```python
#Read in CSV file specifying date field and encoding. Sort by date
all_crime = pd.read_csv('SFPD_Incidents_-_Current_Year__2016_.csv', parse_dates = ['Date'],\
                        encoding = 'utf-8').sort_values('Date').reset_index(drop=True)

#Clean up the encoding on the Crime Description field 
all_crime.Descript = all_crime.Descript.apply(lambda x: unicodedata.normalize("NFKD", x))

#Create a field that contains a string representation of the date, for later use
all_crime['DateStr'] = all_crime.Date.apply(lambda x: x.strftime("%Y-%m-%d"))

#Identify those crimes that are categorized as assaults
is_assault = all_crime.Category == 'ASSAULT' 

#Identify those crimes that were committed in the most recent 10 days of the dataset
recent = all_crime.Date.isin(all_crime.Date.unique()[-10:]) 

#Subset the data to get assaults commited within the last 10 days
assaults = all_crime[is_assault & recent].drop_duplicates('IncidntNum').reset_index(drop = True)
assaults = assaults[['IncidntNum', 'Descript', 'DateStr', 'Time', 'Address','X', 'Y']]
print '{} assaults in the last 10 days'.format(str(len(assaults)))
```

    329 assaults in the last 10 days
    

I now want to convert the assault data Pandas DataFrame to a GeoPandas GeoDataFrame (a spatial version of the former). The raw crime data comes with lat/long coordinates, which I use these to create Shapely point geometry objects (these are the values in the "geometry" field for each record in a GeoDataFrame). I specify the spatial reference system as ESPG 4326 which represents the standard WGS84 coordinate system.


```python
#First create a GeoSeries of crime locations by converting coordinates to Shapely geometry objects
#Specify the coordinate system ESPG4326 which represents the standard WGS84 coordinate system
assault_geo = gpd.GeoSeries(assaults.apply(lambda z: Point(z['X'], z['Y']), 1),crs={'init': 'epsg:4326'})

#Create a geodataframe from the pandas dataframe and the geoseries of shapely geometry objects
assaults = gpd.GeoDataFrame(assaults.drop(['X', 'Y'], 1), geometry=assault_geo)
print assaults.head()
```

       IncidntNum                     Descript     DateStr   Time  \
    0   160885198         THREATS AGAINST LIFE  2016-10-30  14:20   
    1   160883142  BATTERY OF A POLICE OFFICER  2016-10-30  02:00   
    2   160883158                      BATTERY  2016-10-30  02:00   
    3   160883073                      BATTERY  2016-10-30  01:35   
    4   160882928  BATTERY OF A POLICE OFFICER  2016-10-30  00:29   
    
                              Address                                     geometry  
    0        0 Block of MCALLISTER ST   POINT (-122.412596970637 37.7811192121542)  
    1        400 Block of BROADWAY ST   POINT (-122.405065483077 37.7980134745487)  
    2  500 Block of BUENAVISTAWEST AV  POINT (-122.443281739239 37.76645765488571)  
    3          400 Block of POWELL ST   POINT (-122.408431861057 37.7887772719153)  
    4              0 Block of HOFF ST   POINT (-122.420575720933 37.7641819463712)  
    

### Calculate Census Tract Crime Density
Next, I read in a Shapefile of San Francisco Census Tracts, which I also downloaded from the SF Open Data Portal. With GeoPandas, I can read a Shapefile directly into Python really easily. Then in one line of code, I spatially join census tracts to assaults (determine the census tract of each assault), and generate counts of assaults per census tract. Note that I use the ```to_crs``` function to convert assaults to the same coordinate system as Census Tracts (EPSG 3310) prior to spatially joining them.

Lastly, I calculate the number of assaults per square mile, which is the metric that I'm interested in plotting.


```python
#Read tracts shapefile into GeoDataFrame
tracts = gpd.read_file('sf_census_tracts.shp').set_index('CTFIPS10')

#Generate Counts of Assaults per Census Tract
#Spatially join census tracts to assaults (after projecting) and then group by Tract FIPS while counting the number of crimes
tract_counts = gpd.tools.sjoin(assaults.to_crs(tracts.crs), tracts.reset_index()).groupby('CTFIPS10').size()

#Calculate Assault Density, converting square meters to square miles.
tracts['AssaultsPSqMi'] = (tract_counts/(tracts.geometry.area*3.86102e-7)).fillna(0)
tracts = tracts.reset_index()
print tracts.head()
```

          CTFIPS10                                           geometry  \
    0  06075010100  POLYGON ((-212660.1301711957 -20053.0335317570...   
    1  06075010200  (POLYGON ((-212986.3528985226 -20191.607399463...   
    2  06075010300  POLYGON ((-212512.5989250286 -20763.4272515336...   
    3  06075010400  POLYGON ((-211456.8561540585 -20837.2873740978...   
    4  06075010500  POLYGON ((-211050.6276144625 -20707.0181740056...   
    
       AssaultsPSqMi  
    0       3.423806  
    1      19.871972  
    2       0.000000  
    3       7.721341  
    4      15.009851  
    

## Using Folium to Plot Data
The general approach I take here is to first create a Folium basemap and then add two layers to it: (1) a choropleth of census tracts, symbolized crime density, and (2) crime point locations. I write a separate function to plot each of these two layers, each of which takes a GeoDataFrame as its input. Folium takes unprojected lat/long coordinates for all of its plotting, so I make sure to convert all of my projected GeoDataFrames to WGS84 within the plotting functions.

###  Choropleth Layer of Tract Crime Density
As its inputs, my choropleth function takes a Folium map object, a GeoDataFrame, the name of the feature ID field, and the name of the field whose values will be symbolized. 

Leaflet uses GeoJSON objects to plot vector geometries (GeoJSON is a data format that is used to represent geographical features along with their non-spatial attributes). GeoPandas has a ```to_json``` method which I use to convert the GeoDataFrame to GeoJSON to be used as one of the inputs to Folium's choropleth function. I also specify the id field, which is used to link the geometry in the GeoJSON with the data in the GeoDataFrame.

The function also takes optional parameters for fill color, fill opacity, line opacity, number of classifiers, and classification scheme. All of these have default values if not specified. Folium / Leaflet uses the [Color Brewer](http://colorbrewer2.org/#type=sequential&scheme=YlOrRd&n=5) sequential color schemes, which can easily be specified to view different combinations. 

Lastly, I allow the user to specify the number of classes and the classification scheme. At this point, Folium has limited map classification options, so I instead use Pysal's choropleth [map classfication module](http://pysal.readthedocs.io/en/latest/library/esda/mapclassify.html) to provide some basic classification options. My function defaults to "Fisher_Jenks", but also has options for "Equal_Interval", and "Quantiles". 

Below, I first create a basemap that is centered in San Francisco, and then I run my function on this basemap specifying the Census Tract ID Field as well as the field I want to symbolize on (Assaults Per Square Mile).


```python
#Create SF basemap specifying map center, zoom level, and using the default OpenStreetMap tiles
crime_map = folium.Map([37.7556, -122.4399], zoom_start = 12)

def add_choropleth(mapobj, gdf, id_field, value_field, fill_color = 'YlOrRd', fill_opacity = 0.6, 
                    line_opacity = 0.2, num_classes = 5, classifier = 'Fisher_Jenks'):
    #Allow for 3 Pysal map classifiers to display data
    #Generate list of breakpoints using specified classification scheme. List of breakpoint will be input to choropleth function
    if classifier == 'Fisher_Jenks':
        threshold_scale=ps.esda.mapclassify.Fisher_Jenks(gdf[value_field], k = num_classes).bins.tolist()
    if classifier == 'Equal_Interval':
        threshold_scale=ps.esda.mapclassify.Equal_Interval(gdf[value_field], k = num_classes).bins.tolist()
    if classifier == 'Quantiles':
        threshold_scale=ps.esda.mapclassify.Quantiles(gdf[value_field], k = num_classes).bins.tolist()
    
    #Convert the GeoDataFrame to WGS84 coordinate reference system
    gdf_wgs84 = gdf.to_crs({'init': 'epsg:4326'})
    
    #Call Folium choropleth function, specifying the geometry as a the WGS84 dataframe converted to GeoJSON, the data as 
    #the GeoDataFrame, the columns as the user-specified id field and and value field.
    #key_on field refers to the id field within the GeoJSON string
    mapobj.choropleth(geo_str = gdf_wgs84.to_json(), data = gdf,
                columns = [id_field, value_field], key_on = 'feature.properties.{}'.format(id_field),
                fill_color = fill_color, fill_opacity = fill_opacity, line_opacity = line_opacity,  
                threshold_scale = threshold_scale)
    return mapobj

#Update basemap with choropleth
crime_map=add_choropleth(crime_map, tracts, 'CTFIPS10','AssaultsPSqMi')
```

### Create Crime Point Cluster Layer
Before displaying my map, I will also add the layer of crime point locations. Rather than display each individual point, I use Leaflet's marker clustering feature, which makes it easier to visualize large numbers of points by grouping together those that are close to each other. Additionally, I use popups to display information about a particular crime when the user clicks on a point. Folium lets you create HTML-rich popups called IFrames. I use this feature only in the most basic form, just to display a few lines of information: crime description, date, time, and address. There are obviously much more creative things that can be done with an IFrame popup (tables, graphs, sub-maps, etc.) but for my purposes this is all I need. 

My function takes as its inputs a Folium map object, a GeoDataFrame, and a list of fields to include in the popup. I run this function on my previously created map object (already updated with a choropleth layer), specifying 4 fields of interest that I want to display. 



```python
def add_point_clusters(mapobj, gdf, popup_field_list):
    #Create empty lists to contain the point coordinates and the point pop-up information
    coords, popups = [], [] 
    #Loop through each record in the GeoDataFrame
    for i, row in gdf.iterrows():
        #Append lat and long coordinates to "coords" list
        coords.append([row.geometry.y, row.geometry.x])
        #Create a string of HTML code used in the IFrame popup
        #Join together the fields in "popup_field_list" with a linebreak between them
        label = '<br>'.join([row[field] for field in popup_field_list])
        #Append an IFrame that uses the HTML string to the "popups" list 
        popups.append(IFrame(label, width = 300, height = 100))
        
    #Create a Folium feature group for this layer, since we will be displaying multiple layers
    pt_lyr = folium.FeatureGroup(name = 'pt_lyr')
    
    #Add the clustered points of crime locations and popups to this layer
    pt_lyr.add_children(MarkerCluster(locations = coords, popups = popups))
    
    #Add this point layer to the map object
    mapobj.add_children(pt_lyr)
    return mapobj

#Update choropleth with point clusters
crime_map = add_point_clusters(crime_map, assaults, ['Descript','Address','DateStr','Time'])
```

### Add Layer Control, Display and Save Map
As a finishing touch, I add Layer Control to the map, which allows me to toggle on/off either of my two layers (see widget on the top right). Then I save my finished map as an HTML and display it! 

I hope this was helpful in demonstrating some of the mapping capabilities of Leaflet accessed through the package Folium.  The functions I wrote provide a nice way of displaying two very common types of spatial data on a basemap and can obviously be tweaked to add more custom functionality.


```python
folium.LayerControl().add_to(crime_map) #Add layer control to toggle on/off
crime_map.save('sf_assaults.html') #save HTML
crime_map #display map
```





