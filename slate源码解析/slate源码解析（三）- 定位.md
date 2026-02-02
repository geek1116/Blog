#### 接口定义

能够对于文字、段落乃至任何元素的精准定位 并做出增删改查，都是在开发一款富文本编辑器时一项最基本也是最重要的功能之一。让我们先来看看Slate中对于如何在文档树中定位元素是怎么定义的[[源码]](https://github.com/ianstormtaylor/slate/blob/slate%400.82.0/packages/slate/src/interfaces/location.ts#L12)：

```typescript
/**
 * The `Location` interface is a union of the ways to refer to a specific
 * location in a Slate document: paths, points or ranges.
 *
 * Methods will often accept a `Location` instead of requiring only a `Path`,
 * `Point` or `Range`. This eliminates the need for developers to manage
 * converting between the different interfaces in their own code base.
 */
export type Location = Path | Point | Range
```

`Location`是一个包含了`Path`、`Point`及`Range`的联合类型，代指了Slate中所有关于“定位”的定位，同时也方便了开发。例如在几乎所有的*Transforms*方法中，都可以通过传递`Location`参数来决定*Transforms*方法需要应用到文档中的哪些位置上[[链接]](https://docs.slatejs.org/api/transforms#node-options)。

> All transforms support a parameter `options`. This includes options specific to the transform, and general `NodeOptions` to specify which Nodes in the document that the transform is applied to.
>
> ```typescript
> interface NodeOptions {
>   at?: Location
>   match?: (node: Node, path: Location) => boolean
>   mode?: 'highest' | 'lowest'
>   voids?: boolean
> }
> ```





#### Path

`Path`是三个中最基本的概念，也是唯一一个不可拓展的类型。

```typescript
/**
 * `Path` arrays are a list of indexes that describe a node's exact position in
 * a Slate node tree. Although they are usually relative to the root `Editor`
 * object, they can be relative to any `Node` object.
 */
export type Path = number[]
```

`Path`类型就是一个数组，用来表示Slate文档树中自根节点root到指定node的绝对路径。我们以下边的示例来演示下各个node所代表的路径：

```typescript
const initialValue: Descendant[] = [
    // path: [0]
    {
        type: 'paragraph',
        children: [
            { text: 'This is editable ' },		// path: [0, 0]
            { text: 'rich text!', bold: true }	// path: [0, 1]
        ]
    },
    // path: [1]
    {
        type: 'paragraph',
        children: [
            { text: 'It\' so cool.' }	// path: [1, 0]
        ]
    }
]
```

虽然`Path`所代表的路径通常是以顶层*Editor*作为root节点的，但也会有其他情况，比如由`Node`提供的`get`方法中传入的`Path`参数则代表的是相对路径[[源码]](https://github.com/ianstormtaylor/slate/blob/slate%400.82.0/packages/slate/src/interfaces/node.ts#L339)：

```typescript
/**
 * Get the descendant node referred to by a specific path. If the path is an
 * empty array, it refers to the root node itself.
 */
get(root: Node, path: Path): Node {
    let node = root

    for (let i = 0; i < path.length; i++) {
        const p = path[i]

        if (Text.isText(node) || !node.children[p]) {
            throw new Error(
                `Cannot find a descendant at path [${path}] in node: ${Scrubber.stringify(
                    root
                )}`
            )
        }

        node = node.children[p]
    }

    return node
}
```



#### Point

`Point`是在`Path`的基础上封装而来的概念：

```typescript
/**
 * `Point` objects refer to a specific location in a text node in a Slate
 * document. Its path refers to the location of the node in the tree, and its
 * offset refers to the distance into the node's string of text. Points can
 * only refer to `Text` nodes.
 */
export interface BasePoint {
  path: Path
  offset: number
}

export type Point = ExtendedType<'Point', BasePoint>
```

用于定位单个字符在文档中的位置；先用`Path`定位到字符所属的*Node*，再根据`offset`字段信息精确到字符是在该*Node*的`text`文本中的偏移量。

我们仍然以前面的示例做说明，如果想要将光标位置定位到第一句话"This is editable rich text!"的感叹号之后，其`Point`值为：

```typescript
const initialValue: Descendant[] = [
    // path: [0]
    {
        type: 'paragraph',
        children: [
            { text: 'This is editable ' },		// path: [0, 0]
            { text: 'rich text!', bold: true }	// { path: [0, 1], offset: 10 }
        ]
    },
    // path: [1]
    {
        type: 'paragraph',
        children: [
            { text: 'It\' so cool.' }	// path: [1, 0]
        ]
    }
]
```



#### Range

最后一个`Range`则是再在`Point`基础上延伸封装而来的概念：

```typescript
/**
 * `Range` objects are a set of points that refer to a specific span of a Slate
 * document. They can define a span inside a single node or a can span across
 * multiple nodes.
 */
export interface BaseRange {
  anchor: Point
  focus: Point
}

export type Range = ExtendedType<'Range', BaseRange>
```

它代表的是一段文本的集合；包含有两个`Point`类型的字段`anchor`和`focus`。看到这，应该能发现Slate中`Range`的概念其实与DOM中的*Selection*对象是一样的，`anchor`和`focus`分别对应原生*Selection*中的锚点`anchorNode`和焦点`focusNode`；这正是Slate对于原生*Selection*做的抽象，使之在自身API中更方便地通过光标选区来获取文档树中的内容。

我们在上一篇文章中有提到过的`Editor.selection`是一个`Selections`类型，其本身就是一个`Range`类型，专门用来指定编辑区域中的光标位置[[源码]](https://github.com/ianstormtaylor/slate/blob/slate%400.82.0/packages/slate/src/interfaces/editor.ts#L46)：

```typescript
export type BaseSelection = Range | null

export type Selection = ExtendedType<'Selection', BaseSelection>
```





#### Refs

当我们需要长期追踪某些*Node*时，可以通过获取对应*Node*的`Path`/`Point`/`Range`值并保存下来以达到目的。但这种方式存在的问题是，在Slate文档树经过*insert*、*remove*等操作后，原先的`Path`/`Point`/`Range`可能会产生变动或者直接作废掉。

*Refs*的出现就是为了解决上述问题。

三者的*ref*的定义分别在*slate/src/interfaces/*下的*path-ref.ts*、*point-ref.ts*和*range-ref.ts*文件中：

```typescript
/**
 * `PathRef` objects keep a specific path in a document synced over time as new
 * operations are applied to the editor. You can access their `current` property
 * at any time for the up-to-date path value.
 */
export interface PathRef {
  current: Path | null
  affinity: 'forward' | 'backward' | null
  unref(): Path | null
}

/**
 * `PointRef` objects keep a specific point in a document synced over time as new
 * operations are applied to the editor. You can access their `current` property
 * at any time for the up-to-date point value.
 */
export interface PointRef {
  current: Point | null
  affinity: TextDirection | null
  unref(): Point | null
}

/**
 * `RangeRef` objects keep a specific range in a document synced over time as new
 * operations are applied to the editor. You can access their `current` property
 * at any time for the up-to-date range value.
 */
export interface RangeRef {
  current: Range | null
  affinity: 'forward' | 'backward' | 'outward' | 'inward' | null
  unref(): Range | null
}
```

都包含以下三个字段：

- current：同*React ref*用法一样，用current字段存储最新值
- affinity：当前定位所代表的节点 在文档树变动时如果受到影响的话，所采取的调整策略
- unref：卸载方法；彻底删除当前的ref确保能够被引擎GC掉



另外我们先来看下各种*Refs*在*Slate*存储的方式，跳到*slate/src/utils/weak-maps.ts*中[[源码]](https://github.com/ianstormtaylor/slate/blob/slate%400.82.0/packages/slate/src/utils/weak-maps.ts)：

```typescript
export const PATH_REFS: WeakMap<Editor, Set<PathRef>> = new WeakMap()
export const POINT_REFS: WeakMap<Editor, Set<PointRef>> = new WeakMap()
export const RANGE_REFS: WeakMap<Editor, Set<RangeRef>> = new WeakMap()
```

可以看到*Refs*的存储区在一个*Set*数据结构中的；而对于不同*Editor*下`Set<xxxRef>`的存储则是放在哈希表*WeakMap*中的，使其不会影响到GC（*WeakMap*同Map原理一样，都是ES6之后新出的哈希数据结构，与Map不同点在于其持有的引用算作弱引用）。



生成三种*Ref*以及获取相应*Refs*的方法定义在`EditorInterface`接口上[[源码]](https://github.com/ianstormtaylor/slate/blob/slate%400.82.0/packages/slate/src/interfaces/editor.ts#L279)：

```typescript
export interface EditorInterface {
    // ...
    pathRef: (
    	editor: Editor,
    	path: Path,
    	options?: EditorPathRefOptions
  	) => PathRef
    
    pointRef: (
    	editor: Editor,
    	point: Point,
    	options?: EditorPointRefOptions
  	) => PointRef
    
    rangeRef: (
    	editor: Editor,
    	range: Range,
    	options?: EditorRangeRefOptions
  	) => RangeRef
    
    pathRefs: (editor: Editor) => Set<PathRef>
    
    pointRefs: (editor: Editor) => Set<PointRef>
    
    rangeRefs: (editor: Editor) => Set<RangeRef>
}
```



`Path`、`Point`及`Range`三者的实现逻辑都差不多，下面就以`Path`为例作介绍。

[pathRef](https://github.com/ianstormtaylor/slate/blob/slate%400.82.0/packages/slate/src/interfaces/editor.ts#L1125)：

```typescript
  /**
   * Create a mutable ref for a `Point` object, which will stay in sync as new
   * operations are applied to the editor.
   */
  pointRef(
    editor: Editor,
    point: Point,
    options: EditorPointRefOptions = {}
  ): PointRef {
    const { affinity = 'forward' } = options
    const ref: PointRef = {
      current: point,
      affinity,
      unref() {
        const { current } = ref
        const pointRefs = Editor.pointRefs(editor)
        pointRefs.delete(ref)
        ref.current = null
        return current
      },
    }

    const refs = Editor.pointRefs(editor)
    refs.add(ref)
    return ref
  }
```

实现逻辑非常简单，就是根据传入的参数放入`ref`对象中，并添加卸载方法`unref`。然后通过`pathRefs`拿到对应的*Set*，讲当前的`ref`对象添加进去。`unref`方法中实现的则是相反的操作：通过`pathRefs`拿到对应的*Set*后，将当前`ref`对象移除掉，然后再把`ref.current`的值置空。



[pathRefs](https://github.com/ianstormtaylor/slate/blob/slate%400.82.0/packages/slate/src/interfaces/editor.ts#L1152)：

```typescript
  /**
   * Get the set of currently tracked path refs of the editor.
   */
  pathRefs(editor: Editor): Set<PathRef> {
    let refs = PATH_REFS.get(editor)

    if (!refs) {
      refs = new Set()
      PATH_REFS.set(editor, refs)
    }

    return refs
  }
```

代码非常简短，类似懒加载的方式做*Set*的初始化，然后调用`get`方法获取集合后返回。



#### Ref同步

前一篇文章我们提到过，用于修改内容的单个*Transform*方法会包含有多个*Operation*；*Operation*则是*Slate*中的原子化操作。而在*Slate*文档树更新之后，解决*Ref*同步更新的方式就是：在执行了任意*Operation*之后，对所有的*Ref*根据执行的*Operation*类型做相应的调整。

看到create-editor.ts中的`apply`方法，该方法是所有*Operation*执行的入口[[源码]](https://github.com/ianstormtaylor/slate/blob/slate%400.82.0/packages/slate/src/create-editor.ts#L33)：

```typescript
apply: (op: Operation) => {
    for (const ref of Editor.pathRefs(editor)) {
        PathRef.transform(ref, op)
    }

    for (const ref of Editor.pointRefs(editor)) {
        PointRef.transform(ref, op)
    }

    for (const ref of Editor.rangeRefs(editor)) {
        RangeRef.transform(ref, op)
    }
    
    // ...
}
```

在`apply`方法的最开头就是三组*for of*循环，对所有的*Ref*执行对应的`Ref.transform`方法并传入当前执行的*Operation*。

同样以`Path`为例，看下*path-ref.ts*中的`PathRef.transform`方法[[源码]](https://github.com/ianstormtaylor/slate/blob/slate%400.82.0/packages/slate/src/interfaces/path-ref.ts#L25)：

```typescript
export const PathRef: PathRefInterface = {
  /**
   * Transform the path ref's current value by an operation.
   */
  transform(ref: PathRef, op: Operation): void {
    const { current, affinity } = ref

    if (current == null) {
      return
    }

    const path = Path.transform(current, op, { affinity })
    ref.current = path

    if (path == null) {
      ref.unref()
    }
  }
}
```

将当前*ref*中的数据和*Operation*作为参数，传递给对应的`Path.transform`，返回更新后的*path*值并赋值给`ref.current`。如果`path`为空，则说明当前*ref*指代的位置已经失效，调用卸载方法`unref`。

至于`Path.transform`的细节在本篇就不展开了，我们留到后续的*Transform*篇再统一讲解: )