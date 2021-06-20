Topics
1. Calculation of the visible Tiles in the current viewport
    -> Scanline vs level of detail
2.  Tile Loading
3. Tile Caching
    -> LRU Cache
4. Tile Cache API
5. Tile processing


5. Tile processing
-> Questions
   -> How the url for the web worker is constructed
   -> How the methods in the worker class are called e.g. loadTile
   -> How are the actor.send queued?

-> SourceCache -> Source -> Worker -> VectorTileWorkerSource/GeoJsonWorkerSource -> WorkerTile


-> Worker pool, Actor
-> Dispatcher
   -> this.dispatcher.getActor
   -> Actor[], WorkerPool


Worker
-> bundle_prelude generates object url for WebWorker -> src/source/worker.js ?
-> Member
    -> loadTile

WorkerPool
-> Number of WebWorker -> half size of available processors -> window.navigator.hardwareConcurrency / 2

Dispatcher
-> is created in the Style ctor
-> Member
    -> actors: Actor[]
    -> workerPool: WorkerPool -> globalWorkerPool
    -> broadcast -> message to all actors
    -> getActor
-> creates Actors and assigns Web Worker to them from the WorkerPool
-> Number of actors is the number of WebWorker from the WorkerPool

Actor
-> has a target -> can be a WebWorker
    -> on that wokrer postMessage is called
-> assigned to a Tile
-> tile.actor.send('loadTile', params, done.bind(this))
-> send -> Sends a message from a main-thread map to a Worker or from a Worker back to a main-thread map instance
-> member
    -> receive
        -> execute callback with specific id -> id is returned
        -> is binded in the ctor of the actor -> this.target.addEventListener('message', this.receive, false);
    -> send(methodNameToCall, data, ‚callback)
    -> processTask -> is called on the WebWorker and calls postMessage

web_worker_transfer
-> serialize and deserilize
-> The structured clone algorithm copies complex JavaScript objects
-> It is used internally to transfer data between Workers via postMessage(),
   storing objects with IndexedDB, or copying objects for other APIs
-> Supported types -> All primitive types, ArrayBuffer, Blob, ....
-> Structured cloning is great, but it's still a copy operation
-> The overhead of passing a 32MB ArrayBuffer to a Worker can be hundreds of milliseconds
-> New versions of browsers contain a huge performance improvement for message passing, called Transferable Objects
-> Think of it as pass-by-reference if you're from the C/C++ world -> zero-copy
-> unlike pass-by-reference, the 'version' from the calling context is no longer available once transferred to the new context
-> Transferable -> ArrayBuffer, MessagePort, ImageBitmap and OffscreenCanvas

@startuml
... Load Vector Tiles ...
mapboxgl.Map -> SourceCache: update(transform)
SourceCache -> VectorTileSource: loadTile(Tile)
note right
this.dispatcher.getActor().send('loadTile', params, done.bind(this));
end note
VectorTileSource -> Actor: send('loadTile')
note right
Sends a message from a main-thread map to a Worker or
from a Worker back to a main-thread map instance -> web_worker_transfer.serialize
end note
Actor -> Worker: postMessage('loadTile', buffers)
Worker -> Worker: getWorkerSource
Worker -> VectorTileWorkerSource: loadTile
VectorTileWorkerSource -> WorkerTile: parse
VectorTileWorkerSource -> WorkerTile: parse(vectorTile)
WorkerTile -> Bucket: foreach layer group
note right
Each bucket holds the vertex and element array data
needed to render its group of style layers
end note
WorkerTile -> Painter: Buckets
Painter -> render_.js: Delegates
@enduml





4. Tile Cache API
-> The Cache interface provides a persistent storage mechanism for Request / Response
   object pairs that are cached in long lived memory
-> Note that the Cache interface is exposed to windowed scopes as well as workers -> You don't have
   to use it in conjunction with service workers, even though it is defined in the service worker spec
    -> window.caches.open(CACHE_NAME)
-> tile_request_cache


-> SourceCache
    -> getTileById(): Tile
    -> painter -> SourceCache.getVisibleCoordinates(): Array<OverscaledTileID> -> getRenderableIDs
        -> renderables -> this._tiles
        -> coord.posMatrix
    -> Type of tiles
        -> ideal Tiles -> transform.coveringTile -> Return all coordinates (OverscaledTileID) that could cover this transform for a covering zoom level
        -> renderable Tiles -> Tile.hasData, ...
        -> retained Tiles -> ideal tiles, children tiles, parent tiles
            -> Retain is a list of tiles that we shouldn't delete, even if they are not the most ideal tile
                for the current viewport -> this may include tiles like parent or child tiles that are already loaded
                -> retain any loaded children of ideal tiles up to maxCoveringZoom -> form missing ideal tiles
                -> We couldn't find child tiles that entirely cover the ideal tile -> look for parents now
    -> _tiles vs TileCache?
    -> Traversal -> first children then parents?

-> TileCache
    -> CacheSize -> commonZoomRange (5) * tilesInView
    -> data -> Map with Tiles and timeout
    -> order -> array with ordered keys
    -> SourceCache -> TileCache.getByKey
    -> SourceCache._addTile -> Tile loading
        -> if (!cached) -> _loadTile -> TileCache.getAndRemove
        -> Add tile from cache to _tiles if present


