# 基于ThreeJS的大屏3D地图（二）——气泡图、渐变柱体与热力图

## 前提

上一篇文章中我们完成了地图区块模型的渲染，在此基础之上本篇来讲解气泡图、3D柱形图以及3D热力图的实现方式。

首先，为了更好的关注点分离及与地图渲染模块的解耦，我们可以把所有类型的可视化元素抽象出一个图层基类`BaseLayer`：

```typescript
/**
 * 图层基类
 */
abstract class BaseLayer {

    map: ThreeMap

    uuid: string

    cfg: any

    setting: LayerSetting
    
    /**
     * 初始化时机
     */
    abstract init(initCfg): void

    /**
     * 每帧更新函数
     */
    abstract update(timeStamp): void
    
    /**
     * 销毁函数
     */
    abstract destroy(): void

    /**
     * 显示图层
     */
    abstract show(): void

    /**
     * 隐藏图层
     */
    abstract hide(): void

    /*
     *
     */
    // ......
}
```

其中`ThreeMap`类型为上一篇中所实现的地图实例；LayerSetting为图层的配置内容。

图层的更新、销毁等生命周期交由地图/使用方接管。

```typescript
class Threemap {

    constructor() {
        /* ... */
        const animate = (timeStamp) => {
            /* 更新图层 */
            this.layers.forEach(layer => {
                layer.update(timeStamp)
            })

            this.requestID = requestAnimationFrame(animate)
        }
        /* ... */
    }

    /*
    ......
    */

    public addLayer(layer: BaseLayer, initCfg?) {
        this.layers.push(layer)
        layer.map = this
        layer.init(initCfg)
    }

    public removeLayer(layer: BaseLayer) {
        const idx = this.layers.findIndex(curLayer => curLayer === layer)
        if(idx >= 0) {
            this.layers.splice(idx, 1)
        }
        
        layer.destroy()
    }

    public clearLayer() {
        this.layers.forEach(layer => {
            layer.destroy()
        })

        this.layers = []
    }
}
```



## 气泡图

气泡包括散点和作扩散动画的波纹两部分组成。其中散点的半径大小通常是由该点所对应值的大小计算而来：

```typescript
type RenderData = {
    name: string
    value: [number, number, number]
}
```

```typescript
scatterGroup: Group
circleGroup: Group

private createFeatures() {
    this.clear()

    const { minDiameter, maxDiameter } = this.setting

    this.data.forEach((item: RenderData) => {
        /* 计算气泡大小 */
        const t = item.value[2] / this.maxValue
        const diameter = lerp(minDiameter, maxDiameter, t), radius = diameter / 2
        /* 散点 */
        const scatter = this.createScatter(item, radius)
        this.scatterGroup.add(scatter)
		/* 波纹 */
        const circles = this.createCircle(item, radius)
        this.circleGroup.add(circles)
    })
}
```



### 创建散点

```typescript
private createScatter(item: RenderData, radius) {
    const { color, opacity } = this.setting
    const [x, y] = item.value

    const material = new MeshBasicMaterial({ color, opacity, transparent: true }), geometry = new CircleGeometry(radius, 50)
    const mesh = new Mesh(geometry, material)
    mesh.position.set(x, y, this.map.cfg.depth + 1)

    mesh.name = item.name
    mesh.userData = {/* */}

    return mesh
}
```



### 创建波纹

通过`Curve`类曲线来绘制圆圈生成`Line`需要的几何体。常量`Circle_Count`用来指定一个散点上的扩散动画中的波纹数量，同组的波纹用一个`Group`来组织：

```typescript
const Circle_Count = 3

private createCircle(item: RenderData, radius) {
    const { color } = this.setting
    const [x, y] = item.value

    const arc = new ArcCurve(0, 0, radius)
    const points = arc.getPoints(50)
    const geometry = new BufferGeometry().setFromPoints(points)

    const circlesGroup = new Group()

    for(let i = 0; i < Circle_Count; ++i) {
        const material = new LineBasicMaterial({ color, opacity: 1, transparent: true })
        const circle = new Line(geometry, material)

        circle.position.set(x, y, this.map.cfg.depth + 1)

        circlesGroup.add(circle)
    }

    return circlesGroup
}
```

