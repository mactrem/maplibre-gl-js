## paint properties
-> FillStyleLayer -> extends StyleLayer
    -> StyleSpecification as Ctor argument
    -> paint properties for drawFill -> PossibleEvaluatedProperty
    -> paint.get("fill-style")
    -> createBucket
    -> ProgramConfiguration.setUnfiorms

## Components vector tiles rendering
Map:
_render Method
    -  Is called among other things when:
        - The style has changed (`setPaintProperty()`, etc.)
        - Source data has changed (e.g. tiles have finished loading)
        - The map has is moving (or just finished moving)

Buckets:
- The concrete bucket types, using layout options from the style layer, transform feature geometries into vertex and index data for use by the vertex shader
- Buckets are designed to be built on a worker thread
- On the worker side, a bucket's vertex, index, and attribute data is stored in `bucket.arrays: * ArrayGroup`
- Which coordinates are stored in the bucket?
- Members
    - IndexBuffer
    - VertexBuffer
    - Segments -> SegmentVector -> Segment.vaos
LineStyleLayer
- create a LineBucket

- Draw Layers
    - General
      - Roughly 650 individual draw calls per frame -> 39k drawcalls per second
      - Layers are orderd list of draw calls
      - The less layers you have in your style the less draw calls you have -> better performance -> using fewer layers via expressions
    - Render passes
        - currently render layers in three different passes — offscreen, opaque and translucent
        - opaque pass is “just” a performance optimization to allow early depth testing
    - Symbol
      - symbol collisions -> debug.html
    - Line
    - Fill
    - Fill-Extrusion
        - Using earcut for tesselation of the polygons
        - Extrude and add a vertical polygon for each side
        - Add normal vector to each vertex -> used to determine the final color based on the light property of the map
    - Circle
    - Background
      - dont come from data
    - Hillshade
    - Heatmap


Style.addLayer / Style._load -> createStyleLayer (create a Layer) -> LineStyleLayer
VectorTileWorkerSource.done/reloadTile -> wokerTile.parse -> createBucket


Styles:
- map.addLayer -> style.addLayer -> validate_style_min -> validate_layer -> validate_object -> validate.js
- setStyle
    - diff defaults to true -> style.setState performas diff
    - Map._diffStyle -> Map._updateDiff -> Style.setState -> Style.diffStyles
    - stylespec/diff.js -> diffStyles(before: StyleSpecification, after: StyleSpecification) -> Diff two stylesheet
- Mapbox GL Style Specification
    - style-spec
        - types.js -> StyleSpecification -> setStyle, diffStyle
        - diff.js -> Diff for updates
        - validate_style.js

WorkerTile:

Source
- The Source interface must be implemented by each source type, including "core" types (`vector`, `raster`,
  `video`, etc.) and all custom, third-party types
- implementations of the interface -> VectorTileSource, GeoJsonTileSource
- Members
    - tileSize, minZoom, maxZoom
    - loadTile
    - sourceType -> vector, raster, raster-dem, geojson, video, image, canvas


SourceCache
- creating an instance of Source -> 1 to 1 matching with a source (this._source) e.g. VectorTileSource
- caching tiles loaded from an instance of Source
- loading the tiles needed to render a given viewport
- unloading the cached tiles not needed to render a given viewport
- Members
    - update() -> Removes tiles that are outside the viewport and adds new tiles that are inside the viewport
    - getSource()
    - getTile(OverscaledTileID) -> called by the draw_* methods
    - _source: VectorTileSource
    - transform: Transform
    - cache: TileCache
    - tileId: OverscaleTileID
    - _tiles:Tile -> buckets
    - tileCacheSize
    -   Larger viewports use more tiles and need larger caches -> larger viewports
        are more likely to be found on devices with more memory and on pages where
        the map is more important

Program:
- WebGLProgram as member
- draw
    - context.program.set(this.program) -> gl.useProgram
    - context.setDepthMode
    - context.setStencilMode
    - context.setColorMode
    - vao.bind
    - gl.drawElements
    - drawLine
        - called for every layer family? (layout) -> all Tiles withing the SourceCache are drawn
            - all with the same ref properties -> ref_properties
        - get styles from layer -> layer.paint.get('line-opacity'). width = layer.paint.get('line-width')
        - get Tile from sourceCache -> Tile has buckets -> tile.getBucket(layer)
            - Tile has a bucket for every source layer (landcover, hillshade, ...)
        - configure Program with parameter from bucket
        - program.draw(context, gl.TRIANGLES, uniformValues)
