# 流程
## 创建lintmodule
新建一个java module，添加lint相关依赖
```
dependencies {

    testImplementation 'junit:junit:4.+'
    // Lint
    compileOnly "com.android.tools.lint:lint-api:26.3.2"
    compileOnly "com.android.tools.lint:lint-checks:26.3.2"
}
```
然后在需要检查的module的build.gradle中，将lint module以依赖的方式添加进去。
```
dependencies {
    lintChecks project(':lint')
}
```
## 注册检查表
lint module中实现一个继承自IssueRegistry的类，并实现`getIssues()`方法， 同时重写`getApi()`方法
```
class ResIssueRegistry : IssueRegistry() {
    override val issues: List<Issue>
        get() = listOf(XmlDetector.ISSUE)
    override val api: Int
        get() = CURRENT_API

}
```
其中getIssues返回的是需要进行lint检查的问题列表，它的构造见下一节
## 构建ISSUE
上面提到的问题列表是ISSUE对象的列表，代表着需要进行检查的问题，它的构造如下:
```
val ISSUE = Issue.create(
            "Use Other module resources", "不要在插件module引用其他module资源",
            "qigsaw在8.x设备，引用其他module资源进程重启会崩溃", Category.CORRECTNESS, 10,
            Severity.ERROR, IMPLEMENTATION
        )
```
其中create方法定义如下：
```

        /**
         * Creates a new issue. The description strings can use some simple markup;
         * see the [TextFormat.RAW] documentation
         * for details.
         *
         * @param id the fixed id of the issue
         * @param briefDescription short summary (typically 5-6 words or less), typically
         * describing the **problem** rather than the **fix**
         * (e.g. "Missing minSdkVersion")
         * @param explanation a full explanation of the issue, with suggestions for
         * how to fix it
         * @param category the associated category, if any
         * @param priority the priority, a number from 1 to 10 with 10 being most
         * important/severe
         * @param severity the default severity of the issue
         * @param implementation the default implementation for this issue
         * @return a new [Issue]
         */
        @JvmStatic
        fun create(
            id: String,
            briefDescription: String,
            explanation: String,
            category: Category,
            priority: Int,
            severity: Severity,
            implementation: Implementation
        ): Issue {
            val platforms = computePlatforms(null, implementation)
            return Issue(
                id, briefDescription, explanation, category, priority,
                severity, platforms, null, implementation
            )
        }
```
参数的定义如下：
1. id：唯一的 id，简要表达当前问题。
2. briefDescription：简单描述当前问题。
3. explanation：详细解释当前问题和修复建议。
4. category：问题类别，在 Android 中主要有如下六大类：
   - SECURITY：安全性。例如在 AndroidManifest.xml 中没有配置相关权限等。
   -  USABILITY：易用性。例如重复图标，一些黄色警告等。
   - PERFORMANCE：性能。例如内存泄漏，xml 结构冗余等。
   - CORRECTNESS：正确性。例如超版本调用 API，设置不正确的属性值等。
   - A11Y：无障碍（Accessibility）。例如单词拼写错误等。
   - I18N：国际化（Internationalization）。例如字符串缺少翻译等。
5. priority：优先级，从 1 到 10，10 最重要。
6. severity：严重程度，包括 FATAL、ERROR、WARNING、INFORMATIONAL和 IGNORE。
7. implementation：Issue 和哪个 Detector 绑定，以及声明检查的范围。
> 这里有必要提一下，gradle打包工程时，默认会执行lintVitalRelease或lintVitalDebug，它们会对fatal级别的问题进行检查，所以需要编译时抛出错误，需要将ISSUE定义为fatal

七个参数中，前面几个参数可以根据开发者的想法进行定义，值得研究的是最后一个参数
## 构建implementation
Implementation的构造方法如下:
```
    /**
     * Creates a new implementation for analyzing one or more issues
     *
     * @param detectorClass the class of the detector to find this issue
     * @param scope the scope of files required to analyze this issue
     */
    @SuppressWarnings("unchecked")
    public Implementation(
            @NonNull Class<? extends Detector> detectorClass, @NonNull EnumSet<Scope> scope) {
        this(detectorClass, scope, EMPTY);
    }
```
参数定义如下：
1. Detector：用于扫描代码发现问题
2. Scopes：指定分析的范围，常见的范围包括以下几种
   - 资源文件
   - Java 源文件
   - Class 文件
   - Proguard 配置文件
   - Manifest 文件
   - 其他
## Scanner
上述Detector的构建需要实现scanner接口，不同接口提供了扫描不同的文件或文件夹方法，常见的有:
- UastScanner：扫描 Java 文件和 Kotlin 文件
- ClassScanner：扫描 Class 文件
- XmlScanner：扫描 XML 文件
- ResourceFolderScanner：扫描资源文件夹
- BinaryResourceScanner：扫描二进制资源文件
- OtherFileScanner：扫描其他文件
- GradleScanner：扫描 Gradle 脚本
  