![](./bubble_static.png)



### 波纹动画

波纹的扩散动画我们就参照Echarts的地图气泡图来：每一个散点的外围会同时存在`Circle_Count`个波纹，随着时间推移波纹半径会逐步增大，且透明度也逐步增大，在达到指定的最大半径时透明度也趋近于0最终消失；接着从又立即生成一个同散点大小一致的新波纹。这些波纹互相之间的间距和生成时间都是**”等距“**的。

在图层的`update`函数中调用一个`animateCircles`函数。在该函数中实现根据传入的时间戳更新每个波纹的大小及透明度：

```typescript
update(timeStamp): void {
    /* ... */
    this.animateCircles(timeStamp)
}
```

再定义如下常量：`Circle_Max_Scale`指定波纹的半径扩散到多大时消失，`Circle_Remain_Time`指定波纹从出生到消失的持续时间。

```typescript
const Circle_Count = 3,				// 波纹数量
      Circle_Max_Scale = 2,			// 波纹扩散的最大半径
      Circle_Remain_Time = 3000,	// 单次动画的生命周期
      Circle_Unit_Step = 1 / Circle_Count	// 步进单位
```

虽然知道了波纹扩散的持续时间和变化的半径区间，接下来就需要计算在某一时刻下各个波纹的状态。

以`Circle_Remain_Time`作为一个周期的话，用当前的时间戳取余它，得到的就是此时刻动画所已经历的时间`curLifeTime`。由前文知道各个波纹之间的进度是等距的，那么以1为单个周期则每个波纹在进度上的间隔就是`Circle_Unit_Step = 1 / Circle_Count`。

从下标0开始，对于第`i`个波纹，它当前所处的进度就是：`step = curLifeTime / Circle_Remain_Time + i * Circle_Unit_Step`。这个`step`值是可能大于1的，因此还需要再对1做一次取余，应用取余1后的值相当于重新从散点处开始扩散；这么一来就能不断复用这`Circle_Count`个波纹无需重新创建。最后再基于得出`step`值设置波纹此时的大小`scale`和透明度`opacity`：

```typescript
private animateCircles(timeStamp) {
    this.circleGroup.children.forEach(circles => {

        for(let i = 0; i < Circle_Count; ++i) {
            const circle = circles.children[i] as Line

            let step: number, scale: number, material = circle.material as Material

            const curLifeTime = timeStamp % Circle_Remain_Time
            step = (curLifeTime / Circle_Remain_Time + i * Circle_Unit_Step) % 1

            scale = lerp(1, Circle_Max_Scale, step)
            circle.scale.set(scale, scale, 1)
            material.opacity = 1 - lerp(0, 1, step)
        }
        
    })
}
```

![](./bubble_animation.gif)



## 3D渐变柱体

### 创建柱体

3D柱体的形状我们支持四种常见的类型：四方体、圆柱、三角柱和六角柱。圆柱和四方体可以通过`three.js`内置的`CylinderGeometry`和`BoxGeometry`来创建几何体。三角柱和六角柱则可以先使用`Shape`绘制平面等边三角形/六边形，接着通过`ExtrudeGeometry`来挤压出立体形状：

