-> FillStyleLayer -> extends StyleLayer
    -> StyleSpecification as Ctor argument
    -> paint properties for drawFill -> PossibleEvaluatedProperty
    -> paint.get("fill-style")
    -> createBucket
    -> ProgramConfiguration.setUnfiorms

-> Workflow
    -> map._render -> painter.render -> painter.renderLayer -> drawFill

Questions
-> where are the VBO for the layouts bound
-> where are the VBO for data driven paint properties bound
-> for what is dynamicVertexBuffer, dynamicVertexBuffer2 and paintVertexBuffers in the VertexArrayObject
-> How many paint buffer are created -> for every buffer a binder
-> Are expression and then binders also allowed for layout properties
-> is there one program per bucket -> a bucket should have the same layout properties
    -> so probably when other layout properties the pragmas in the shader has to be changed
    -> but also on datadriven styling in paint propterties there are other pragmas
-> Where are the shaders compiled -> ProgramConfiguration
-> TileClippngMask -> Painter -> renderTileClippingMasks

-> indexBuffer2 and segments2 is in fill for outline
-> one ProgramConfiguration for every layer
-> Correct the Archtitecture.md about ArrayGroups and BufferGroups -> not exist anymore


-> Program
    -> How Ctor works?
    -> this.numAttributes = gl.getProgramParameter(this.program, gl.ACTIVE_ATTRIBUTES)


-> Attributes
    -> staticAttriubtes -> a_pos
    -> dynamicAttributes -> a_color, a_outline_color
    -> statiUniforms
    -> dynamicUniforms

@startuml
... Render Fill Style ...
mapboxgl.Map -> mapboxgl.Map: triggerRepaint
activate mapboxgl.Map
mapboxgl.Map -> mapboxgl.Map: map._render
activate mapboxgl.Map
mapboxgl.Map -> Painter: render(Style, PainterOptions)
note right
Upload vertices and image textures (patterns, icons) to GPU:
SourceCache.prepare(Context)
    -> Tile.upload(Context)
        -> FillBucket.upload(Context)
            -> context.createVertexBuffer (layoutVertexArray) -> gl.bufferData
            -> context.createIndexBuffer (layoutAttributes)
            -> programConfigurations.upload(context) -> binder.upload() e.g. SourceExpressionBinder.upload()
                -> only update when type of SourceExpressionBinder, CompositeExpressionBinder, CompositeExpressionBinder, ...
                -> SourceExpressionBinder implements AttributeBinder -> ("color": ["get", "color"])
                -> ConstantBinder implements UnfiformBinder -> setUniform
                -> Binder -> definition for the strategies for constructing, uploading, and binding paint property data as GLSL attributes
                    -> paintVertexAttributes -> components: 2, name: "a_color", offset: 0, type: "Float32"
                    -> new ProgramConfiguation -> Binders are created -> in Worker?
                    -> One binder for every layer.paint._values -> PossiblyEvaluatedPropertyValue
                -> does painter configure the binders in ProgramConfiguration.value -> expression
                -> ConstantBinder e.g. for colors are uploaded via uniforms in Program.draw
end note
activate Painter
Painter -> Painter: drawLayer(SourceCache, FillStyleLayer, OverscaledTileID[])
activate Painter
Painter -> drawFill: drawFill
activate Painter
drawFill -> drawFill: drawFillTiles
note right
foreach OversaledTileID
tile = sourceCache.getTile(OversaledTileID)
tile.getBucket(FillStyleLayer)
programConfiguration = bucket.programConfigurations.get(layer.id)
program = painter.useProgram(programName, programConfiguration)
end note
activate drawFill
drawFill -> Program: draw
note right
-> Fixed Uniforms -> set fixedUnforms (u_matrix) -> gl.uniformMatrix4fv()
-> Binder Uniforms-> ProgramConfiguration.setUniforms(context, binderUniforms)
    -> Only ConstantBinder are uploaded e.g. fill-color (u_color), fill-opacity (u_opacity) when not expression is used
    -> ConstantBinder.setUniform -> gl.uniform4f(this.location, ...)
VertexArrayObject.bind(layoutVertexBuffer, indexBuffer, dynamicLayoutBuffer, dynamicLayoutBuffer2)
    -> when fresh bind for layoutVertexBuffer, paintVertexBuffers and IndexBuffer
            -> context.extVertexArrayObject.createVertexArrayOES(),
                vertexBuffer.enableAttributes(gl, program), setVertexAttribPointers()
    -> when no fresh bind -> context.bindVertexArrayOES.set(this.vao);
gl.drawElements()
end note
@enduml