这里我用到了XmlScanner来逐一分析xml文件，来检查该xml中是否有引用其他module中的文件
# 实现lint检查xml是否引用其他module资源文件
思路很简单，首先构造待检测module资源文件名集合，然后遍历本module的xml文件，查找引用的资源名是否未出现在集合中，未出现则表示存在引用，然后抛出异常打断编译。
## 构建资源文件名集合
Detertor接口提供了很多插入点，方便开发者可以在lint检查分析前
进行预处理。预先构建文件名集合就是利用这个来实现的，下面介绍实现的过程。
### 遍历res文件夹
我们通过`beforeCheckRootProject(context: Context)`方法，可以拿到当前module的context对象，这时lint检查还未开始。Context中包含file对象，指向module根目录，于是我们就可以从根目录找到res文件夹，遍历文件名后得到res列表
```
override fun beforeCheckRootProject(context: Context) {
        super.beforeCheckRootProject(context)
        val dir = context.file
        println("*****${dir.name}*****")
        val resDir = dir.absolutePath + "/src/main/res"
        val file = File(resDir)
        val valueFile  = File("$resDir/values")
        val fileList = mutableListOf<String>()
        listFiles(file, fileList)
        println("**********************values*********************************")
        listValuesFiles(valueFile, fileList)
        resFileList = fileList
        println("**********************************************************")
    }

...

/**
     * 列出文件清单,以一个数组形式返回，
     *
     * @param filePath 磁盘文件路径
     * @param fileArr  此参数需要传一个 MutableList<>()进入方法体,在方法体创建一个对象数组，子目录的文件存放不了进数组进行返回
     */
    fun listFiles(filePath: File, fileArr: MutableList<String>) {
        val files = filePath.listFiles()
        files.forEach { file ->
            if (file.isDirectory) {
                if (file.name != "values")
                    listFiles(file, fileArr)
            } else {
                fileArr.add(file.nameWithoutExtension)
            }
        }
    }
```
这里有个小插曲，values文件夹下的内容，需要进入xml文件内才能获取，包括具体的string\dimen\color等资源的名称
### 获取values目录内容
这个过程也不难，通过`java document api`来依次读取values目录下的xml文件，并对记录包含name属性的属性值，该值即对应的资源引用名。例如：` <color name="ugc_tools_green">#35cc67</color>`
```
private fun listValuesFiles(valueFile: File, fileList: MutableList<String>) {
        valueFile.listFiles().forEach {
            val document = try {
                DocumentBuilderFactory.newInstance().newDocumentBuilder().parse(it)
            } catch (e: Exception) {
                null
            }
            resTraversal(document, fileList)
        }
    }

    private fun resTraversal(document: Node?, fileList: MutableList<String>) {
        document?.let {
            processNode(it, fileList)
            if (it.hasChildNodes()) {
                for (i in 0 .. it.childNodes.length) {
                    resTraversal(it.childNodes.item(i), fileList)
                }
            }
        }
    }

    private fun processNode(node: Node, fileList: MutableList<String>) {
        node.attributes?.getNamedItem("name")?.let {
            fileList.add(it.nodeValue)
        }
    }
```
至此，我们得到了待监测module中可被引用的资源文件名称列表，接下来就是分析xml，查看是否有引用本module外的资源
## 使用scanner扫描待监测xml文件
使用scanner时，需要实现`getApplicableAttributes()`方法，决定待监测的文件类型
```
override fun getApplicableAttributes(): Collection<String>? {
        return XmlScannerConstants.ALL
    }
```
这里表示所有的资源文件类型，也可以选择自己需要的返回，例如` return Arrays.asList(SdkConstants.TAG_ACTIVITY, SdkConstants.TAG_STYLE);`

然后实现`visitAttribute`方法，该方法会遍历每个xml的每个属性（注意，是属性不是节点，节点对应的方法是`visitElement`）。

## 处理属性值
### 识别资源类型
我们需要处理的，只包括资源类型的引用，所以需要判断属性是否为资源类型引用。先对属性值做处理，分离出`type`与`name`
```
 override fun visitAttribute(context: XmlContext, attribute: Attr) {
        super.visitAttribute(context, attribute)
        val content = attribute.value
        if (!content.startsWith("@")) {
            return
        }
        // 以@开头为引用，后面/的前一半为类型，后一半为名称
        val contentList = content.replace("@", "").split("/")
        if (resourceTypes.contains(contentList[0])) {
            checkResourceInOtherModule(contentList[1])
        }
    }
```
### 判断类型有效性
前面提到，我们需要判断属性是否为资源类型引用，所以需要明确哪些是资源类型。下面是Android官网提到的资源类型种类：
```
private val resourceTypes = listOf("anim", "drawable", "color", "xml", "layout", "menu", "string", "style", "font", "array", "plurals")
```
对于资源类型引用，最后判断是否在前面的出的资源列表中
```
private fun checkResourceInOtherModule(fileName: String) {
        if (!resFileList.contains(fileName)) {
            println(fileName)
        }
    }
```
# 总结
lint检查的使用比较简单，可以实现的功能却很多，需要开发者根据这些基础能力，发挥自己的想象力，实现对应的需求。