```typescript
private createColumn(shape: string, pos: number[], height: number, userData: any) {
    const { width, color } = this.setting
    const [x, y] = pos
    let geometry: BufferGeometry

    switch(shape) {
        case 'cylinder':        // 圆柱体
            geometry = new CylinderGeometry(width / 2, width / 2, height)
            geometry.rotateX(Math.PI / 2)
            geometry.translate(0, 0, height / 2)
            break

        case 'triangleColumn':  // 三角柱
            {
                const x = pos[0], y = pos[1], halfWidth = width / 2, h = Math.sqrt(3) / 2 * width
                const vertices = [
                    [x, y - h / 2],
                    [x - halfWidth, y + h / 2],
                    [x + halfWidth, y + h / 2]
                ]

                const shape = new Shape()
                vertices.forEach((v, index) => {
                    if(!index) shape.moveTo(v[0], v[1])
                    else shape.lineTo(v[0], v[1])
                })

                geometry = new ExtrudeGeometry(shape, {
                    depth: height,
                    bevelEnabled: false
                })
                geometry.translate(0, 0, this.map.cfg.depth)
            }
            break

        case 'hexagonColumn':   // 六角柱
            {
                const x = pos[0], y = pos[1], halfWidth = width / 2, h = Math.sqrt(3) / 2 * halfWidth
                const vertices = [
                    [x - halfWidth, y],
                    [x - halfWidth / 2, y + h],
                    [x + halfWidth / 2, y + h],
                    [x + halfWidth, y],
                    [x + halfWidth / 2, y - h],
                    [x - halfWidth / 2, y - h]
                ]

                const shape = new Shape()
                vertices.forEach((v, index) => {
                    if(!index) shape.moveTo(v[0], v[1])
                    else shape.lineTo(v[0], v[1])
                })

                geometry = new ExtrudeGeometry(shape, {
                    depth: height,
                    bevelEnabled: false
                })
                geometry.translate(0, 0, this.map.cfg.depth)
            }
            break

        case 'squareColumn':	// 四方体
            geometry = new BoxGeometry(width, width, height)
            geometry.translate(0, 0, height / 2)
            break
    }

    const material = new MeshBasicMaterial({ color })
    const mesh = new Mesh(geometry, material)
    if(['cylinder', 'squareColumn'].includes(shape)) {
        mesh.position.set(x, y, this.map.cfg.depth)
    }
    mesh.userData = userData

    return mesh
}
```

`CylinderGeometry`和`BoxGeometry`的原心都默认位于几何中心，为了方便定位笔者这里还调用`geometry`的`translate`和`rotateX`方法将原心移动到几何体的底面中心。

![](./column_basic.gif)

### 添加材质与光照

目前我们的柱体使用的都是`MeshBasicMaterial`材质，因此渲染出来的柱体表面都是纯色的。为了给柱体增添立体感，我们需要为场景中的物体打光并使用能够接受光照计算的材质。

添加一个环境光`AmbientLight`赋予所有物体一个最低的亮度以及一个平行光`DirectionalLight`使得某些面得到高亮：

```typescript
ambientLight: AmbientLight
directLight: DirectionalLight

private createLight() {
    this.ambientLight = new AmbientLight(0xFFFFFF, 1.8)
    this.map.scene.add(this.ambientLight)

    this.directLight = new DirectionalLight(0xFFFFFF, 2)
    this.directLight.position.set(1, -3, 2)
    this.directLight.target.position.set(0, 0, 0)

    this.map.scene.add(this.directLight)
    this.map.scene.add(this.directLight.target)
}
```

