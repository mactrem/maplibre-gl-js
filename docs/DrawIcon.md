

Icon Click
-> handler -> _createDelegatedListener

queryRenderedFeatures
-> style.queryRenderedFeatures(queryGeometry, options, this.transform)
    -> distinction between geometries (grid-index) and symbols (CollisionIndex)
-> query_features.js -> queryRenderedFeatures
    -> sourceCache.tilesIn(pixelCoordinates): TileIn[] -> {queryGeometry, Tile} -> Calculate tile coordiantes from pixel coordinates
        -> transform.pointCoordinate -> pixelCoordinates to mercator coordinates -> via pixelMatrixInverse
    -> Tile.queryRenderedFeatures -> Queries non-symbol features rendered for this tile -> Symbol features are queried globally
        -> FeatureIndex.query -> Finds non-symbol features in this tile at a particular position
            -> FeatureIndex uses grid-index
    -> CollisionIndex.queryRenderedSymbols(queryGeometry);
        -> collision index used to prevent symbols from overlapping -> uses grid-index library with query method
        -> used by the placement class
        -> Because the geometries in the CollisionIndex are an approximation of the shape of symbols on the map,
           we use the CollisionIndex to look up the symbol part of queryRenderedFeatures
