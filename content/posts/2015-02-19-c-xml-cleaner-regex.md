---
title: 'C# XML Cleaner Regex'
author: Ben E. Boyter
type: post
date: 2015-02-19T22:41:47+00:00
url: /2015/02/c-xml-cleaner-regex/
nkweb_code_in_head:
  - default
nkweb_Use_Custom_js:
  - default
nkweb_Use_Custom_Values:
  - default
nkweb_Use_Custom:
  - 'false'
categories:
  - 'C#'
  - Tip

---
One of the most annoying things I deal with is XML documents with invalid characters inside them. Usually caused by copy pasting from MS Word it ends up with invisible characters that you cannot easily find and cause XML parsers to choke. I have encountered this problem enough that I thought a quick blog post would be worth the effort.

As such here mostly for my own reference is a regular expression for C# .NET that will clean invalid XML characters from any XML file.

`const string InvalidXmlChars = @"[^\x09\x0A\x0D\x20-\xD7FF\xE000-\xFFFD\x10000-x10FFFF]";`

Simply take the XML document as a string and do a simple regular expression replace over it to remove the characters. You can then import into whatever XML document structure you want and process it as normal.

For note the reason I encountered this was when I was building a series of WSDL web-services over HR data. Since the data had multiple sources merged during an ETL process and some of them were literally CSV files I hit this issue a lot. Attempts to sanitize the data failed as it was overwritten every few hours and it involved multiple teams to change the source generation. In the end I just ran the cleaner over every field before it was returned and everything worked perfectly.