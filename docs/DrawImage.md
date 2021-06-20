

- ImageSource
		- Konverierung in Tile Coordiantes -> 0 - 8192
		- Es wird ein Tile berechnet wo dass Image im Zentrum liegt
		- const zoom = Math.max(0, Math.floor(-Math.log(dMax) / Math.LN2)) -> dmax in Mercator Coordinates (0..1)
		    -> 2^zoom = numTilesPerRow
  			-> zoom = log2(numTilePerRow) -> Math.log(numTilePerRow) / Math.LN2
  			-> dMax 0.1 -> 10 Tile per ZoomLevel when - is added before Math.log -> Math.log(10) === -Math.log(0.1) -> -Math.log(5) === -Math.log(0.2)
  			-> so the zoomLevel is calculated in which the image matches a tile
		- Mipmaps only make sense when the texture is scaled down significantly (way below 50% of its original size), as a mean to improve the quality of the texels used to calculate the fragment color
            Anisotropic filtering will make textures clearer when viewed at an angle

-> Painter verewaltet Tiles? -> holen und löschen -> d.h. arbeitet generische auf SourceCache und
	nicht auf spezieller Implementierung d.h. nur auf Tile basis

-> Map._render -> Style.updateSources -> SourceCache.update -> SourceCache.updateRetainedTiles -> SourceCache._addTile -> ImageSource.loadTile

-> SourceCache
	-> loadTile
	-> unloadTile
	-> prepare -> tile.upload, tile.prepare
-> ImageSource
	-> loadTile
	-> unloadTile -> Tile.unloadVectorData -> buckets.destroy
	-> tileId -> Property which only has ImageSource -> used in the SourceCache to get the Tile
		-> for the other sources transform.coverintTiles is calculated

-> WebGL
	-> gl.createTexture();
	-> gl.bindTexture(gl.TEXTURE_2D, this.texture);
    -> gl.pixelStorei(gl.UNPACK_ALIGNMENT, v);
	-> gl.pixelStorei(gl.UNPACK_PREMULTIPLY_ALPHA_WEBGL, (v: any));
	-> gl.texImage2D(gl.TEXTURE_2D, 0, this.format, this.format, gl.UNSIGNED_BYTE, image);
	-> gl.generateMipmap(gl.TEXTURE_2D); -> this.useMipmap && this.isSizePowerOfTwo
