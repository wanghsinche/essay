# How to design a world map component
> Before we talk about this topic, I would like to list down the items that I am going to cover.
## Table of content
1. requirements
2. layout component
3. APIs
4. data flow
5. optimization 
6. accessibility
## Requirement
<img width="263" alt="image" src="https://user-images.githubusercontent.com/14119632/182647884-835d1077-cca5-42ed-9e53-f758544df773.png">

Let’s start with the functional requirements and non-functional requirements. 

The world map component is quite common in data visualization. It usually displays a whole world map with features like zoom in / zoom out, move viewport, etc. It also supports user interactions such as tooltips and highlights. 

There’re also some non-functional requirements for our component. 

For example, the rendering performance is crucial because the data magnitude is quite large for data visualization. And we should take care of the accessibility as well.

## Tech option decision
Before sketching out the high-level architecture, we should choose an appropriate rendering engine. 

There’re two common practices, the canvas solution, and the SVG solution. 

Canvas solution has lots of benefits. It can take advantage of webGL to have a better rendering performance. Furthermore, vivid animation effects are easy to create via canvas, and the world map can be beautiful in the end. However, this approach comes with some disadvantages. It implies that we must create everything from scratch, including the geometry elements, the event bubbling, etc. It also means that we lose the semantics and accessibility of the HTML. 

According to these shortcomings, I choose the SVG solution, which takes advantage of HTML and is easy to debug via chrome dev tools. Besides, the accessibility would be better. However, using the SVG solution means I must be careful about the performance, especially when we have thousands of regions to display. 

## High level architecture

![image](https://user-images.githubusercontent.com/14119632/182647939-a70634b1-4e48-4acf-8b37-259bf260e89e.png)

It consists of several layers, the transform/viewport layer, the geometry layer, and the marker layer. 

The transform/viewport layer is for determining the final visual result of the component. It takes the transform parameters and viewport to calculate the result of the visible area.  The geometry layer is used to render the region geometry and the marker layer is for the tooltips. 

## The API definition

```typescript
{
dataSource: {region: string; data: number;}
geoSource: IGeoJSON[]
zoomLevel: number
viewPort: [[x, y], [x, y]]

onZoom: (zoom Level, viewPort)=>void
onHover: (lon, lat)=>void
onMove: (zoom Level, viewPort)=>void

tooltipRender: (lon, lat, zoom Level)=>ReactElement

}
```

## The data format and data flow
![image](https://user-images.githubusercontent.com/14119632/182649331-018092ef-3ed3-4f67-91c9-1ca56b855710.png)

## Optimization
After finishing all the high-level designs, let’s dig deeper into some details. 

As mentioned above, performance is the crucial factor for a large magnitude data visualization component. 

To speed up the rendering and reduce the blocking time, we can approach it in 2 ways. 

Firstly, we can reduce the unnecessary re-rendering. For example, we separate the whole component into different layers and only re-render the corresponding layer when there’re some changes. We can also use the memorization technique to eliminate the duplicated re-rendering. After introducing these techniques, the frame per second increased from 40 to around 60. 
But there’s still some space to improve. We found it’s unnecessary to use a high-resolution map when at a low zoom level. Besides, the geometries that are out of the viewport should be hidden. Thus, we used the similar tech that derives from the virtual list to create our own ‘virtual geometry’, which means we only render the geometries that are in the viewport. Furthermore, we can use different resolutions for different zoom levels.

But the challenge is how we know the geometries that need to be displayed in our viewport. In other words, how do we find out the regions within the viewport by using longitude and latitude? 

After having several meetings with the BE engineers, we decided to involve a proximity service to solve this issue. 
It’s not complex to implement a simple proximity service by using Geo-Hash,  which is compatible with Redis. By encoding the longitude and latitude into Geo-Hash, we can easily conduct the nearby searching by matching the hash prefix. To avoid the edge case of Geo-Hash, we can search the center point of the viewport and its eight neighborhoods, which results in a larger scope than the viewport. But it is acceptable in this scenario. 

By using the proximity service, we keep a smooth user experience whatever the zoom level is. 

## Accessibility
1. semantic svg elements (tooltip tag, path, rectangle, etc)
2. arial-label for each region and its value