Painter:
- WebGLRenderingContext
- Transform -> TransformationMatrix
- has a Program cache -> one program for every layer type (background, fill, ...)
- The state needed by the painter_xxx modules to render tiles is split up into two places:
    - The map-wide Style / StyleLayer structure, which dictates the layers to be rendered, their types,
      and the layout/paint definitions that control their details.
    - The per-source Tile objects (managed by SourceCache, populated by Sources, often via Worker requests), which hold Buckets (one per StyleLayer) of buffers containing vertex positions, colors, etc.

OverscaleTileId:
- posMatrix

ProgramConfiguration:
- contains uniforms -> u_width, u_color
- bucket.programConfigurations
- painter.useProgram(programId, programConfiguration)

drawLine:
- for every OverscalingTileID get Tile
    - getLinBucket from Tile
    - getProgram with ProgramConfiguration -> paitner.useProgram
    - get Uniform Values -> u_matrix, u_device_pixel_ratio, u_ratio, u_units_to_pixels -> lineUniformValues(painter, tile, layer)
        -> painter.translatePosMatrix is called
    - program.draw(context, uniformValues, bucket.layoutVertexBuffer, bucket.indexBuffer, bucket.segments, programConfiguration)
        - context.program.set(this.program);
        - context.setDepthMode(depthMode);
        - context.setStencilMode(stencilMode);
        - context.setColorMode(colorMode);
        - context.setCullFace(cullFaceMode);
        - set uniforms -> UniformMatrix4f.set
        - for segment of segments.get() -> vao.bind, gl.drawElements

VAO
- VAOs belong to Bucket which is recreated only during layout changes,
  though VAO attributes may change during a paint property change (e.g. fill-pattern)

Transform:
- for transforming a single tile
- When moving the map first _calcMatrices (projMatrix, invProjMatrix, pixelMatrix, mercatorMatrix) is called and then calculatePosMatrix
- projMatrix -> matrix for conversion from location to GL coordinates (-1 .. 1)
- labelPlaneMatrix
    -> transforms from NDC (-1, 1) to pixel Coordinates
- pixelMatrix -> matrix for conversion from location to screen coordinates
    -> multiplies the labelPlaneMatrix with the projectionMatrix
    -> the projectionMatrix converts to NDC
    -> the labelPlaneMatrix converts to pixel coordinates -> order first translate then scale -> scale * translate * vec
        -> Translates the coordinate system it to top left -> mat4.translate -1, 1
            -> calculating the diff from the source crs to the dst crs
        -> mat4.scale this.width / 2, -this.height / 2 -> the negative y value also rotates the y-axis
- mercatorMatrix -> The mercatorMatrix can be used to transform points from mercator coordinates ([0, 0] nw, [1, 1] se) to GL coordinates.
- calculatePosMatrix -> Calculate the posMatrix that, given a tile coordinate, would be used to display the tile on a map
    - called in sourc_cache.getVisibleCoordinates -> posMatrix is added to OverscaleTileId
    - multiplied with the projection Matrix which is calculate in _calcMatrices -> to translate for tile?
    - painter.render -> sourceCache.getVisibleCoordiantes()
    - per Tile the posMatrix is calculated because the rendering happens per tile (layer)?
- _calcMatrices -> mat4.perspective
  - _calMatrices is executed when set bearing, set pitch, set fov, set zoom, set center, ... is called
- Matrices
    - pixelMatrix
- coveringTiles
    - Return all coordinates that could cover this transform for a covering zoom level
