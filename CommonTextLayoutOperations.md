# 常用文本布局操作

本章描述了一些常见的文本布局操作，并展示了如何使用 Core Text 执行它们。

[Demo](https://github.com/hezhujun/CommonTextLayoutOperations)

## 布置段落

排版中最常见的操作之一是在任意大小的矩形区域内布置多行段落。Core Text 使这个操作变得简单，只需要几行 Core Text 特定的代码。要对段落进行布局，您需要一个要绘制的图形上下文、一个提供文本布局区域的矩形路径以及一个属性字符串。此示例中的大部分代码都是创建和初始化上下文、路径和字符串所必需的。完成之后，Core Text 只需要三行代码来进行布局。

**清单 2-1** 显示了段落的布局
```swift
override func draw(_ rect: CGRect) {
    guard let context = UIGraphicsGetCurrentContext() else { return }
    context.translateBy(x: 0, y: rect.size.height)
    context.scaleBy(x: 1.0, y: -1.0)
    context.textMatrix = CGAffineTransform.identity
    let path = CGMutablePath()
    path.addRect(rect)
    let textString = "Hello, World! I know nothing in the world that has as much power as a word. Sometimes I write one, and I look at it, until it begins to shine."
    let attrString = NSMutableAttributedString(string: textString, attributes: [.font: UIFont.systemFont(ofSize: 15, weight: .regular), .backgroundColor: UIColor.white.cgColor])
    attrString.setAttributes([.foregroundColor: UIColor.red.withAlphaComponent(0.8).cgColor], range: NSRange(location: 0, length: 12))
    
    // 生成 framesetter
    let framesetter = CTFramesetterCreateWithAttributedString(attrString)
    // 生成一帧
    let frame = CTFramesetterCreateFrame(framesetter, CFRange(location: 0, length: 0), path, nil)
    // 绘制
    CTFrameDraw(frame, context)
}
```

## 简单文本标签

另一种常见的排版操作是绘制单行文本以用作用户界面元素的标签。在 Core Text 中，这只需要两行代码：第一行使用 CFAttributedString 创建 line 对象，另一行将 line 绘制到图形上下文中。

**清单 2-2** 展示了如何在 UIView 或 NSView 子类的 drawRect: 方法中完成此操作。
```swift
override func draw(_ rect: CGRect) {
    guard let context = UIGraphicsGetCurrentContext() else { return }
    context.translateBy(x: 0, y: rect.size.height)
    context.scaleBy(x: 1.0, y: -1.0)
    context.textMatrix = CGAffineTransform.identity
    let string = "Hello, World!"
    let font = UIFont.systemFont(ofSize: 15, weight: .regular)
    let attrString = NSAttributedString(string: string, attributes: [.font: font, .foregroundColor: UIColor.white.cgColor])
    
    let line = CTLineCreateWithAttributedString(attrString)
    context.textPosition = CGPoint(x: 0, y: 0)
    CTLineDraw(line, context)
}
```

## 列式布局

在多列中布局文本是另一种常见的排版操作。严格来说，Core Text 本身一次只布局一列，并不计算列的大小或位置。您在调用 Core Text 之前执行这些操作以在您计算的路径区域内布置文本。在此示例中，Core Text 除了在每列中布置文本外，还为每列提供文本字符串中的子范围。

**清单 2-3** 中的 `createColumnsWithColumnCount:` 方法接受要绘制的列数作为参数，并返回一个路径数组，每列一个路径。
```swift
func createColumns(with columnCount: Int, rect: CGRect) -> [CGPath] {
    var columnRects = [CGRect]()
    let columnWith = rect.width / CGFloat(columnCount)
    for i in 0..<columnCount {
        let x = rect.minX + columnWith * CGFloat(i)
        let y = rect.minY
        let width = min(columnWith, rect.width - x)
        let height = rect.height
        columnRects.append(CGRect(x: x, y: y, width: width, height: height))
    }
    // 增加内边距
    for i in 0..<columnRects.count {
        columnRects[i] = columnRects[i].insetBy(dx: 8, dy: 15)
    }
    
    let paths = columnRects.map { rect in
        CGPath(rect: rect, transform: nil)
    }
    
    return paths
}
```

**清单 2-4** 包含 `drawRect:` 方法的实现，该方法调用本地 `createColumnsWithColumnCount` 方法。
```swift
override func draw(_ rect: CGRect) {
    guard let context = UIGraphicsGetCurrentContext() else { return }
    context.translateBy(x: 0, y: rect.size.height)
    context.scaleBy(x: 1.0, y: -1.0)
    context.textMatrix = CGAffineTransform.identity
    
    let textString = "Hello, World! I know nothing in the world that has as much power as a word. Sometimes I write one, and I look at it, until it begins to shine."
    let attrString = NSMutableAttributedString(
        string: textString,
        attributes: [
            .font: UIFont.systemFont(ofSize: 15, weight: .regular),
            .foregroundColor: UIColor.white.cgColor
        ]
    )
    let framesetter = CTFramesetterCreateWithAttributedString(attrString)
    let columnPaths = createColumns(with: 3, rect: rect)
    let pathCount = columnPaths.count
    var startIndex: CFIndex = 0
    for column in 0..<pathCount {
        let frame = CTFramesetterCreateFrame(framesetter, CFRange(location: startIndex, length: 0), columnPaths[column], nil)
        CTFrameDraw(frame, context)
        
        let frameRange = CTFrameGetVisibleStringRange(frame)
        startIndex += frameRange.length
    }
}
```

## 手动换行

在 Core Text 中，您通常不需要手动换行，除非您有特殊的断字处理或类似要求。框架设置器自动执行换行。或者，Core Text 使您能够准确指定每行文本的中断位置。

**清单 2-5** 展示了如何创建排版器，即框架设置器使用的对象，并直接使用排版器找到合适的换行符并手动创建排版行。此示例还显示了如何在绘制之前使行居中。
```swift
guard let context = UIGraphicsGetCurrentContext() else { return }
context.translateBy(x: 0, y: rect.size.height)
context.scaleBy(x: 1.0, y: -1.0)
context.textMatrix = CGAffineTransform.identity

let textString = "Hello, World! I know nothing in the world that has as much power as a word. Sometimes I write one, and I look at it, until it begins to shine."
let attrString = NSMutableAttributedString(
    string: textString,
    attributes: [
        .font: UIFont.systemFont(ofSize: 19, weight: .regular),
        .foregroundColor: UIColor.white.cgColor
    ]
)

let typesetter = CTTypesetterCreateWithAttributedString(attrString)
var start: CFIndex = 0;
// 给定展示长度，找到换行的位置
let count = CTTypesetterSuggestLineBreak(typesetter, start, rect.width)
let line = CTTypesetterCreateLine(typesetter, CFRange(location: start, length: count))

// 居中的偏移量
let flush: CGFloat = 0.5
let penOffset = CTLineGetPenOffsetForFlush(line, flush, rect.width)
context.textPosition = CGPoint(x: context.textPosition.x + penOffset, y: context.textPosition.y)
CTLineDraw(line, context)

start += count
```

## 应用段落样式

**清单 2-6** 实现了一个将段落样式应用于属性字符串的函数。该函数接受字体名称、点大小和行距作为参数，该行距增加或减少文本行之间的空间量。

```swift
func applyParaSyle(fontName: String, pointSize: CGFloat, plainText: String, lineSpaceInc: CGFloat) throws -> NSAttributedString {
    let font = UIFont(name: fontName, size: pointSize)
    if font == nil {
        throw ViewError.FontNotFound
    }
    var lineSpacing = (CTFontGetLeading(font!) + lineSpaceInc) * 2
    
    var setting = CTParagraphStyleSetting(spec: .lineSpacingAdjustment, valueSize: MemoryLayout<CGFloat>.size, value: &lineSpacing)
    let paragraphStyle = CTParagraphStyleCreate(&setting, 1)
    
    let attrString = NSAttributedString(string: plainText, attributes: [.font: font!, .paragraphStyle: paragraphStyle, .foregroundColor: UIColor.white.cgColor])
    return attrString
}
```

**清单 2-7** 调用2-6中的代码，使用 applyParaStyle 函数创建一个具有给定段落属性的属性字符串，然后创建一个框架设置器和框架，并绘制框架。

```swift
guard let context = UIGraphicsGetCurrentContext() else { return }
    context.translateBy(x: 0, y: rect.size.height)
    context.scaleBy(x: 1.0, y: -1.0)
    context.textMatrix = CGAffineTransform.identity
    
    let fontName = "Didot Italic"
    let pointSize: CGFloat = 24
    let string = "Hello, World! I know nothing in the world that has as much power as a word. Sometimes I write one, and I look at it, until it begins to shine."
    do {
        let attrString = try applyParaSyle(fontName: fontName, pointSize: pointSize, plainText: string, lineSpaceInc: 10)
        let framesetter = CTFramesetterCreateWithAttributedString(attrString)
        let path = CGPath(rect: rect, transform: nil)
        let frame = CTFramesetterCreateFrame(framesetter, CFRange(location: 0, length: 0), path, nil)
        CTFrameDraw(frame, context)
    } catch {
        print(error)
    }
```

## 在非矩形区域中显示文本

在非矩形区域中显示文本的难点在于描述非矩形路径。
**清单 2-8** 中的 `AddSquashedDonutPath` 函数返回一个甜甜圈形状的路径。一旦你有了路径，只需调用常用的 Core Text 函数来应用属性和绘制。

```swift
var path: CGMutablePath {
    let path = CGMutablePath()
    var bounds = self.bounds
    bounds = bounds.insetBy(dx: 10, dy: 10)
    addSquashedDonutPath(path: path, rect: bounds)
    return path
}

func addSquashedDonutPath(path: CGMutablePath, transform: CGAffineTransform = .identity, rect: CGRect) {
    let width = rect.width
    let height = rect.height
    
    let radiusH = width / 3.0
    let radiusV = height / 3.0
    
    path.move(to: CGPoint(x: rect.origin.x, y: rect.origin.y + height - radiusV), transform: transform)
    path.addQuadCurve(to: CGPoint(x: rect.origin.x, y: rect.origin.y + height), control: CGPoint(x: rect.origin.x + radiusH, y: rect.origin.y + height), transform: transform)
    path.addLine(to: CGPoint(x: rect.origin.x + width - radiusH, y: rect.origin.y + height), transform: transform)
    path.addQuadCurve(to: CGPoint(x: rect.origin.x + width, y: rect.origin.y + height), control: CGPoint(x: rect.origin.x + width, y: rect.origin.y + height - radiusV), transform: transform)
    path.addLine(to: CGPoint(x: rect.origin.x + width, y: rect.origin.y + radiusV), transform: transform)
    path.addQuadCurve(to: CGPoint(x: rect.origin.x + width, y: rect.origin.y), control: CGPoint(x: rect.origin.x + width - radiusH, y: rect.origin.y), transform: transform)
    path.addLine(to: CGPoint(x: rect.origin.x + radiusH, y: rect.origin.y), transform: transform)
    path.addQuadCurve(to: CGPoint(x: rect.origin.x, y: rect.origin.y), control: CGPoint(x: rect.origin.x, y: rect.origin.y + radiusV), transform: transform)
    path.closeSubpath()
    path.addEllipse(in: CGRect(x: rect.origin.x + width / 2 - width / 5, y: rect.origin.y + height / 2 - height / 5, width: width / 5 * 2, height: height / 5 * 2), transform: transform)
}

override func draw(_ rect: CGRect) {
    guard let context = UIGraphicsGetCurrentContext() else { return }
    context.translateBy(x: 0, y: rect.size.height)
    context.scaleBy(x: 1.0, y: -1.0)
    context.textMatrix = CGAffineTransform.identity
    
    let textString = "Hello, World! I know nothing in the world that has as much power as a word. Sometimes I write one, and I look at it, until it begins to shine."
    let attrString = NSMutableAttributedString(
        string: textString,
        attributes: [
            .font: UIFont.systemFont(ofSize: 15, weight: .regular),
            .foregroundColor: UIColor.blue.cgColor
        ]
    )
    let framesetter = CTFramesetterCreateWithAttributedString(attrString)
    
    var startIndex = 0
    
    context.setFillColor(UIColor.yellow.cgColor)
    context.addPath(path)
    context.fillPath()
    
    let frame = CTFramesetterCreateFrame(framesetter, CFRange(location: startIndex, length: 0), path, nil)
    CTFrameDraw(frame, context)
}
```