材质上选择基于[兰伯特光照模型](https://zh.wikipedia.org/wiki/%E6%9C%97%E4%BC%AF%E4%BD%99%E5%BC%A6%E5%AE%9A%E5%BE%8B)的`MeshLambertMaterial`：

```typescript
const material = new MeshLambertMaterial({ color })
```

![](./column_lambert.gif)



### 渐变色

要实现柱体的渐变色可以有很多种方法，既可以采用在JS层面计算好顶点颜色后传入`attributes.color`来达到渐变效果，也可以通过覆写材质的shader代码来实现更高效的渲染。

考虑到在不同类型的柱体上其几何体`attributes.uv`和`attributes.normal`的差异性，需要分别做额外处理才能实现正确的shader计算，笔者这里采用了更为通用的方法，让材质允许使用顶点颜色数据（`vertexColors`）渲染来实现渐变效果。

#### 顶点着色

在材质对象上设置`vertexColors: true`，使其允许使用顶点颜色数据：

```typescript
const material = new MeshLambertMaterial({ vertexColors: true })
```

通过`geometry.attributes.position`中的`count`和`array`变量可以访问到该几何体的顶点数量及位置。建立`color`缓冲区并遍及所有的顶点，依据顶点坐标中的`z`值与柱体高度`height`计算得出该处的颜色：

```typescript
private createColumn(idx: number, shape: string, material: Material, pos: number[], height: number, userData: any) {
    const { colorGradient, topColor, bottomColor } = this.setting

    /* 
    ......
     */

    if(colorGradient) {
        const colors = [], posLen = geometry.attributes.position.count
        for(let i = 0, pIdx = 2; i < posLen; ++i, pIdx+=3) {
            const step = geometry.attributes.position.array[pIdx] / height  // 计算步进值
            const curColor = interpolate(bottomColor, topColor, step)       // 插值计的颜色
            const colorVector = getRGBAScore(curColor)

            colors.push(colorVector[0], colorVector[1], colorVector[2])
        }

        geometry.attributes.color = new BufferAttribute(new Float32Array(colors), 3)
    }
    
    /* 
    ......
     */
}
```

#### 色彩空间处理

假设我们在上述代码中使用如下的颜色进行测试查看实际的渲染效果：

![](./top_bottom_color.png)
![](no_color_convert.png)

渲染出来后会发现柱体的顶端和底端与对应的`topColor`和`bottomColor`并不匹配。之所以造成这种问题的原因在于：通过`attributes.color`传入的顶点颜色，是直接使用css颜色的r、g、b三个分量除以255归一化到[0-1]区间的。但渲染管线是在线性颜色空间下工作的，而前端中日常使用的CSS颜色、纹理贴图的颜色信息都是使用的sRGB颜色空间。我们向`color`缓冲区注入的颜色数据会被直接用到shader计算中而缺失了颜色空间转换这一步，这才导致了我们看到的渲染颜色不一致。

因此在传入`color`前可以借助第三方库或者ThreeJS内置的`Color.convertSRGBToLinear`转换一次颜色空间：

```typescript
const color = new Color(colorVector[0], colorVector[1], colorVector[2])
color.convertSRGBToLinear()
colors.push(color.r, color.g, color.b)

// or：通过ThreeJS的Color对象完成所有颜色相关的操作，直接new Color(css颜色字符串)构造的颜色对象会自行做处理

// const color = new Color(bottomColor).lerp(new Color(topColor), step)
// colors.push(color.r, color.g, color.b)
```

![](./column_gradient.gif)



## 3D热力图

### 在canvas上绘制2D黑白热力图

首先创建一个Canvas画布。根据渲染的地图坐标范围创建对应宽高的画布后，为了使得热力点的坐标系与画布匹配，调用`translate(width / 2, height / 2)`来将原点移至画布中心：

```typescript
private generateCanvas() {
    const width = this.sideInfo.sizeX, height = this.sideInfo.sizeY
    const canvas = document.createElement('canvas')
    canvas.width = width
    canvas.height = height
    const ctx = canvas.getContext('2d')
    ctx.translate(width / 2, height / 2)
}
```

接着就可以在这个画布上绘制如下黑白渐变的半透明圆形了：

![](./gradient_transparent.png)
```typescript
Draw_Circle_Radisu = 150

/**
 * 根据value值绘制渐变圆圈
 */
private drawCircle(ctx: CanvasRenderingContext2D, x: number, y: number, alpha: number) {
    const grad = ctx.createRadialGradient(x, y, 0, x, y, this.Draw_Circle_Radisu)
    grad.addColorStop(0.0, 'rgba(0,0,0,1)')
    grad.addColorStop(1.0, 'rgba(0,0,0,0)')
    ctx.fillStyle = grad

    ctx.beginPath()
    ctx.arc(x, y, this.Draw_Circle_Radisu, 0, 2 * Math.PI)
    ctx.closePath()

    ctx.globalAlpha = alpha
    ctx.fill()
}
```

绘制时使用的全局透明度由热力点的数值计算而来，将所有热力点绘制到canvas画布后就是下面这样：

```typescript
this.data.forEach((item: RenderData) => {
    const [x, y, value] = item.value
    const alpha = (value - this.sideInfo.minValue) / this.sideInfo.sizeValue

    this.drawCircle(ctx, x, y, alpha)
})
```

![](./gradient_transparent_points.png)

### 转换至彩色热力图

将canvas画布上的所有像素信息通过`getImageData`方法提取出来，其返回的数组中每四个下标表示一个像素的r/g/b/alpha值。根据alpha值计算出每一个像素点所对应色阶中的颜色值，最后覆盖回画布上：

```typescript
type DivisionSetting = {
    pStart: number
    pEnd: number
    color: [r: number, g: number, b: number]
}

/* 根据透明度上色 */
const imageData = ctx.getImageData(0, 0, width, height)
const divisionSetting: DivisionSetting[] = this.setting.divisionSetting

for(let i = 3; i < imageData.data.length; i += 4) {
    const alpha = imageData.data[i], step = alpha / 255

    let idx
    for(let i = 0; i < this.divisionSetting.length; ++i) {
        if(this.divisionSetting[i].pStart <= step && step <= this.divisionSetting[i].pEnd) {
            idx = i
            break
        }
    }

    if(idx === undefined) return
    imageData.data[i - 3] = this.divisionSetting[idx].color[0]
    imageData.data[i - 2] = this.divisionSetting[idx].color[1]
    imageData.data[i - 1] = this.divisionSetting[idx].color[2]
}

ctx.putImageData(imageData, 0, 0)
```

![](./heat_color_points.png)

### 渲染3D效果

创建一个平面几何`PlaneGeometry`并将上面的canvas作为纹理贴图赋予给它，就能得到一张2D的彩色热力图了。而要让热力图呈现出立体效果的关键点就在于自定义shader使得平面上的顶点高度依据纹理贴图的透明度来推算得出。

```typescript
const map = new CanvasTexture(canvas)
const geometry = new PlaneGeometry(this.sideInfo.sizeX, this.sideInfo.sizeY, 500, 500)
const material = new ShaderMaterial({
    vertexShader: VertexShader,
    fragmentShader: FragmentShader,
    transparent: true,
    side: DoubleSide,
    uniforms: {
        map: { value: map },
        uHeight: { value: 10 }	// 最大高度
    }
})

const plane = new Mesh(geometry, material)
plane.position.set(0, 0, this.map.cfg.depth + 0.5)
plane.renderOrder = 1		// 保证渲染顺序不与地图模型上的半透明材质冲突
this.map.scene.add(plane)
```

```typescript
const VertexShader = `
${ShaderChunk.logdepthbuf_pars_vertex}
bool isPerspectiveMatrix(mat4) {
    return true;
}
uniform sampler2D map;
uniform float uHeight;
varying vec2 v_texcoord;
void main(void)
{
    v_texcoord = uv;
    float h = texture2D(map, v_texcoord).a * uHeight;
    gl_Position = projectionMatrix * modelViewMatrix * vec4(position.x, position.y, h, 1.0 );
    ${ShaderChunk.logdepthbuf_vertex}
}
`

const FragmentShader = `
${ShaderChunk.logdepthbuf_pars_fragment}
precision mediump float;
uniform float uOpacity;
uniform sampler2D map;
varying vec2 v_texcoord;

void main (void)
{
    vec4 color = texture2D(map, v_texcoord);
    gl_FragColor.rgb = color.rgb;
    
    gl_FragColor.a = min(color.a * 1.2, 1.0);
    ${ShaderChunk.logdepthbuf_fragment}
}
`
```

为材质中的`uniforms.uHeight`变量添加动画效果使其随时间从0缓慢增长到最大高度，以此观察热力图从平面转换到3D的过程：

```typescript
uniforms: {
    /* ... */
    uHeight: { value: 0 }
}

setInterval(() => {
	material.uniforms.uHeight.value = material.uniforms.uHeight.value + 0.5
}, 100)
```

![](./heat_animation.gif)