---
layout: post
title: Apache POI合并单元格解惑
---

Apache POI，是一款用来读写Excel文档的神器。早期，Apache POI只有用Java语言开发的版本，现在，.NET社区已经翻译出C#版本的[NPOI][1]。

虽然POI强大、好用、而且还是免费的，但仍有一些奇怪、费解的地方，比如当合并单元格（POI 3.8）的时候，出现了如下这样奇葩的结果：

![POI][2]

合并单元格是成功，还是失败？POI 3.8没有任何告警、提示，只看上面图中的合并后的效果，`[A4:B5]`区域虽然中间的框线已经合并没了，但还是能够看出是分成上下2块的，有点摸不着头绪。在升级到POI 3.17之后，偶然发现多了个`addMergedRegionUnsafe`方法，此时继续调用`addMergedRegion`方法导出上面的Excel文件，就会**导出失败**，控制台打印出如下错误：

```java
java.lang.IllegalStateException: Cannot add merged region A4:B5 to sheet because it overlaps with an existing merged region (A5:B5).
```

意思是说，需要合并的区域`[A4:B5]`与已合并好的区域`[A5:B5]`冲突了。

知道具体的bug成因，解决起来就easy了。解决方案是，如果需要合并的目标区域内已经有合并好的单元格了，就先把这些合并好的单元格拆掉，再继续合并，完整的代码如下：

```java
/**
 * 合并单元格；
 *
 * @param sheet       工作簿；
 * @param firstRow    起始行；
 * @param lastRow     结束行；
 * @param firstColumn 起始列；
 * @param lastColumn  结束列；
 */
public static void mergeCells(HSSFSheet sheet, int firstRow, int lastRow, int firstColumn, int lastColumn) {
    CellRangeAddress mergedRegion;
    for (int i = sheet.getNumMergedRegions() - 1; i >= 0; i--) {
        mergedRegion = sheet.getMergedRegion(i);
        if (!(firstRow > mergedRegion.getLastRow() || lastRow < mergedRegion.getFirstRow() || firstColumn > mergedRegion.getLastColumn() || lastColumn < mergedRegion.getFirstColumn())) {
            sheet.removeMergedRegion(i);
        }
    }
    CellRangeAddress region = new CellRangeAddress(firstRow, lastRow, firstColumn, lastColumn);
    sheet.addMergedRegion(region);
}
```

[1]: https://github.com/dotnetcore/NPOI
[2]: ../images/2019/12/30/1.png

