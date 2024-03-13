---
title: "China Map With Google and Autonavi"
date: 2024-03-11T14:55:29+08:00
author: "Feng Xue"
tags: ["Frontend", "Map", "Chinese Map", "GCJ-02", "Typescript"]
toc: true
usePageBundles: false
draft: false
---

These days I am investigating the map used in Hongkong and China mainland since it uses the different standard according to Chinese law. There are some several interesting points can be discussed here:

## Chinese Coordinate System GCJ-02 vs GPS WGS-84

Today most of the world uses the World Geodetic System WGS-84, where 84 is the latest vision of the standard applied in 1984. It's definitely created by US and used by GPS, and apparently google uses this standard for its map.

However, Chinese regulators added an obfuscation algorithm to the WGS-84, which would add an offset to the coordinate to protect its geographic data. Since the standard is created by (Chinese: 国测局; pinyin: guó-cè-jú) in 2002, so it's called GCJ-02, or Mars coordinates. And according to Chinese law, there is no official API to convert GCJ-02 to WGS-84.

So conversation between GCJ-02 and WGS-84 is illegal to be published, it has to be approved by [SBSM](https://en.wikipedia.org/wiki/State_Bureau_of_Surveying_and_Mapping). However, we still can find some open-source implementation, [here](https://github.com/wandergis/coordtransform/blob/master/index.js) is the js one. I won't say it's 100% accurate, but enough for the everyday usage.

To simplify the work, we would not use Baidu map since it has its own offset algorithm to distinguish its geodata, instead Autonavi/amap/高德 is a good alternative. It provides every good geo data and layer, also is my first choice when I am in China. Currently, instead of using its official web API, we only need its layer data, which can be achieved by [its url](https://github.com/muyao1987/leaflet-tileLayer-baidugaode) directly with [leaflet](https://leafletjs.com/), such as for map type `street map`/`roadmap`:

```js
{
  roadmap: {
    url: '//webrd0{s}.is.autonavi.com/appmaptile?lang=zh_cn&size=1&scale=1&style=8&x={x}&y={y}&z={z}',
    attribution:
      "&copy; <a href='http://osm.org/copyright'>OpenStreetMap</a> contributors",
    subdomains: ['1', '2', '3', '4'],
  },
  satellite: {
    url: '//webst0{s}.is.autonavi.com/appmaptile?style=6&x={x}&y={y}&z={z}',
    attribution:
      "&copy; <a href='http://osm.org/copyright'>OpenStreetMap</a> contributors",
    subdomains: ['1', '2', '3', '4'],
  },
}
```

It looks like this:

<div style="text-align:center;">
  <img src="/images/chinese-map/autonavi-roadmap-map.png" alt="Autonavi roadmap within leaflet" width="600"/>
</div>

## HongKong and Macau in Google Map

What about Hongkong and Macau? As China applies the [One country, two systems](https://en.wikipedia.org/wiki/One_country,_two_systems) in Hongkong and Macau, so google map is available in these two places. But, the standards applied by Google and Chinese map provider are different. Gaode states that [GCJ-02 would be applied in Hongkong and Macau](https://lbs.amap.com/faq/advisory/others/39840), while WGS-84 is used by google map. It means when we switch the map provider in Hongkong/Macau, we need to convert the standard.

## GCJ-02 in Google Map

There is one exception for Google map, according to [Wikipedia](https://en.wikipedia.org/wiki/Restrictions_on_geographic_data_in_China):

> Google has worked with Chinese location-based service provider AutoNavi since 2006 to source its maps in China.[44] **Google uses GCJ-02** data for the **street map**, but **does not shift the satellite** imagery layer, which continues to use **WGS-84 coordinates**,[45] with the benefit that WGS-84 positions can still be overlaid correctly on the satellite image (but not the street map). **Google Earth also uses WGS-84 to display the satellite imagery**.

The consequence would be no conversation is required if the coordinate(s) are created based on the google's street map (roadmap), such as `Point of interest` or `Geofences`. And also the map type switching also needs to consider the conversion.

And for the hybrid type map, it would be combined by the street map with GCJ and satellite with WGS:

<div style="text-align:center;">
  <img src="/images/chinese-map/China-google-hybrid.png" alt="Google hybrid map in China Mainland" width="600"/>
</div>

## How to check if a coordinate is in an closed region

Now we have one coordinate already, how could we check if it's inside of Mainland, Hongkong or Macau? One way would be calling map provider's API, the answer would be accurate, but the down side is it's asynchronous, which would increase the complexity of the codes. Is it possible to check the coordinate synchronously?

Yes, we have. If we can get the coordinates of these regions' borders with the closed polygon format in advance, we can check if the given coordinate is inside the given polygon.

### GeoJson data

We can find these places' geojson format data in GitHub: [China](https://github.com/xfsnowind/geojson-map-china/blob/master/china.json), [Hongkong](https://github.com/xfsnowind/HK-geojson/blob/master/polygon.json) and [Macau](https://github.com/xfsnowind/macau-map-data/blob/master/macau.json). To check the shape of these regions, we can copy/paste this json file to the [Geojson](https://geojson.io) and check the result.

### Pre-handling

Before we calculate the geojson data, we need to pre handle the data. Since WGS is also applied in Taiwan, so we need to remove this region from our China geojson data. Find the word `台湾省` in the file and remove the whole province data from the file.

### Extract the outermost border coordinates

Now we have all geojson data for these regions, but what we need is only the coordinates of the border. All the internal coordinates should be removed. To achieve this, we can use a python script (with help of [Microsoft copilot](https://copilot.microsoft.com)) to convert them:

```python
import json
from shapely.geometry import Polygon, mapping
from shapely.ops import unary_union

# Load GeoJSON file
with open('GEOJSON_FILE', 'r') as f:
    data = json.load(f)

# Extract Polygons and convert them to Shapely objects
polygons = []
for feature in data['features']:
    geometry = feature['geometry']
    if geometry['type'] == 'Polygon':
        polygons.append(Polygon(geometry['coordinates'][0]))

# Compute the union of all polygons
union = unary_union(polygons)

# Prepare the GeoJSON data
geojson_data = {
    "type": "Feature",
    "properties": {},
    "geometry": mapping(union)
}

# Save the GeoJSON data to a file
with open('output.geojson', 'w') as f:
    json.dump(geojson_data, f)
```

Replace `GEOJSON_FILE` with your real geojson data file. The result should be in the generated file `output.geojson`. To verify the generated data, we can copy/paste it to the [Geojson](https://geojson.io). Sometimes the generated polygon is not perfect, we can clean it simply by hand.

### Checking if a coordinate is inside of a polygon

Install the two @turf subset packages with [pnpm](https://pnpm.io/), (you can use your own package manager):

```shell
pnpm i @turf/boolean-point-in-polygon @turf/helpers
```

Then the method `isPointInPolygon` can be used to check if the coordinate is inside of the given polygon.

```typescript
import * as turf from '@turf/helpers'
import booleanPointInPolygon from '@turf/boolean-point-in-polygon'

type LatLng = {
  lat: number
  lng: number
}

export function isPointInPolygon(
  { lat, lng }: LatLng,
  polygon: Array<[lng: number, lat: number]>,
) {
  const point = turf.point([lng, lat])

  // Define your polygon
  const polygonTurf = turf.polygon([polygon])

  // Check if the point is inside the polygon
  return booleanPointInPolygon(point, polygonTurf)
}
```

**NOTE**: There is one thing we need to be careful. According to the [turf](https://github.com/Turfjs/turf?tab=readme-ov-file#data-in-turf):

<div style="text-align:center;">
  <img src="/images/chinese-map/turf-wgs.png" alt="Use Turf" width="600"/>
</div>

The coordinate sent to turf has to be `WGS` standard.

## China Mainland is swallowed by HongKong in the border

There is one more thing we need to consider when using google map in China mainland. Because of two coordinate systems in Mainland and Hongkong/Macau, the border between them is twisted:

The border in Google map:
<div style="text-align:center;">
  <img src="/images/chinese-map/China-hk-google-border.png" alt="China Mainland Border is swallowed by Hongkong" width="600"/>
</div>

The border in Autonavi:
<div style="text-align:center;">
  <img src="/images/chinese-map/China-hk-amap-border.png" alt="The correct mainland and hongkong border" width="600"/>
</div>

You can see the Mainland border is swallowed by HongKong, which would cause the overlapping of the maps and confusion of the point or path lines above it.
