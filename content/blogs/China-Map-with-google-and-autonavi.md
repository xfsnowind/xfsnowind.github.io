---
title: "China Map With Google and Autonavi"
date: 2024-03-11T14:55:29+08:00
author: "Feng Xue"
draft: true
---

These days I am investigating the map used in Hongkong and China mainland since it uses the different standard according to Chinese law. There are some several interesting points can be discussed here:


- [Chinese Coordinate System GCJ-02 vs GPS WGS-84](#chinese-coordinate-system-gcj-02-vs-gps-wgs-84)
- [HongKong and Macau](#hongkong-and-macau)
- [GCJ-02 in Google Map](#gcj-02-in-google-map)
- [How to check if a coordinate is in an closed region](#how-to-check-if-a-coordinate-is-in-an-closed-region)
- [China Mainland is swallowed by HongKong in the border](#china-mainland-is-swallowed-by-hongkong-in-the-border)

## Chinese Coordinate System GCJ-02 vs GPS WGS-84

Today most of the world uses the World Geodetic System WGS-84, where 84 is the lastest vision of the standard applied in 1984. It's definitely created by US and used by GPS, and apparently google uses this standard for its map. 

However, Chinese regulators added an obfuscation algorithm to the WGS-84, which would add an offset to the coordinate to protect its geographic data. Since the standard is created by (Chinese: 国测局; pinyin: guó-cè-jú) in 2002, so it's called GCJ-02, or Mars coordinates. And according to Chinese law, there is no official API to convert GCJ-02 to WGS-84.

So conversation between GCJ-02 and WGS-84 is illegal to be published, it has to be approved by [SBSM](https://en.wikipedia.org/wiki/State_Bureau_of_Surveying_and_Mapping). However, we still can find some open-source implmentation, [here](https://github.com/wandergis/coordtransform/blob/master/index.js) is the js one. I won't say it's 100% accurate, but enough for the everyday usage.

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

![Autonavi-within-leaflet](/images/chinese-map/autonavi-roadmap-map.png "Autonavi roadmap within leaflet")


## HongKong and Macau

What about Hongkong and Macau? As China applies the [One country, two systems](https://en.wikipedia.org/wiki/One_country,_two_systems) in Hongkong and Macau, so google map is available in these two places. But, the standards applied by Google and Chinese map provider are different. Gaode states that [GCJ-02 would be applied in Hongkong and Macau](https://lbs.amap.com/faq/advisory/others/39840), while WGS-84 is used by google map. It means when we switch the map provider in Hongkong/Macau, we need to convert the standard.

## GCJ-02 in Google Map

There is one exception for Google map, according the [Wikipedia](https://en.wikipedia.org/wiki/Restrictions_on_geographic_data_in_China):

> Google has worked with Chinese location-based service provider AutoNavi since 2006 to source its maps in China.[44] Google uses GCJ-02 data for the street map, but does not shift the satellite imagery layer, which continues to use WGS-84 coordinates,[45] with the benefit that WGS-84 positions can still be overlaid correctly on the satellite image (but not the street map). Google Earth also uses WGS-84 to display the satellite imagery.

The consequence would be no conversation is required if the coordinate(s) are created based on the google's street map (roadmap), such as `Point of interest` or `Geofences`. And also the map type switching also needs to consider the conversion.

## How to check if a coordinate is in an closed region

Now we have one coordinate already, how could we check if it's inside of Mainland, Hongkong or Macau? One way would be calling map provider's API, the answer would be accurate, but the down side is it's asynchronized, which would increase the complexity of the codes. Is it possible to check the coordinate synchronously?

Yes, we have. We can find these places' geojson format data in github: [China](https://github.com/longwosion/geojson-map-china/tree/master), [Hongkong]() and [Macau]()

## China Mainland is swallowed by HongKong in the border

![China-hk-border-google](/images/chinese-map/China-hk-google-border.png "China Mainland Border is swallowed by Hongkong")

![China-hk-border-amap](/images/chinese-map/China-hk-amap-border.png "The correct mainland and hongkong border")