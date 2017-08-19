title: 檔案格式(File format)
author: Terry Chen
date: 2017-08-19
tags:
 - file format 
---


我們知道電腦是0和1所組成的世界，不管是軟體程式、影片、文件、圖片等等都是用0和1來組成的。例如用01000001來代表A，用01100011來代表c，而用111111110000000000000000(分別代表紅、綠、藍三原色，分別為8個bits)來代表紅色。那麼電腦在運作時，要如何區分這些差別呢？這就要靠制定檔案格式來區分，根據[File format - Wikipedia](https://en.wikipedia.org/wiki/File_format)中所記錄，有三種類型用來辨識檔案的格式，分別是`Filename extension(副檔名)`、`Internal metadata`和`External metadata`。

不同的作業系統採用不同的方式，像是Windows是以副檔名的方式來判斷開啟該檔案所需要的軟體，在沒有副檔名的情況之下或副檔名沒有關聯到預設程式的話，Windows會要求你"選擇您想要用來開啟這個檔案的程式"。External metadata是將該檔案格式的資訊存放file system中，而不是記錄在檔案本身，故檔案在不同作業統中交換時，造成處理上的困難。Internal metadata是本文想要了解的主題，相對於External metadata，它將檔案格式的資訊直接存放在檔案本身，其中一種方式是在檔案的開頭加入數個bytes來代表檔案類型，這些bytes稱為Magic number。例如JPG檔案的Magic number為`ff d8 ff db`四個bytes所組成，ISO檔案則是`43 44 30 30 31`，而bz2壓縮檔則是三個bytes所成(`42 5a 68`)。想要知道其他檔案的Magic number，請查閱[List of file signatures - Wikipedia](https://en.wikipedia.org/wiki/List_of_file_signatures)或是[File Signatures - GaryKessler.net](http://www.garykessler.net/library/file_sigs.html)。除了Magic number之外，不同的檔案類型也各自定義其他的metadata，以提供程式獲取該檔案的資訊。像是JPG檔案在File Header中提供該圖檔的解釋度大小，ELF檔(Executable and Linkable Format)提供該執行檔能被何種作業系統執行的資訊等等，本文在此並不對每種檔案類型一一介紹。

### xxd

以下在Linux環境使用**`xxd`**工具以16進制的方式來檢視檔案(也可以搭配`-b`選項以2進制呈現檔案內容)，搭配`-g`選項設定以1個byte各自列印為一行，以方便說明(預設為2 bytes當成一行)。就來看看一個test.jpg的檔案前160 bytes內容:

```
root@b88a81659790:~# xxd -g 1 test.jpg | head
00000000: ff d8 ff db 00 84 00 08 06 06 07 06 05 08 07 07  ................
00000010: 07 09 09 08 0a 0c 14 0d 0c 0b 0b 0c 19 12 13 0f  ................
00000020: 14 1d 1a 1f 1e 1d 1a 1c 1c 20 24 2e 27 20 22 2c  ......... $.' ",
00000030: 23 1c 1c 28 37 29 2c 30 31 34 34 34 1f 27 39 3d  #..(7),01444.'9=
00000040: 38 32 3c 2e 33 34 32 01 09 09 09 0c 0b 0c 18 0d  82<.342.........
00000050: 0d 18 32 21 1c 21 32 32 32 32 32 32 32 32 32 32  ..2!.!2222222222
00000060: 32 32 32 32 32 32 32 32 32 32 32 32 32 32 32 32  2222222222222222
00000070: 32 32 32 32 32 32 32 32 32 32 32 32 32 32 32 32  2222222222222222
00000080: 32 32 32 32 32 32 32 32 ff c0 00 11 08 01 cc 01  22222222........
00000090: cc 03 01 22 00 02 11 01 03 11 01 ff c4 01 a2 00  ..."............
```

第一行(column)代表該列(row)第一個byte的編號，此編號為16進制，接著的16行放的是檔案內容。例如00000000代表`ff`為編號0(第0個)byte，接著d8為編號1(第1個)byte，第二列00000010代表07為編號10(第16個)byte。最後一行將該列的檔案內容的值用[ASCII](http://ascii.cl/)(範圍0~127)印出，ff、d8、07在ASCII沒有對映到任何的文字或符號，xxd就用**`.`**來替代，而在編號30的byte `23`在ASCII中對映到的是**`#`**符號，就印出#符號。在本例test.jpg檔案開頭的`ff` `d8` `ff` `db`這四個byte就代表它是JPG格式的圖形檔案。


### 文字檔

值得一提的是純文字檔並沒有Magic number([註1](#note1))，它是將文字的編碼直接存入檔案。以一個純英文的文字檔案(UTF-8編碼)來說，會看到的是：

```
00000000: 48 65 6c 6c 6f 20 57 6f 72 6c 64 21 0a           Hello World!.
```

會發現最後一行的內容恰好為用文字編輯器所看到的內容是一樣的，這是因為UTF-8在設計上相容於ASCII。句尾的部份多了`.`(0a)這個換行符號，用以表示該行的結束。而以一個中文的文字檔來說(其內容只存二個字`中文`，一樣用UTF-8編碼)，會看的是：

```
00000000: e4 b8 ad e6 96 87 0a                             .......
```

一樣會有`0a`這個換行符號，而`e4 b8 ad` 3 bytes為`中`的[UTF-8 16進制編碼](http://www.fileformat.info/info/unicode/char/4e2d/index.htm)，`e6 96 87`則是`文`。兩個例子顯示了文字檔本身沒有特別的檔案格式，單純的只存入文字編碼。但是，電腦的發展在每個地區的時間點不同，各自陸續創造出不同的編碼方式。在沒有Magic number用以指出該檔案的文字編碼情況之下，依靠程式自動判斷出其編碼方式是困難的，也經常發生打開文字檔卻出現亂碼的現象。這邊引用Python `chardet` package中的[Frequently asked questions](https://chardet.readthedocs.io/en/latest/faq.html#isnt-that-impossible)來說明其困難的原因：

> In general, yes. However, some encodings are optimized for specific languages, and languages are not random. Some character sequences pop up all the time, while other sequences make no sense. A person fluent in English who opens a newspaper and finds “txzqJv 2!dasd0a QqdKjvz” will instantly recognize that that isn’t English (even though it is composed entirely of English letters). By studying lots of “typical” text, a computer algorithm can simulate this kind of fluency and make an educated guess about a text’s language.

在UTF-8大量使用來取代各種編碼碼的情況之下，未來出現亂碼的機會可望減少。下面引用[UTF-8 - Wikipedia](https://en.wikipedia.org/wiki/UTF-8)文章中的圖片，它展示了在Google 2001~2012的記錄中所有網頁所使用的編碼方式，UTF-8的使用已成長接近70%。![](https://upload.wikimedia.org/wikipedia/commons/c/c4/Utf8webgrowth.svg)

UTF-8的延伸閱讀：
* [A Programmer’s Introduction to Unicode](http://reedbeta.com/blog/programmers-intro-to-unicode/)
* [每個軟體開發者都絕對一定要會的Unicode及字元集必備知識(沒有藉口！)](http://local.joelonsoftware.com/wiki/The_Joel_on_Software_Translation_Project:%E8%90%AC%E5%9C%8B%E7%A2%BC)

<a name="note1"></a>註1：使用Unicode編碼的文字檔，會在檔案的開頭放入Byte order mark(BOM)來表示檔中byte的存放順序。以UTF-16來說，BOM若是`FE FF`代表示Big-endian的擺放方式，其詳情請看[Byte order mark - Wikipedia](https://en.wikipedia.org/wiki/Byte_order_mark)。另外UTF-8的檔案，並不一定需要BOM，在有些文字編輯器的編碼選項(ex:Notepad++)會有UTF-8和UTF-8 without BOM可以選擇。