- variables
    - this.scale -> numberOfTiles per ZoomLevel per row -> 2^exactZoomLevel
    - this.zoomScale -> pixelPerTile
    - worldSize -> tilesAtZoom -> numberOfPixels per ZoomLevel per row -> this.tileSize(512) * this.scale
    - postMatrix.scale -> pixel per Tile
    - unwrappedX -> index tile in x row
    - canonical -> exact Tile Index
    - translation matrix -> number of Pixels from origin on exact zoomLevel e.g. 6.4
    - scaling matrix -> scaled to 8192 e.g. from 576.04
    - cameraToCenterDistance -> Z value from the camera to the map
      -> if no pitch value -> value e.g. 1329 equals the z value of a geometry multiplied with the projectionMatrix
    - EXTENT
      -> The maximum value of a coordinate in the internal tile coordinate system -> Coordinates of all source features normalized to this extent upon load
      -> Vertex buffer store positions as signed 16 bit integers
    - PositonMatrix-> Pixel coordinates for the origin of the specified tile for the exact zoomLevel based on 8192 pixel per tiles
    - projMatrix
        - Transformation of PixelCoordinates to NDC? -> View Projection Matrix for transforming World Coordinates to NDC?
        - translation in Pixel Coordiantes from Center is calculated -> project(center: LatLng) -> -x, -y in pixels
        - bearing and pitch is rotaed
    - gl coordinates? -> -1 ... 1
      -> clip space coordinates in the demo html -> larger values then 1 in the demo -> multiplied with the projection matrix -> before projective divison to NDC
    - tile coordinates — the a_pos in the vertex shader — range from 0-8192
        - foreach individual tile which gets placed on our screen a projection matrix is calculated
        - It's rendered tile by tile and for each tile a projection matrix is calculated -> transformation to clip space (-1 .. 1)
        - translate a tile from 0-8192 to clip space -> Clip space is actually one step away from NDC, all coordinates are divided by Clip.W to produce NDC
            -> Clip space is where the space points are in after the point transformation by the projection matrix, but before the normalisation by w
        - uniform in vertex shader (u_matrix) which transforms the tile coordiantes (a_position) into WebGL clip space
- Workflow
    - On every change to the viewport (set bearing, set pitch, set fov, set zoom, set center,) _calcMatrices is called where different matrices are calculated (projMatrix, invProjMatrix, pixelMatrix, mercatorMatrix)
    - painter.render -> source_cache.getVisibleCoordinates():Array<OverscaleTileID> -> calculatePosMatrix()
        - posMatrix is added to OverscaleTileID -> OverscaledTileID has a postMatrix
        - calculatePosMatrix uses the matrices calculated from _calMatrices
    - painter.render -> renderLayer(this, sourceCache, layer, coords) -> draw[layer.type](painter, sourceCache, layer, coords, this.style.placement.variableOffsets)
    - is positionMatrix set as uniform?

Tile Loading
- Limitation in the pitch to 60 degrees in v1 because of the number of tiles to load
- Basically we should change the tile loading algorithm so that it loads lower zoom tiles on
  farther side, depending on how small they appear when projected
- besides level-of-detail tile loading we'll also need to fix:
  label-placement (the tiled placement approach will completely fall apart) and antialiasing
  -> labels have to be put up when have large pitches see v2
- map._sourcesDirty
    - When map._sourcesDirty === true, it starts by asking each source if it needs to load any new data
    - map._update() central place for setting the sourceDirty flag and trigger a repaint -> true when style changed -> map's style and sources
      -> map._update() called by setPaintProperty(), setLayoutProperty(), setFilter(), setLayerZoomRange(),
         removeLayer(), addLayer(), removeSource(), addSource(), ...
    - _render -> if _sourceDirty === true -> style.updateSource(transform)
- Tile retention logic (Aufbewahrungslogik)
    - SourceCache.update -> _retainLoadedChildren -> iterate over all Tiles in Cache
    -  For a given set of tiles, retain children that are loaded and have a zoom between `zoom` (exclusive) and `maxCoveringZoom` (inclusive)
- No wrap -> renderWorldCopies is false
- Zoom
    - Maxzoom refers the maximum zoom of source tiles, not the maximum zoom over overscaled tiles
    - minCoveringZoom and maxCoveringZoom are used only for looking up loaded parent and child tiles,
      some of which many be overscaled tiles with z > maxzoom

Type of Tile Ids:
- CanonicalTileID
  - has z, x, and y, with x/y being within the bounds z defines -> z can be anything from 0-32
  - Used for requesting data -> represents data tiles that exist out there
  - z is never larger than the source's maxzoom
- OverscaledTileID
  - is composed of a z value, and a canonical tile ->  Tte z value indicates the zoom level the tile is intended for
  - It is primarily used for indexing overscaled data tiles
  - overscaledZ describes the zoom level this tile is intented to represent, e.g. when parsing data
  - holds a posMatrix -> Over the OverscaledTileID classes is iterate in the draw_* files
    -> the posMatrix is set as uniform (u_matrix)
    -> the connetn of the tile (buckets) is queried via sourceCache.getTile(OverscaledTileID)
