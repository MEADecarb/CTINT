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

