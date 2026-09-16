# os-zoomstack-pmtiles

These are instructions for converting the Ordnance Survey (OS) [Open Zoomstack data](https://osdatahub.os.uk/data/downloads/open/OpenZoomstack) vector tiles into a single [PMTiles](https://docs.protomaps.com/pmtiles) archive, optimized for self-hosting with [MapLibre](https://maplibre.org/maplibre-gl-js/docs).

## Introduction

OS Open Zoomstack is a _"comprehensive basemap of Great Britain, showing coverage from national level right down to street detail"_ - it's ideal for web mapping and is free to use under the [Open Government Licence v3.0](http://www.nationalarchives.gov.uk/doc/open-government-licence/).

- `✅` **Full Control**\
  Host on your own terms, and without paying API compute costs.\
  _You can also [serve a PMTiles archive from behind an API server](https://docs.protomaps.com/deploy/): this unlocks greater access control, usage metrics, and caching. That approach is out of scope for this guide which focuses on static file hosting._

- `✅` **Predictable Pricing**\
  OS Open Zoomstack is free. Aside from the (typically minimal) costs of static file storage, there are no managed API or licensing fees to consider.

- `ℹ️` **Non-Premium Data**\
  OS Open Zoomstack contains simplified geometries - suitable for most use cases. If you require high-accuracy coverage at lower zoom-levels, you may wish to use the OS NGD Vector Tile API.

- `ℹ️` **Non-Managed Service**\
  New releases of OS Open Zoomstack will require re-conversion to PMTiles and re-upload.

### Available Styles

I've adapted the six official layer styles - developed by [GeoDataVis experts at Ordnance Survey](https://github.com/OrdnanceSurvey/OS-Open-Zoomstack-Stylesheets/tree/master/Vector%20Tiles/Mapbox%20GL%20Styles) - to work with the PMTiles format. Additionally, Mapbox metadata has been removed and crucially **the JSON is simplified to only contain the layers array instead of a full MapLibre Style Spec**.

- `☀️` **Light**\
  `/extracted-layers/os-zoomstack-layers-light.json` is a clean, minimal, and unobtrusive basemap for data visualization purposes.

- `🌙` **Night**\
  `/extracted-layers/os-zoomstack-layers-night.json` is a dark-mode alternative of the light style, with reduced brightness and contrast.

- `🚗` **Road**\
  `/extracted-layers/os-zoomstack-layers-road.json` is reminiscent of a typical road atlas, it prioritises features that relate to the highway network.

- `🏕️` **Outdoor**\
  `/extracted-layers/os-zoomstack-layers-outdoor.json` is a topographic styling emphasizing terrain, maintained greenspace, and natural features.

- `🎨` **Deuteranopia**\
  `/extracted-layers/os-zoomstack-layers-deuteranopia.json` is a colorblind-accessible variant of the light style, optimized for blue-yellow color blindness.

- `🎨` **Tritanopia**\
  `/extracted-layers/os-zoomstack-layers-tritanopia.json` is a colorblind-accessible variant of the light style, optimized for blue-yellow color blindness.

## Getting Started

Download the following resources:

- The `/assets` and `/extracted-layers` directories from this repository.
- The OS Open Zoomstack dataset (in MBTiles format) from the [OS Data Hub](https://osdatahub.os.uk/data/downloads/open/OpenZoomstack).

### 1. Convert to PMTiles

Use the [pmtiles CLI tool](https://docs.protomaps.com/pmtiles/create#mbtiles) to convert the MBTiles (Mapbox Tiles) dataset into the PMTiles format:

```bash
pmtiles convert input.mbtiles vectors.pmtiles
```

### 2. Upload to Static File Storage

The PMTiles file is designed to be accessed directly over the internet using a technology called [HTTP Range Requests](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Range_requests). Upload the newly created `vectors.pmtiles` file and the `/assets` and `/extracted-layers` directories to a static file service. This could be an S3-compatible store with public access, a CDN, or a self-hosted web server.

You'll need to configure the CORS headers to permit access from your origin:

- HTTP `GET` requests with `Range` headers for the PMTiles archive.
- HTTP `GET` requests for the `/assets` and `/extracted-layers` directories.

The OS Open Zoomstack data is roughly ~2.6 GB; exposing an unrestricted PMTiles archive may incur egress costs if frequently downloaded in full. Consider restricting full downloads through your storage provider.

### 3. Configure MapLibre

Now the PMTiles archive and associated assets are accessible to end users, we can configure MapLibre to access them.

#### PMTiles Plugin

Follow the [Protomaps documentation](https://docs.protomaps.com/pmtiles/maplibre#installation) to load the PMTiles plugin and register the `pmtiles://` protocol with MapLibre. For React apps, it's best to handle this process in your `main.ts` or `index.ts` entrypoint.

```js
import { Protocol } from "pmtiles";
import { addProtocol } from "maplibre-gl";

const protocol = new Protocol();
addProtocol("pmtiles", protocol.tile);
```

#### Style Configuration

Contrary to most examples published by Ordnance Survey, I prefer to specify the base structure for the MapLibre Style Specification directly in the app. This approach has several advantages: you can set the URL for the PMTiles archive and assets dynamically in your app, and you can side-load any other styles and data sources you need.

The intention of this guide is for your app to fetch the a layer style JSON document from your static file service, marshall it into a JavaScript object, and then reference it in the `style.layers[]` array. This approach colocates the OS Open Zoomstack assets to a single static file service location.

```js
const osOpenZoomstackLightTheme = await fetchBasemapStyle("light");

/**
 * Replace FILE_SERVER_URL with the location
 * where the PMTiles archive / assets are stored.
 */

const style = {
  version: 8,
  name: "My Awesome Map with OS Open Zoomstack Data",

  sources: {
    // The source key MUST be os-zoomstack
    "os-zoomstack": {
      type: "vector",
      url: "pmtiles://https://FILE_SERVER_URL/vectors.pmtiles",
    },
    // Additional sources can be appended here...
  },

  sprite: "https://FILE_SERVER_URL/assets/sprites/sprites",
  glyphs: "https://FILE_SERVER_URL/assets/fonts/{fontstack}/{range}.pbf",

  layers: [
    ...osOpenZoomstackLightTheme,
    // Additional layer styles can be appended here...
  ],
};
```

#### Style Configuration - Local Hosting

You _could_ host the layer styles and assets in your app, thus only sending requests for the PMTiles archive itself.

If you choose this approach, consider lazy-loading requests to the JSON documents to reduce initial bundle size and network requests.

## Feedback

Suggestions are always welcome!\
Please note this repository is not affiliated with Ordnance Survey.
