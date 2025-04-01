---
tags:
  - python
  - lxml
  - xml
  - beautifulsoup
  - eTree
---
针对xml文件的解析， python提供了多种方法，针对不同场景可以有针对性的选择。

	1.ElementTree 会解析xml为 tree,操作直观, 但是耗费内存
	2.sax 会挨个解析, 并执行回调
	3.lxml API 使用类似 elementTree, 但是速度快
	4.beautifulSoup 解析xml,底层需要依赖lxml (lxml-xml), api丰富



测试文件内容：
```xml
<collection shelf="New Arrivals">
<movie title="Enemy Behind">
   <type>War, Thriller</type>
   <format>DVD</format>
   <year>2003</year>
   <rating>PG</rating>
   <stars>10</stars>
   <description>Talk about a US-Japan war</description>
</movie>
<movie title="Transformers">
   <type>Anime, Science Fiction</type>
   <format>DVD</format>
   <year>1989</year>
   <rating>R</rating>
   <stars>8</stars>
   <description>A schientific fiction</description>
</movie>
<movie title="Trigun">
   <type>Anime, Action</type>
   <format>DVD</format>
   <episodes>4</episodes>
   <rating>PG</rating>
   <stars>10</stars>
   <description>Vash the Stampede!</description>
</movie>
<movie title="Ishtar">
   <type>Comedy</type>
   <format>VHS</format>
   <rating>PG</rating>
   <stars>2</stars>
   <description>Viewable boredom</description>
</movie>
</collection>
```

#### elementTree
```python
import xml.etree.ElementTree as ET
import os

current = os.getcwd()
filepath = os.path.join(current, "base", "xml", "analyse.xml")
destfile = os.path.join(current, "base", "xml", "dest.xml")
print(filepath)

## parse xml file
tree = ET.parse(filepath)

## get root node
root = tree.getroot()
print(f"root is {root.tag}")

## find node
for item in root.iter("movie"):
    print(f"tag: {item.tag}, text: {item.text},title: {item.attrib}")

# get sibling node of '<movie title="Enemy Behind">'
movie = root.find(".//movie[@title='Enemy Behind']")
print(f"tag: {movie.tag}, title: {movie.attrib.get('title')}")

## update movie's attribute and child's text
movie.attrib.update({"title":"update1"})

## print all content or write to files
result = ET.tostring(root, encoding="unicode")
print(f"result = {result}")

### write to new file
tree.write(destfile,method="xml")
```


#### lxml
```python
from lxml import etree
import os

current = os.getcwd()
filepath = os.path.join(current, "base", "xml", "analyse.xml")
destpath = os.path.join(current, "base", "xml", "dest.xml")

with open(filepath) as file:
    content = file.read()

## parse content
tree = etree.fromstring(content)

## get field and methods
print(dir(tree))

## find all 'movie' node
movies = tree.findall("movie")
print(dir(movies[0]))
for m in movies:
    print(f"tag: {m.tag}, text: {m.text}, ")

## find node with xpath
trans = tree.xpath("//movie[@title='Transformers']")[0]
print(f"trans tag: {trans.tag}, attributs: {trans.attrib}")
  
## update trans node value
trans.attrib.update({"title":"transformer from soup"})

## print final xml content
result = etree.tounicode(tree)
print(f"result: {result}")
```


####  beautifulSoup
```python
from bs4 import BeautifulSoup
import os

current = os.getcwd()
filepath = os.path.join(current, "base", "xml", "analyse.xml")
destfile = os.path.join(current, "base", "xml", "dest.xml")
  
# get file content
with open(filepath, "r")  as file:
    content = file.read()

## parse file content
soup = BeautifulSoup(content, features="lxml-xml")

## find all movie node
items = soup.find_all("movie", {"title": "Transformers"})

for item in items:
    #print(dir(item))   # inspect the fields and methods
    print(f"tag: {item.name}, text: {item.text}, attrs: {item.attrs} ")

print("*" * 50)
## find sibling node
node = items[0]
nextnode = node.find_next_sibling()

print(f"next_node: {nextnode.name}, text: {nextnode.text}, attrs: {nextnode.attrs}")
  
print("*" * 20, "encoding to string ", "*" * 20)
## dump to string
result = soup.decode_contents()
print(result)
```


> note:
>  针对一些object, 不清楚其有哪些field and methods,  使用`print(dir(obj))`把对象的内容打印出来。



