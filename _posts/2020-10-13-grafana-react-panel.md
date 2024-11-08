---
layout: post
title: Developing Grafana Panel plugin
subtitle: developing a basic custom Graph Panel plugin for Grafana
tags: [typescript, grafana]
---

**NOTE: This is an old post that I've transferred here, the original post is [here in medium](https://medium.com/@hariom.2711/grafana-react-panel-plugins-545cb9afa42d). The blog post date is the date when I originally posted it, although I moved it here in Oct. 2024**  


In a previous project of mine, I had to write some basic panel plugins for the Grafana ecosystem. Now in my searches I couldn't find a lot of good small beginner examples for the React Ecosystem of their plugins. I managed to go through with my job by looking at the component props and types in the library.
Therefore I thought I'll share my experience for someone who's looking for tutorials of this type.   
**Note:** You can view the final working code for this tutorial [here](https://github.com/narang99/grafana-plugin-tutorial)

---

Grafana React Plugins are written in TypeScript and you need to have basic TypeScript knowledge along with its API in the React ecosystem.
There are three types of plugins in the Grafana system.
1. Panel Plugins: These are the plugins you write to make a custom panel.
2. Datasource Plugins: This facilitates for you to provide your own custom datasource.
3. App Plugins: With these you can use Grafana API to make app level plugins.

This blog post discusses React panel plugins for Grafana.

# Setup

**Note:** If you know the basic setup and directory structure of Grafana plugins you can skip to the next section.

You can take a look at detailed structure of Grafana plugins [here](https://grafana.com/developers/plugin-tools/tutorials/build-a-panel-plugin#2)

You can create a folder for storing plugins anywhere in your system. You have to add the path of the folder in the Grafana configuration file. Grafana loads up all the plugins you have kept on startup.  
The configuration file should have:

```toml
[paths]
plugins = "path/to/GrafanaPlugins"
```

`@grafana-toolkit` is a CLI which will help you create basic structure of your plugin and also provide build and debug scripts. Think of it as a Grafana plugin equivalent of `create-react-app`

```bash
npm install -g @grafana/toolkit
```
Use `@grafana-toolkit` to create a skeleton for your plugin. Here the plugin name is GraphPlugin

```bash
npx @grafana/toolkit plugin:create graph-plugin
cd graph-plugin
npm install
```

You’ll see a skeleton directory structure. `module.tsx` (the skeleton structure may have `module.ts`, its easier to change the extension to `tsx`)is the entry-point of the Plugin. `plugin.json` has plugin related metadata. You can change the name attribute to provide a name to your plugin. `type` can be `panel`, `datasource`, `app`.


# Panel Plugin Basics

Your main panel should have props of type `PanelProps<options>`. You have to specify the type for options. `options` are basically used to enable the user to provide custom settings for your panel. The type you specify for options governs this behavior. For now we keep it empty.

```typescript
import React from 'react';
import { PanelProps } from '@grafana/data';

export interface GraphPanelOptions {

};

export const GraphPanel:React.FC<PanelProps<GraphPanelOptions>>
  = props => {
  return (
    <div></div>
  );
}
```

`props` will contain many attributes. Relevant ones for our GraphPanel are `data`, `width`, `height`, `timeRange`.
`data` attribute contains the whole of your time-series data. Each time-series contains two fields, the first one being your time points and the other one being the function you plot on y-axis. There will be multiple time-series which will come as different lines plotted on the same graph in your `data` attribute depending on your backend. `width`, and `height` contain the actual width and height of the space allocated to your panel when it is rendered on Grafana. `timeRange` is the time range selected by the user.

# Making a Graph Panel

The Graph Panel can be made by the components `Graph` or `GraphWithLegend`. We will use `Graph` for simplicity.
You need to pass the props correctly to Graph. Here is the definition of type of props defined in `@grafana-ui` for `Graph` component.


```typescript
export interface GraphProps {
  ariaLabel?: string;
  children?: JSX.Element | JSX.Element[];
  series: GraphSeriesXY[];
  timeRange: TimeRange; // NOTE: we should aim to make `time` a property of the axis, not force it for all graphs
  timeZone?: TimeZone; // NOTE: we should aim to make `time` a property of the axis, not force it for all graphs
  showLines?: boolean;
  showPoints?: boolean;
  showBars?: boolean;
  width: number;
  height: number;
  isStacked?: boolean;
  lineWidth?: number;
  onHorizontalRegionSelected?: (from: number, to: number) => void;
}
```

The required props are `width`, `height`, `timeRange`, `series`. You can pass `props.width`, `props.height`, `props.timeRange` to the first three respectively (These are good defaults). `series` prop holds the data you want to render in your graph. Each line that you see on a normal Grafana is of type `GraphSeriesXY.` Each panel can render multiple lines for different metrics/variables and therefore series is an array of `GraphSeriesXY.` We can construct `series` prop from `props.data.` All time-series data in Grafana is stored and used as an object of type `DataFrame.` It is kind of like data frames in data science if you are familiar with them (like pandas dataframes). `props.data.series` is of type `DataFrame[]`, ie. you have a collection of time-series data stored as an array. Each item in this array corresponds to a single element in your `GraphSeriesXY[]` array. We have to therefore map object of type `DataFrame` to object of type `GraphSeriesXY`. This can be done in a straightforward way as follows:


```typescript
let ser_ind = 0;
const series: GraphSeriesXY[] = props.data.series.map(item => {
const timeVals: GraphSeriesValue[] = item.fields[0].values.toArray();
const yVals: GraphSeriesValue[] = item.fields[1].values.toArray();

const data: GraphSeriesValue[][] = [];
for (let i = 0; i < timeVals.length; i++) {
  data.push([timeVals[i], yVals[i]]);
}

const unixTimeRange = props.timeRange.to.unix() - props.timeRange.from.unix();
const ser: GraphSeriesXY = {
  seriesIndex: ser_ind++,
  yAxis: { index: 0 },
  isVisible: true,
  timeField: props.data.series[0].fields[0],
  valueField: props.data.series[0].fields[1],
  timeStep: props.width / unixTimeRange,
  data: data,
  label: 'some label',
};
return ser;
});
```

We map through `props.data.series`. Each item is of type `DataFrame`. The time-series data is stored in the `fields` array of `item`. Each field inside the `fields` array consists of data points for a single axis. We have two axis in our graphs, `time` (`fields[0]`, x axis) and `values` (`fields[1]`, y axis). We construct `data` of type `GraphSeriesValue[][]`. We create `ser` using `data`, `fields` and `timeRange` in unix values. From this transformation we get a `series` array which can be passed to `Graph` component. Make sure to give a distinct index to each item in `series` (`ser_ind` here). You can take a look at type `GraphSeriesXY` if you want to play around with other props like `color`, etc.

# Graph Component

Finally we pass the `series` object to `Graph` component.

```typescript
return (
  <div>
     <Graph
        height={props.height}
        width={props.width}
        series={series}
        timeRange={props.timeRange}
        showLines={true}
      />
  </div>
);
```

# Final Code for GraphPanel
Here is everything in a single snippet

```typescript
import React from 'react';
import { PanelProps, GraphSeriesXY, GraphSeriesValue } from '@grafana/data';
import { Graph } from '@grafana/ui';

export interface GraphPanelOptions {
  graphType: string;
}

export const GraphPanel: React.FC<PanelProps<GraphPanelOptions>> = props => {
  let ser_ind = 0;
  const series: GraphSeriesXY[] = props.data.series.map(item => {
    const timeVals: GraphSeriesValue[] = item.fields[0].values.toArray();
    const yVals: GraphSeriesValue[] = item.fields[1].values.toArray();

    const data: GraphSeriesValue[][] = [];
    for (let i = 0; i < timeVals.length; i++) {
      data.push([timeVals[i], yVals[i]]);
    }

    const unixTimeRange = props.timeRange.to.unix() - props.timeRange.from.unix();
    const ser: GraphSeriesXY = {
      seriesIndex: ser_ind++,
      yAxis: { index: 0 },
      isVisible: true,
      timeField: props.data.series[0].fields[0],
      valueField: props.data.series[0].fields[1],
      timeStep: props.width / unixTimeRange,
      data: data,
      label: 'some label',
    };
    return ser;
  });

  const showLines = props.options.graphType === 'line';
  const showBars = props.options.graphType === 'bar';
  return (
    <div>
      <Graph
        height={props.height}
        width={props.width}
        series={series}
        timeRange={props.timeRange}
        showLines={showLines}
        showBars={showBars}
      />
    </div>
  );
};
```

# module.tsx

We have to export the `GraphPanel` component wrapped inside the `PanelPlugin` component from Grafana from `module.tsx` (the entry point of the plugin). A basic `module.tsx` would be:

```typescript
import { PanelPlugin } from '@grafana/data';
import { GraphPanel, GraphPanelOptions } from './GraphPanel';

export const plugin = new PanelPlugin<GraphPanelOptions>(GraphPanel);
```
Thats it. Doing this will create a basic GraphPanel which you can further customize according to your own needs.

# Options

The default Graph Panel provided by Grafana has rich features already and you would need to make your own very rarely for highly specific applications. Many times you would want to give the user custom options for fine-grained user control.  

I would suggest looking at this video for uses of options and their basics. The video uses older deprecated API.

<iframe src="https://www.youtube.com/embed/Y31wnP_jDBY?si=c9KQipioUE-a-NAt" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>


I will add a basic option to our `PanelPlugin`. We would add a `radio` button with two options to let the user select between bar graphs and line graphs. First you have to add an attribute inside `GraphPanelOptions` interface.

```typescript
export interface GraphPanelOptions {
  graphType: string;
}
```

The `options` will be accessible using `props.options`. Here it would have one property: `props.options.graphType`. It would be passed down to your component with the value selected by user.

Now we need to add a radio button for user to choose the type from. We do that in `module.tsx` in the following way.
```typescript
import { PanelPlugin } from '@grafana/data';
import { GraphPanel, GraphPanelOptions } from './GraphPanel';

export const plugin = new PanelPlugin<GraphPanelOptions>(GraphPanel).setPanelOptions(builder => {
  return builder
    .addRadio({
    path: 'graphType',
    defaultValue: 'line',
    name: 'Graph Type',
    settings: {
      options: [
          {
              value: "line",
              label: "Line"
          },
          {
              value: "bar",
              label: "Bar"
          }
      ],
    },
  })
});
```

One final step is to add support for these options in `GraphPanel.tsx`. If user has selected **Line** then `showLines` has to be set to `true` in props to `Graph` component. If **Bar** is selected `showBars` has to be set `true`. Following changes are made:

```typescript
const showLines = (props.options.graphType === 'line');
const showBars  = (props.options.graphType === 'bar');
return (
<div>
 <Graph
    height={props.height}
    width={props.width}
    series={series}
    timeRange={props.timeRange}
    showLines={showLines}
    showBars={showBars}
  />
</div>
);
```

Start the grafana server. Use `npm run dev` to build the plugin. Grafana will load plugins at startup but any build changes will be reflected immediately (you don't need to restart). You only need to restart the server when you make a brand new plugin.

# Final Product
Make a new dashboard. Add a panel. In edit panel, select GraphPlugin (or name of you plugin) in Visualization setting. Options can be changed from Display settings.


<div class="image-with-caption">
  <img src="/assets/grafana-graph-panel-plugin/bar-option.webp" alt="Bar option selected for GraphPanel plugin">
  <p>Bar option selected for GraphPanel plugin</p>
</div>

<div class="image-with-caption">
  <img src="/assets/grafana-graph-panel-plugin/line-option.webp" alt="Line option selected for GraphPanel plugin">
  <p>Line option selected for GraphPanel plugin</p>
</div>

# Conclusion

The react ecosystem for plugin development is relatively new for Grafana and thus lacks beginner support and tutorials. The best way to make a plugin is to use a TypeScript IDE and go through the type signatures of the components and their props.

This is the final working code supporting this article.  
[https://github.com/narang99/grafana-plugin-tutorial](https://github.com/narang99/grafana-plugin-tutorial)
