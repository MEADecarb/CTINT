# Maryland GeoJSON Visualization

This project visualizes various GeoJSON and CSV data layers over a map of Maryland using the Folium library. The map includes different layers such as county boundaries, census tracts, ENOUGH Act school locations, other school locations, and MDOT Solar locations. The map also features custom icons, popups, a collapsible legend, and a link to the GitHub README page.

## Features

- **MDOT SHA County Boundaries**: Displayed in a fixed color format.
- **MD HB 550 Census Tracts**: Displayed with custom labels.
- **ENOUGH Act Child Poverty Census Tracts**: Highlighted in bright orange.
- **ENOUGH Act School Locations**: Displayed as black points on the map.
- **Other School Locations**: Displayed as blue stars and defaulted to being turned off.
- **MDOT Solar Locations**: Displayed as green leaf icons and defaulted to being turned off.
- **Collapsible Legend**: The legend can be toggled to show or hide details about the different layers.
- **Geocoder Plugin**: Allows searching for addresses on the map.
- **Link to GitHub README**: Provides easy access to the project documentation.

## Installation

1. **Clone the repository**:
    ```sh
    git clone https://github.com/MEADecarb/CTINT.git
    ```

2. **Install the required libraries**:
    ```sh
    pip install folium requests pandas branca
    ```

## Usage

Run the Python script to generate the map:

```python
import folium
from folium.plugins import Geocoder
import requests
import pandas as pd
import branca

# Define a color palette
color_palette = ["#2C557E", "#fdda25", "#B7DCDF", "#000000"]

# Create a base map centered over Maryland
m = folium.Map(location=[39.0458, -76.6413], zoom_start=8)

# Function to add GeoJSON from a URL to a feature group with custom color and pop-up
def add_geojson_from_url(geojson_url, name, color, map_obj):
    feature_group = folium.FeatureGroup(name=name)
    style_function = lambda x: {'fillColor': color, 'color': color}
    response = requests.get(geojson_url)
    geojson_data = response.json()

    geojson_layer = folium.GeoJson(
        geojson_data,
        style_function=style_function
    )

    if name == "MDOT SHA County Boundaries":
        geojson_layer.add_child(folium.GeoJsonPopup(fields=['COUNTY_NAME'], aliases=['County:'], labels=True))
    elif name == "MD HB 550 Census Tracts":
        all_fields = list(geojson_data['features'][0]['properties'].keys())
        geojson_layer.add_child(folium.GeoJsonPopup(fields=all_fields, labels=True))
    elif name == "Enough Act Child Poverty Census Tracts":
        geojson_layer = folium.GeoJson(
            geojson_data,
            style_function=lambda x: {'fillColor': '#FF8C00', 'color': '#FF8C00'}
        )
        geojson_layer.add_child(folium.GeoJsonPopup(fields=list(geojson_data['features'][0]['properties'].keys()), labels=True))

    geojson_layer.add_to(feature_group)
    feature_group.add_to(map_obj)

# Add each GeoJSON source as a separate feature group with a color, label, and pop-up
github_geojson_sources = [
    ("https://services.arcgis.com/njFNhDsUCentVYJW/arcgis/rest/services/MDOT_SHA_County_Boundaries/FeatureServer/0/query?outFields=*&where=1%3D1&f=geojson", "MDOT SHA County Boundaries"),
    ("https://raw.githubusercontent.com/MEADecarb/schools/main/MDHB550CensusTracts.geojson", "MD HB 550 Census Tracts"),
    ("https://raw.githubusercontent.com/MEADecarb/CTINT/main/enough_CT.geojson", "Enough Act Child Poverty Census Tracts")
]

for i, (url, name) in enumerate(github_geojson_sources):
    if name == "Enough Act Child Poverty Census Tracts":
        add_geojson_from_url(url, name, '#FF8C00', m)
    else:
        color = color_palette[i % len(color_palette)]
        add_geojson_from_url(url, name, color, m)

# Function to add ENOUGH Act school locations as black points
def add_enough_act_school_locations(geojson_url, map_obj):
    feature_group = folium.FeatureGroup(name="ENOUGH Act School Locations", show=True)
    
    response = requests.get(geojson_url)
    geojson_data = response.json()
    
    for feature in geojson_data['features']:
        properties = feature['properties']
        coordinates = feature['geometry']['coordinates']
        
        popup_content = f"School Name: {properties['SCHOOL_NAME']} District: {properties['SCHOOL_DISTRICT_NAME']} Status: {properties['CHILD_CPG_STATUS']}"
        
        folium.CircleMarker(
            location=[coordinates[1], coordinates[0]],
            color='black',
            radius=5,
            popup=folium.Popup(popup_content, parse_html=True)
        ).add_to(feature_group)
    
    feature_group.add_to(map_obj)

# Add the ENOUGH Act school locations layer to the map
add_enough_act_school_locations('https://raw.githubusercontent.com/MEADecarb/CTINT/main/Enough_Act_School_Locations.geojson', m)

# Function to add other school locations from CSV as blue stars
def add_other_school_locations_from_csv(url, map_obj, lat_col='LAT', long_col='LON', popup_cols=None):
    feature_group = folium.FeatureGroup(name="Other School Locations", show=False)
    data = pd.read_csv(url)
    
    for index, row in data.iterrows():
        if pd.notna(row[lat_col]) and pd.notna(row[long_col]):
            popup_content = ' '.join([f"{col}: {row[col]}" for col in popup_cols]) if popup_cols else ''
            folium.Marker(
                location=[row[lat_col], row[long_col]],
                icon=folium.Icon(color='blue', icon='star'),
                popup=folium.Popup(popup_content, parse_html=True)
            ).add_to(feature_group)
    
    feature_group.add_to(map_obj)

# Add the other school locations layer from CSV to the map
add_other_school_locations_from_csv('https://raw.githubusercontent.com/MEADecarb/CTINT/main/schools24%20-%20Sheet2.csv', m, lat_col='LAT', long_col='LON', popup_cols=['SCHOOL', 'PROJECT'])

# Function to add MDOT Solar locations from CSV as custom icons
def add_mdot_solar_locations_from_csv(url, map_obj, lat_col='Lat', long_col='Long', popup_cols=None):
    feature_group = folium.FeatureGroup(name="MDOT Solar Locations", show=False)
    data = pd.read_csv(url)
    
    for index, row in data.iterrows():
        if pd.notna(row[lat_col]) and pd.notna(row[long_col]]:
            popup_content = ' '.join([f"{col}: {row[col]}" for col in popup_cols]) if popup_cols else ''
            folium.Marker(
                location=[row[lat_col], row[long_col]],
                icon=folium.Icon(color='green', icon='leaf'),
                popup=folium.Popup(popup_content, parse_html=True)
            ).add_to(feature_group)
    
    feature_group.add_to(map_obj)

# Add the MDOT Solar locations layer from CSV to the map
add_mdot_solar_locations_from_csv('https://raw.githubusercontent.com/MEADecarb/CTINT/main/MDOTSolar.csv', m, lat_col='Lat', long_col='Long', popup_cols=['MDOT Location', 'Address'])

# Add layer control to the map
folium.LayerControl().add_to(m)

# Initialize the geocoder plugin
geocoder = Geocoder(
    collapse=True,
    position='topleft',
    add_marker=True,
    popup_on_found=True,
    zoom=12,
    search_label='address'
)

geocoder.add_to(m)

# Add a custom legend
legend_html = '''
     <div id="legend" style="position: fixed; 
                 bottom: 50px; left: 50px; width: 300px; height: auto; 
                 background-color: white; z-index:9999; font-size:14px;
                 border:2px solid grey; border-radius: 5px; padding: 10px;">
     <button onclick="toggleLegend()">Toggle Legend</button><br>
     <div id="legend-content">
     <b>Legend</b><br>
     <i style="background:#2C557E;color:#2C557E;">&nbsp;&nbsp;&nbsp;&nbsp;</i> County Boundaries<br>
     <i style="background:#fdda25;color:#fdda25;">&nbsp;&nbsp;&nbsp;&nbsp;</i> MD HB 550 Census Tracts<br>
     <i style="background:#FF8C00;color:#FF8C00;">&nbsp;&nbsp;&nbsp;&nbsp;</i> ENOUGH Act Child Poverty Census Tracts<br>
     <i style="background:black;color:black;">&nbsp;&nbsp;&nbsp;&nbsp;</i> ENOUGH Act School Location<br>
     <i class="fa fa-star" style="color:blue"></i> Other School Location<br>
     <i class="fa fa-leaf" style="color:green"></i> MDOT Solar Location
     </div>
     </div>
     <script>
     function toggleLegend() {
         var legendContent = document.getElementById('legend-content');
         if (legendContent.style.display === 'none') {
             legendContent.style.display = 'block';
         } else {
             legendContent.style.display = 'none';
         }
     }
     </script>
     '''

m.get_root().html.add_child(folium.Element(legend_html))

# Add a textbox with a link to the GitHub README page
readme_html = '''
     <div style="position: fixed; 
                 bottom: 10px; left: 50px; width: 300px; height: auto; 
                 background-color: white; z-index:9999; font-size:14px;
                 border:2px solid grey; border-radius: 5px; padding: 10px;">
     <a href="https://github.com/MEADecarb/CTINT/blob/main/README.md" target="_blank">GitHub README</a>
     </div>
     '''

m.get_root().html.add_child(folium.Element(readme_html))

# Save the map to an HTML file
m.save('index.html')

# Optional: Display the map in a Jupyter Notebook (only if you are running this in a Jupyter environment)
m