- UnwrappedTileID
  - is composed of a wrap value, and a canonical tile -> the wrap value is used for representing tiles to the left and right of the main (0/0/0 based) tile pyramid
  - It is primarily used for indicating the position a tile should be rendered at
  - Wrap describes tiles that are left/right of the main tile pyramid, e.g. when wrapping the world
  - Used for describing what position tiles are getting rendered at (= calc the matrix)
  - On top of the regular z/x/y values, TileIDs have a `wrap` value that specify
    which cppy of the world the tile belongs to -> For example, at `lng: 10` you
    might render z/x/y/0 while at `lng: 370` you would render z/x/y/1
  - When lng values get wrapped (going from `lng: 370` to `long: 10`) you expect
    to see the same thing on the screen (370 degrees and 10 degrees is the same
    place in the world) but all the TileIDs will have different wrap values

  - sourceCache.update -> Transform.getVisibleUnwrappedCoordinates -> Return any "wrapped" copies of a given tile coordinate that are visible in the current view
  - sourceCache.update -> sourceCache.handleWrapJump
Cleanup
- Tile.unloadVectorData() -> Release any data or WebGL resources referenced by this tile -> this.buckets[i].destroy()
- Bucket.destroy
    - Release the WebGL resources associated with the buffers
    - Note that because buckets are shared between layers having the same layout properties, they
      must be destroyed in groups (all buckets for a tile, or all symbol buckets)
    - segments.destroy(), layoutVertexBuffer.destroy(), indexBuffer.destroy(), ... -> gl.deleteBuffer()

GeoJson handling
- geojson-vt lib turns GeoJSON data into vector tiles
- see https://blog.mapbox.com/rendering-big-geodata-on-the-fly-with-geojson-vt-4e4d2a5dd1f2
- Addint a GeoJson source
    - map.addSource -> style.addSource(SourceSpecification) -> new SourceCache()
      -> source.js createSource -> GeoJsonSource -> createSource within the source module is a factory for creating a specific Source (GeoJsonSource, VectorTilesSource, ...)
      -> Source interface has to be implemented by all sources, also custom ones

Questions:
- where is the transformation matrix calculated?
- which coordinates are stored in a bucket?
- groupByLayout? -> creates a Bucket for each 'family' of style layers that share the same underlying features and 'layout' properties
- bucket coordinates -> MercatorCoordinate?
- workflow -> map class
- OverzoomedTile, UnwrappedTile, CanoncialTile
- w -> w=1 neutral scaling
    - Why four dimensions instead of 3 -> Very roughly: we use 'w' to calculate perspective scaling effects
    - Note that the 'perspective' matrix is the only transformation that directly modifies 'w'
- grouping of the layer to a LayerGroup and creation of the buckets?
sourceCache -> Tile -> Bucket(layer)

WebGL
- Vertex Buffer Object
- Vertex Array Object

General
- paint
  - line-color, line-width, line-opacity, line-dasharray
  - fill-color, fill-opacity, fill-pattern (image)
  - circle-radius, circle-color, circle-stroke-color, circle-stroke-width, circle-opacity
- layout
  - line-join, line-cap
  - icon-image, icon-rotate, icon-rotation-alignment (map), icon-allow-overlap, icon-ignore-placement
  - text-field, text-font, text-offset, text-anchor
  - visibility

Textures
-> ImageSource -> implements Source
    -> Data Source containing an image -> source type image
-> ImageSource.load -> ajax
-> map.render -> Painter.render -> sourceCache.prepare -> ImageSource.prepare

- Map._render
  -> Update style -> layer.recalculcate -> recumpute paint properties on every layert type
  -> Update Source cache
  -> Painter.render
    -> sourceCache.prepare(this.context);
    -> offscreen render pass -> foreach layer -> renderLayer
    -> opaque render pass -> foreach layers
        -> get layer, sourceCache and coords (sourceCache.getVisibleCoordinates() -> adds posMatrix)
        -> renderLayer(sourCache, layer, coords)
    -> transcluent render pass -> foreach layer -> renderLayer(sourceCache, layer, coords)
  -> drawFill
    -> tile.getBucekt(layer)
    -> bucket.programConfigurations.get(layer.id) -> binders -> painter.useProgram(programConfiguraiton)
    -> program.draw
        -> fixedUnfiorms.set -> u_matrix
        -> configuration.setUniforms -> binders -> paint properties
        -> foreach segment
            -> vao.bind
            -> gl.drawElements

