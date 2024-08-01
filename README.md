# Maryland GeoJSON Visualization

This project visualizes various GeoJSON data layers over a map of Maryland using the Folium library. The map includes different layers such as county boundaries, census tracts, and school locations, with custom icons and popups. The map also features a collapsible legend and a link to the GitHub README page.

## Features

- **MDOT SHA County Boundaries**: Displayed in a fixed color format.
- **MD HB 550 Census Tracts**: Displayed with custom labels.
- **ENOUGH Act Child Poverty Census Tracts**: Highlighted in bright orange.
- **School Locations**: Schools are marked with different icons and colors based on their eligibility status for the ENOUGH Act.
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

# Function to add point layer from GeoJSON with custom icon based on CHILD_CPG_STATUS
def add_school_locations_from_geojson(geojson_url, map_obj):
    feature_group = folium.FeatureGroup(name="School Locations")
    
    response = requests.get(geojson_url)
    geojson_data = response.json()
    
    for feature in geojson_data['features']:
        properties = feature['properties']
        coordinates = feature['geometry']['coordinates']
        status = properties['CHILD_CPG_STATUS']
        school_name = properties['SCHOOL_NAME']
        school_district_name = properties['SCHOOL_DISTRICT_NAME']
        
        if status == "Eligible":
            icon = folium.Icon(color='purple', icon='star')
        else:
            icon = folium.Icon(color='green', icon='graduation-cap')
        
        popup_content = f"School Name: {school_name} District: {school_district_name} Status: {status}"
        
        folium.Marker(
            location=[coordinates[1], coordinates[0]],
            icon=icon,
            popup=folium.Popup(popup_content, parse_html=True)
        ).add_to(feature_group)
    
    feature_group.add_to(map_obj)
    
    # Add JavaScript to control visibility based on zoom level
    map_obj.get_root().html.add_child(folium.Element("""
    <script>
        function updateVisibility() {
            var map = window.m;
            var zoom = map.getZoom();
            var layers = map._layers;
            for (var layer_id in layers) {
                var layer = layers[layer_id];
                if (layer.options && layer.options.pane == 'markerPane') {
                    if (zoom >= 13) {
                        layer.setStyle({opacity: 1.0, fillOpacity: 1.0});
                    } else {
                        layer.setStyle({opacity: 0.0, fillOpacity: 0.0});
                    }
                }
            }
        }
        m.on('zoomend', updateVisibility);
        updateVisibility();
    </script>
    """))

# Add the school locations layer to the map
add_school_locations_from_geojson('https://raw.githubusercontent.com/MEADecarb/CTINT/main/Enough_Act_School_Locations.geojson', m)

# Function to add point layer from CSV with custom icon
def add_point_layer_from_csv(url, map_obj, icon_path):
    data = pd.read_csv(url)
    print("CSV Columns:", data.columns)
    
    for index, row in data.iterrows():
        if pd.notna(row['Lat']) and pd.notna(row['Long']):
            folium.Marker(
                location=[row['Lat'], row['Long']],
                icon=folium.CustomIcon(icon_path, icon_size=(30, 30)),
                popup=row['MDOT Location']
            ).add_to(map_obj)

# Path to your custom icon
icon_path = '/content/image.png'

# Add the point layer to the map
add_point_layer_from_csv('https://raw.githubusercontent.com/MEADecarb/geos/main/data/MDOTSolar.csv', m, icon_path)

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
     <i class="fa fa-star" style="color:pink"></i> ENOUGH Act Eligible School Location<br>
     <i class="fa fa-graduation-cap" style="color:green"></i> ENOUGH Act Non-Eligible School Location
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
                 bottom: 10px
