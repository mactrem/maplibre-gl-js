# Architecture of MapLibre GL JS

## `Map` and its subsystems

## Main thread / worker split

## How (vector tile) rendering works

### Parsing and layout

Vector tiles are fetched and parsed on WebWorker threads.  "Parsing" a vector tile involves:
 - Deserializing source layers, feature properties, and feature geometries from the PBF.  This is handled by the [`vector-tile-js`](https://github.com/mapbox/vector-tile-js) library.
 - Transforming that data into _render-ready_ data that can be used by WebGL shaders to draw the map.  We refer to this process as "layout," and it carried out by `WorkerTile`, the `Bucket` classes, and `ProgramConfiguration`.
 - Indexing feature geometries into a `FeatureIndex`, used for spatial queries (e.g. `queryRenderedFeatures`).

`WorkerTile#parse()` takes a (deserialized) vector tile, fetches additional resources if they're needed (fonts, images), and then creates a `Bucket` for each 'family' of style layers that share the same underlying features and 'layout' properties (see `group_by_layout.js`).

[Bucket](./src/data/bucket.js) is the single point of knowledge about turning vector tiles into WebGL buffers. Each bucket holds the vertex and element array data needed to render its group of style layers (see [ArrayGroup](./src/data/bucket.js)).  The particular bucket types each know how to populate that data for their layer types.

### Rendering with WebGL

Once bucket data has been transferred to the main thread, it looks like this:

```
Tile
  |
  +- buckets[layer-id]: Bucket
  |    |
  |    + ArrayGroup {
  |        globalProperties: { zoom }
  |        layoutVertexArray,
  |        indexArray,
  |        indexArray2,
  |        layerData: {
  |          [style layer id]: {
  |            programConfiguration,
  |            paintVertexArray,
  |            paintPropertyStatistics
  |          }
  |          ...
  |        }
  |    }
  |
  +- buckets[...]: Bucket
        ...
```
_Note that a particular bucket may appear multiple times in `tile.buckets`--once for each layer in a given layout 'family'._

 - Rendering happens style-layer by style-layer, in `Painter#renderPass()`, which delegates to the layer-specific `drawXxxx()` methods in `src/render/draw_*.js`.
 - The `drawXxxx()` methods, in turn, render a layer tile by tile, by:
   - Obtaining a property configured shader program from the `Painter`
   - Setting _uniform_ values based on the style layer's properties
   - Binding layout buffer data (via `BufferGroup`) and calling `gl.drawElements()`

Compiling and caching GL shader programs is managed by the `Painter` and `ProgramConfiguration` classes.  In particular, an instance of `ProgramConfiguration` handles, for a given (tile, style layer) pair:
 - Expanding a `#pragma mapbox` statement in our shader source into either a _uniform_ or _attribute_, _varying_, and _local_ variable declaration, depending on whether or not the relevant style property is data-driven.
 - Creating and populating a _paint_ vertex array for data-driven properties, corresponding to the `attributes` declared in the shader. (This happens at layout time, on the worker side.)


## SourceCache
-> style.loadJson -> _load -> style.addSource -> create SourceCache -> create Source

painter.render
-> sourceCache.prepare
    -> tilel.upload / tile.prepare

-> Transform -> How tile loading and caching works
    -> where the covering tiles are calculated -> transform.coveringTiles
        -> map._render -> if sources dirty -> style._updateSources(this.transform)
            -> sourceCache.update(transform) -> Removes tiles that are outside the viewport and adds new tiles that are inside the viewport
    -> coveringTiles
        -> Return all coordinates that could cover this transform for a covering zoom level
        -> coveringZoomLevel -> return a zoom level that will cover all tiles the transform
        -> const cameraFrustum = Frustum.fromInvProjectionMatrix -> Frustum -> frustumCoords, frustumPlanes
        -> RootTile.aabb.intersects(cameraFrustum) -> Aabb.intersects -> Performs a frustum-aabb intersection test
            -> aabb -> axis-aligned bounding box -> view frustum culling
        -> What happens with tiles which are outside -> are the retained?
## Transform

The Transform (add link) class is responsible for transforming coordinates between different coordinate spaces.
The posMatrix/projection matrix transforms tile coordinates (0...8192) into Normalize Device Coordinates (-1, 1) used by WebGL for
displaying the tile on the screen.
All data sources types are tile based, so for example even for a GeoJson source a tiled map is calculated (via geojson-vt).
The posMatrix is calculated for every tile in the
The local tile coordinates avoid Z-fighting which can happen when large world coordinates (mercator)
would be direcly passed to the vertex shader. ? -> center to eye rendering

The posMatrix can be seen as the ModelViewProjection matrix in OpenGL terms.
The projectionMatrix as the viewProjectionMatrix in OpenGL terms.

For each individual tile which should be displayed on the screen a projection matrix is calculated.


The positonMatrix is set as uniform to the shader?
uniform in vertex shader (u_matrix) which transforms the tile coordiantes (a_position) into WebGL clip space

For calculating the projection matrix of a tile the following two methods in the Transform class are used:
- _calcMatrices
  - the matrices are updated in two ways
    - via Map -> set pitch, fov, zoom, setLocationAtPoint, ...
    - triggered via HandalerManager ->
  - _calMatrices is executed when set bearing, set pitch, set fov, set zoom, set center, ... is called
    -> or via the HandlerManager._updateTransform via events e.g. when the map is panned
  - On every change to the viewport (set bearing, set pitch, set fov, set zoom, set center, ...)
    _calcMatrices is called where different matrices are calculated (projMatrix, invProjMatrix, pixelMatrix, mercatorMatrix)
  - The projection matrix (projMatrix) is calculated for the center point -> mercator world pixel coordinate
      - the center of the map is transalted from the north-west corner
  - Type of matrices:
  - projectionMatrix
    - tow steps
      - from world to camera space
        - flip y axis
        - move (translate) z to the calculated cameraToCenterDistance
        - bearing -> rotate z-Axis
        - pitch -> rotate x-Axis
        - translate x,y to center point / center of the map
        - scale vertically pixel fro meters via multiplying wihte meter/pixel -> z-value (e.g buildings) in vector tiles in meters compared to x,y
      - from camera space to NDC
        - calculate projection matrix via ma4.perspective
        - fov -> default 36.87
        - cameraToCenter
        - topHalfSurfaceDistance
        - furtestDistance
        - farZ -> Add a bit extra to avoid precision problems when a fragment's distance is exactly `furthestDistance`
        - nearDistance
        - Diagram -> centerOffseet not taken into account
  - mercatorMatrix
    - The mercatorMatrix can be used to transform points from mercator coordinates ([0, 0] nw, [1, 1] se) to GL coordinates
    - Input parameter in the CustomLayer render method to convert mercator coordinates to NDC
  - alignedProjMatrix
    - matrix is aligned to a pixel grid for rendering raster tiles
    -  the (floating point) x/y values are rounded to avoid rendering raster images to fractional coordinates
  - glCoordMatrix
  - pixelMatrix
    - matrix for conversion from location to screen coordinates
  - labelPlaneMatrix
  - ViewProjection matrix in OpenGL naming -> transform pixel coordiantes with the origin
      in the north-west corner to clip space -> tile size is 512
      -> translation to the center
      -> rotation of the bearing and pitch
  - Add description for the alignedProjectionMatrix for raster tiles

- calculatePosMatrix
   - create the ModelViewProjectionMatrix by multiplying the ModelMatrix calculated in this method
     with the ViewProjection matrix from the _calcMatrices method
     -> translation to the World Coordiante center at the top left
     -> Scaling from EXTENT (8192) to 512 -> Buckets has to have coordiantes in the EXTENT space
     -> EXTENT -> The maximum value of a coordinate in the internal tile coordinate system. Coordinates of
               -> all source features normalized to this extent upon load
               -> the normalization for vector tiles happens in loadGeometry.js where every geometry of a feature gets transformed
               -> loadGeometry -> loads a geometry from a VectorTileFeature and scales it to the common extent used internally
               -> So the vertices for every tile given to the vertex shader has to be scaled on that EXTENT
               -> loadGeometry is called from the buckets -> LineBucket, FillBucket, ...
               -> The posMatrix (ModelViewProjection Matrix) for a tile expects coordinates in this coordinate system
   - In the painter class for every tile via OverscaledTileID
   - calculates the posMatrix for transforming the coordinates of a specific tile in the webgl clipsace
   - takes the projMatrix an translates to the specified center in pixel coordinates
   - translate center, rotate pitch, rotate angle(bearing?), perspective transformation (fov, nearZ, farZ) via mat4.perspective
   - Tile coordiantes are scaled to EXTENT -> World pixel coordinates -> center with 512 pixel per tile
   - center (lat/lon) world coordiantes -> world pixel coordinates -> expects 8192 range coordinates for the tile
      - mercator coordinate -> the origin of the coordinate space is at the north-west corner instead of the middle
   - the calculated posMatrix is attatched the OverscaledTileID class
    -> so the buckets of every source has to be in range EXTENT?
    -> Each tile has it's own coordinate space going from (0, 0) at the top left to (EXTENT, EXTENT) at the bottom right
    -> At the end of everything, the vertex shader needs to produce a position in GL coordinate space,
       which is (-1, 1) at the top left and (1, -1) in the bottom right
    - The uniform positonMatrix is set as u_matrix in the shader in the Program class in the draw method
    - Questions
        - 512 is used as tileSize -> matching with real tile size?
          -> In VectoTileSize ctor only tiles with this size are accepted -> what is with other sizes?
        - In projection Matrix the center of the map is origin of the crs
            -> But the translation takes North-West as origin
        - What impact does near and far z have?
-> Painter.translatePosMatrix
          - Transform a matrix to incorporate the *-translate and *-translate-anchor properties into it
          - e.g. fill-translate, fill-translate-anchor

In OpenGL terms:
- Model Matrix
    - transformation of a tile into world space
    - the origin of the world coordinate system is in the top left
    -
- View Matrix
- Projection Matrix
- ModelViewProjection Matrix

## Controls