-> Binding Uniforms -> ProgramConfiguration
    -> Binder is the interface definition for the strategies for constructing, uploading, and binding paint property data as GLSL attributes
    -> ProgramConfiguration
        -> ProgramConfiguration contains the logic for binding style layer properties and tile
            layer feature data into GL program uniforms and vertex attributes
        -> Non-data-driven property values are bound to shader uniforms
        -> Data-driven property values are bound to vertex attributes
        -> In order to support a uniform GLSL syntax over both, [Mapbox GL Shaders](https://github.com/mapbox/mapbox-gl-shaders)
            defines a `#pragma` abstraction, which ProgramConfiguration is responsible for implementing
    -> Program#draw functions to
        -> set color, stencil, and depth modes
        -> set all uniforms (both fixedUniforms and paint property-based uniforms,
            henceforth binderUniforms; the state bindings for these are initialized
            upon program construction
        -> binder Unfiforms -> u_opacity, u_outline_color -> ProgramConfiguration.setUniform -> binders are used
        -> fixed Uniforms -> u_matrix, u_world  -> Uniform2f.set
    -> AttributeBinder
        -> SourceExpressionBinder
        -> CrossFadedCompositeBinder
    -> UniformBinder
        -> ConstantBinder
        -> CrossFadedConstantBinder
    -> Attribute and Uniform binder
        -> CompositeExpressionBinder
- Vertex
    - prepareSegment is sort of a bookkeeping and utility function used to make sure the vertex array
      and index array we’re going to use isn’t already too full
        -> these arrays have a maximum capacity of 65535, so if adding this feature would push
      them over the limit, prepareSegment takes care of creating new arrays
    - holeIndices is present in fill_bucket and fill_extrusion_bucket
        -> it’s just doing bookkeeping necessary for running the earcut algorithm

Review:
- Updating the Docs has to be added to DOD -> otherwise such detailed description get fast out of date
- Explain when a source is dirty
- Add Note -> Remove tiles outside the viewport and adds new
- Add covering tiles step on transform
- EventLoop Diagram
    -> Map inherits from Camera -> make one box out of it ?
    -> Add call to _calcMatrices to transfrom box after update -> central for updating the
  inernal matrices like projectionMatrix -> important step for turning tile coordinates in GL coordinates
        -> or matrices update lik projectionMatrix
- Add posMatrix step to diagram
- Start with _update in Map -> because central entry point ?
- Add draw_* and program step
- does upload of vertices realy happen in sourcecache.prepare -> tile.upload -> bucket.upload

## Workflow vector tiles rendering

@startuml
... Render Vector Tiles ...
mapboxgl.Map -> VectorTileWorkerSource
VectorTileWorkerSource -> WorkerTile: parse(vectorTile)
WorkerTile -> Bucket: foreach layer group
note right
Each bucket holds the vertex and element array data
needed to render its group of style layers
end note
WorkerTile -> Painter: Buckets
Painter -> render_.js: Delegates
@enduml

## Workflow rendering based on user interaction

@startuml
mapboxgl.Map -> mapboxgl.Map: easeTo
activate mapboxgl.Map
mapboxgl.Map -> mapboxgl.Map: _ease
activate mapboxgl.Map
mapboxgl.Map -> mapboxgl.Map: _requestRenderFrame
activate mapboxgl.Map
mapboxgl.Map -> mapboxgl.Map: _update
note right
sets the _sourcesDirty flag and possibly
also the _styleDiry flag
end note
activate mapboxgl.Map
mapboxgl.Map -> mapboxgl.Map: triggerRepaint
activate mapboxgl.Map
mapboxgl.Map -> mapboxgl.Map: _render
activate mapboxgl.Map
mapboxgl.Map -> Style: update
note right
Apply queued style updates in a batch and
recalculate zoom-dependent paint properties
end note
mapboxgl.Map -> Style: updateSources(Transform)
Style -> SourceCache: update(Transform)
note right
Removes tiles outside the viewport and
adds new tiles that are inside the viewport
end note
SourceCache -> Transform: coveringTiles(tileSize, minZoom, maxZoom, ...)
Transform -> SourceCache: OversceledTileID[]
mapboxgl.Map -> Painter: render(style, painterOptions)
note right
Actually draw
end note
loop
Painter -> Painter: renderLayer(sourceCache, styleLayer)
activate Painter
Painter -> draw_: draw*(Painter, SourceCache, StyleLayer, OverScaleTileID[])
note right
drawLine, drawFill, ...
end note
draw_ -> Painter: useProgram
draw_ -> Program: draw(context)
note right
gl.useProgram, context.setDepthMode,
context.setStencilMode, context.setColorMode
vao.bind, gl.drawElements
end note
deactivate Painter
end
deactivate mapboxgl.Map
@enduml


