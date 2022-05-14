# 常用字体操作

本章描述了一些常见的字体处理操作，并展示了如何使用 Core Text 对它们进行编码。

[Demo](https://github.com/hezhujun/CommonFontOperations)

## 创建字体描述符

**清单 3-1** 从指定 PostScript 字体名称和字体大小创建字体描述符。
```swift
func createFontDescriptor(fromName name: String, size: CGFloat) -> CTFontDescriptor {
    return CTFontDescriptorCreateWithNameAndSize(name as CFString, size)
}
```

**清单 3-2** 从字体系列名称和字体特征创建字体描述符。
```swift
let familyName = "Papyrus"
let symbolicTraits = CTFontSymbolicTraits.condensedTrait
let size: CGFloat = 24.0

let traits: [NSString: Any] = [
    kCTFontSymbolicTrait: symbolicTraits
]

let attributes: [NSString: Any] = [
    kCTFontFamilyNameAttribute: familyName,
    kCTFontSymbolicTrait: traits,
    kCTFontSizeAttribute: size
]

let descriptor = CTFontDescriptorCreateWithAttributes(attributes as CFDictionary)
```

## 从字体描述符创建字体

**清单 3-2** 展示了如何创建字体描述符并使用它来创建字体。当您调用 `CTFontCreateWithFontDescriptor` 时，您通常为 matrix 参数传递 NULL 以指定默认（identity）矩阵。`CTFontCreateWithFontDescriptor` 的大小和矩阵（第二个和第三个）参数会覆盖字体描述符中指定的任何参数，除非它们未指定（大小为 0.0，矩阵为 NULL）。

```swift
let fontAttributes: [NSString: Any] = [
    kCTFontFamilyNameAttribute: "Courier",
    kCTFontStyleNameAttribute: "Bold",
    kCTFontSizeAttribute: 16,
]
let descriptor = CTFontDescriptorCreateWithAttributes(fontAttributes as CFDictionary)
let font = CTFontCreateWithFontDescriptor(descriptor, 0, nil)
```

## 创建相关字体

将现有字体转换为相关或相似字体通常很有用。

**清单 3-4** 显示了如何根据函数调用传递的布尔参数的值使字体变为粗体或非粗体。如果当前字体系列没有请求的样式，则该函数返回 nil。

```swift
var desiredTrait: CTFontSymbolicTraits = .init(rawValue: 0)
let traitMask: CTFontSymbolicTraits = .traitBold

if makeBold {
    desiredTrait = CTFontSymbolicTraits.traitBold
}

return CTFontCreateCopyWithSymbolicTraits(font, 0.0, nil, desiredTrait, traitMask)
```

为 size 参数传入 0.0，为 matrix 参数传入 nil 会保留原始字体的大小。

**清单 3-5** 把字体转成另一个字体系列

```swift
func createFontConvertedToFamily(font: CTFont, family: String) -> CTFont? {
    return CTFontCreateCopyWithFamily(font, 0.0, nil, family as CFString)
}
```

## 序列化字体

**清单 3-6** 展示了如何创建 XML 数据来序列化可嵌入文档中的字体。或者，最好使用 NSArchiver。这只是完成此任务的一种方法，但它保留了以后重新创建确切字体所需的字体中的所有数据。
```swift
let descriptor = CTFontCopyFontDescriptor(font)
let attributes = CTFontDescriptorCopyAttributes(descriptor)

var cfError: CFError! = CFErrorCreate(nil, "com.hezhujun.ios.CommonFontOperations" as CFErrorDomain, 0, nil)
var unmanagedError: Unmanaged<CFError>? = Unmanaged.passUnretained(cfError)
let data: NSData? = withUnsafeMutablePointer(to: &unmanagedError) { error in
    if CFPropertyListIsValid(attributes, .xmlFormat_v1_0) {
        let unmanagedData: Unmanaged<CFData>! = CFPropertyListCreateData(kCFAllocatorDefault, attributes, .xmlFormat_v1_0, .zero, error)
        let data: NSData = unmanagedData.takeRetainedValue() as NSData
        if let xmlString = NSString(data: data as Data, encoding: String.Encoding.utf8.rawValue) {
            print("createFlattenedFontData: xml data \n\(xmlString)")
        } else {
            print("createFlattenedFontData: convert data to string error!")
        }
        return data
    } else {
        return nil
    }
}
```

## 从序列化数据创建字体

**清单 3-7** 展示了如何从展平的 XML 数据创建字体引用。它展示了如何展开字体属性并使用这些属性创建字体。

```swift
var format = CFPropertyListFormat.xmlFormat_v1_0
let font: CTFont? = withUnsafeMutablePointer(to: &format) { format in
    let unmanagedPropertyList: Unmanaged<CFPropertyList>! = CFPropertyListCreateWithData(nil, data, CFPropertyListMutabilityOptions.mutableContainersAndLeaves.rawValue, format, nil)
    let attributes = unmanagedPropertyList.takeRetainedValue() as! CFDictionary
    let descriptor = CTFontDescriptorCreateWithAttributes(attributes)
    let font = CTFontCreateWithFontDescriptor(descriptor, 0, nil)
}
```

## 更改字距

默认情况下启用连字和字距调整。要禁用，请将 `kCTKernAttributeName` 属性设置为 0。

**清单 3-8** 将前几个字符的紧缩大小设置为一个较大的数字。
```swift
CFAttributedStringSetAttribute(attrString, range, kCTKernAttributeName, NSNumber(value: 20))
```

## 获取字符的字形

**清单 3-9** 展示了如何获取具有单一字体的字符串中的字符的字形。大多数情况下，您应该只使用 CTLine 对象来获取此信息，因为一种字体可能无法编码整个字符串。此外，简单的字符到字形的映射不会为复杂的脚本获得正确的外观。如果您尝试为字体显示特定的 Unicode 字符，这种简单的字形映射可能是合适的。

```swift
let count = CFStringGetLength(string as CFString)
var characters = UnsafeMutableBufferPointer<UniChar>.allocate(capacity: count)
var glyhps = UnsafeMutableBufferPointer<CGGlyph>.allocate(capacity: count)

CFStringGetCharacters(string as CFString, CFRangeMake(0, count), characters.baseAddress)
CTFontGetGlyphsForCharacters(font, characters.baseAddress!, glyhps.baseAddress!, count)
```

## 使用导入字体

使用导入字体有两种方式，第一种是把字体文件注册到系统，通过名字来创建字体描述符，第二种是加载字体文件创建字体描述符。

**清单 3-10** 注册字体
```swift
// scrop 表示字体注册后的生命周期
CTFontManagerRegisterFontsForURL(url as CFURL, .process, nil)

// 通过名字创建字体对象
let name: String = "..."
let font = UIFont(name: name, size: 15)
```

**清单 3-11** 注销字体
```swift
CTFontManagerUnregisterFontsForURL(url as CFURL, .process, nil)
```

**清单 3-12** 加载字体文件创建字体描述符

```swift
// 通过字体文件 url 创建 CTFontDescriptor
guard let fontDescriptors = CTFontManagerCreateFontDescriptorsFromURL(url as CFURL) as? [CTFontDescriptor] else {
    return
}
// 打印字体信息
fontDescriptors.forEach { descriptor in
    guard let attributes = CTFontDescriptorCopyAttributes(descriptor) as? [NSString: Any] else {
        return
    }
    for (key, value) in attributes {
        print("\(key): \(value)")
    }
}

guard let descriptor = fontDescriptors.first else { return }
guard let attributes = CTFontDescriptorCopyAttributes(descriptor) as? [NSString: Any],
        let name = attributes[kCTFontNameAttribute] as? NSString else {
    return
}
// 通过 fontDescriptor 创建 CTFont
let font = CTFontCreateWithFontDescriptor(descriptor, 15, nil)
```