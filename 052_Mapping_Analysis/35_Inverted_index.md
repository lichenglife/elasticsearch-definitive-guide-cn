## 倒排索引

Elasticsearch使用一种叫做**倒排索引(inverted index)**的结构来做快速的全文搜索。倒排索引由在文档中出现的唯一的单词列表，以及对于每个单词在文档中的位置组成。

例如，我们有两个文档，每个文档`content`字段包含：

1. The quick brown fox jumped over the lazy dog
2. Quick brown foxes leap over lazy dogs in summer

为了创建倒排索引，我们首先切分每个文档的`content`字段为单独的单词（我们把它们叫做**词(terms)**或者**表征(tokens)**）（译者注：关于`terms`和`tokens`的翻译比较生硬，只需知道语句分词后的个体叫做这两个。），把所有的唯一词放入列表并排序，结果是这个样子的：

    Term      Doc_1  Doc_2
    -------------------------
    Quick   |       |  X
    The     |   X   |
    brown   |   X   |  X
    dog     |   X   |
    dogs    |       |  X
    fox     |   X   |
    foxes   |       |  X
    in      |       |  X
    jumped  |   X   |
    lazy    |   X   |  X
    leap    |       |  X
    over    |   X   |  X
    quick   |   X   |
    summer  |       |  X
    the     |   X   |
    ------------------------

现在，如果我们想搜索`"quick brown"`，我们只需要找到每个词在哪个文档中出现既可：


    Term      Doc_1  Doc_2
    -------------------------
    brown   |   X   |  X
    quick   |   X   |
    ------------------------
    Total   |   2   |  1

两个文档都匹配，但是第一个比第二个有更多的匹配项。
如果我们加入简单的**相似度算法(similarity algorithm)**，计算匹配单词的数目，这样我们就可以说第一个文档比第二个匹配度更高——对于我们的查询具有更多相关性。

但是在我们的倒排索引中还有些问题：

1. `"Quick"`和`"quick"`被认为是不同的单词，但是用户可能认为它们是相同的。
2. `"fox"`和`"foxes"`很相似，就像`"dog"`和`"dogs"`——它们都是同根词。
3. `"jumped"`和`"leap"`不是同根词，但意思相似——它们是同义词。

上面的索引中，搜索`"+Quick +fox"`不会匹配任何文档（记住，前缀`+`表示单词必须匹配到）。只有`"Quick"`和`"fox"`都在同一文档中才可以匹配查询，但是第一个文档包含`"quick fox"`且第二个文档包含`"Quick foxes"`。（译者注：这段真罗嗦，说白了就是单复数和同义词没法匹配）

用户可以合理的希望两个文档都能匹配查询，我们也可以做的更好。

如果我们将词为统一为标准格式，这样就可以找到不是确切匹配查询，但是足以相似从而可以关联的文档。例如：

1. `"Quick"`可以转为小写成为`"quick"`。
2. `"foxes"`可以被转为根形式`""fox`。同理`"dogs"`可以被转为`"dog"`。
3. `"jumped"`和`"leap"`同义就可以只索引为单个词`"jump"`

现在的索引：

    Term      Doc_1  Doc_2
    -------------------------
    brown   |   X   |  X
    dog     |   X   |  X
    fox     |   X   |  X
    in      |       |  X
    jump    |   X   |  X
    lazy    |   X   |  X
    over    |   X   |  X
    quick   |   X   |  X
    summer  |       |  X
    the     |   X   |  X
    ------------------------

但我们还未成功。我们的搜索`"+Quick +fox"`*依旧*失败，因为`"Quick"`的确切值已经不在索引里，不过，如果我们使用相同的标准化规则处理查询字符串的`content`字段，查询将变成`"+quick +fox"`，这样就可以匹配到两个文档。

>### IMPORTANT
>这很重要。你只可以找到确实存在于索引中的词，所以**索引文本和查询字符串都要标准化为相同的形式**。

这个表征化和标准化的过程叫做**分词(analysis)**，这个在下节中我们讨论。