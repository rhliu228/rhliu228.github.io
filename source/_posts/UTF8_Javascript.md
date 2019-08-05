---
title: 如何用Javascript对字符进行UTF8编解码
date: 2019-07-15 15:00:00
tags: 
- Javascript
- 字符编码
---
前段时间，接到一个字节校验的需求，要求我校验某个字段的长度，用户输入的数据转变为UTF8字符后，其字节长度不得超过1024。我立马想到，javascript内部字符以UTF-16的格式存储，要转化成UTF-8字符，需要找到合适的转化函数。通过查阅资料，发现其实这个问题并没有想象得这么复杂，还突然认识了URL编码函数的强大，所以特写下这篇文章，记录一下解决这个问题的思路。
<!-- more -->
一开始，找到的方法是这样子的：
```
   var utf16ToUtf8 = function (utf16Str) {
        var utf8Arr = [];
        var byteSize = 0;
        for (var i = 0; i < utf16Str.length; i++) {
            //获取字符Unicode码值
            var code = utf16Str.charCodeAt(i);

            //如果码值是1个字节的范围，则直接写入
            if (code >= 0x00 && code <= 0x7f) {
                byteSize += 1;
                utf8Arr.push(code);

                //如果码值是2个字节以上的范围，则按规则进行填充补码转换
            } else if (code >= 0x80 && code <= 0x7ff) {
                byteSize += 2;
                utf8Arr.push((192 | (31 & (code >> 6))));
                utf8Arr.push((128 | (63 & code)))
            } else if ((code >= 0x800 && code <= 0xd7ff)
                || (code >= 0xe000 && code <= 0xffff)) {
                byteSize += 3;
                utf8Arr.push((224 | (15 & (code >> 12))));
                utf8Arr.push((128 | (63 & (code >> 6))));
                utf8Arr.push((128 | (63 & code)))
            } else if(code >= 0x10000 && code <= 0x10ffff ){
                byteSize += 4;
                utf8Arr.push((240 | (7 & (code >> 18))));
                utf8Arr.push((128 | (63 & (code >> 12))));
                utf8Arr.push((128 | (63 & (code >> 6))));
                utf8Arr.push((128 | (63 & code)))
            }
        }

        return utf8Arr
    }
```
这个方法的关键是通过charCodeAt()获取每个字符对应的码点，然后判断码点的范围，将对应的字节大小和长度存放到数组中。但是试验后发现，这个方法无法识别和处理码点大于0xffff的字符。后来我想到encodeURIComponent就是对参数进行UTF-8编码的，能不能借助这个方法干点什么。随后，在网上找到了以下方法：
```
//从utf-16字符串转换成utf-8字节数组	
function utf8_from_str(s) {
    for(var i=0, enc = encodeURIComponent(s), a = []; i < enc.length;) {
        if(enc[i] === '%') {
            a.push(parseInt(enc.substr(i+1, 2), 16))
            i += 3
        } else {
            a.push(enc.charCodeAt(i++))
        }
    }
    return a
}
//从utf-8字节数组转换成utf-16字符
function utf8_to_str(a) {
    for(var i=0, s=''; i<a.length; i++) {
        var h = a[i].toString(16)
        if(h.length < 2) h = '0' + h
        s += '%' + h
    }
    return decodeURIComponent(s)
}

// or
//从utf-8字节数组转换成utf-16字符
function uintToString(uintArray) {
    var encodedString = String.fromCharCode.apply(null, uintArray);
    var decodedString = decodeURIComponent(escape(encodedString));
    return decodedString;
}
```

encodeURIComponent和decodeURIComponent会对参数进行UTF-8编码，但是这两个都是百分号编码，会在编码后的字节前面加上%，所以处理的时候需要充分利用好这个%。
不过观察上述最后一个函数：

```
var decodedString = decodeURIComponent(escape(encodedString));
```

居然用到了不推荐使用的escape函数，真是一脸懵。查阅资料后发现，对于非ASCII字符的Unicode字符，escape的编码方式是%uxxxx，其中的xxxx是用来表示unicode字符的4位十六进制字符。这种方式已经被W3C废弃了。但是在ECMA-262标准中仍然保留着escape的这种编码语法。encodeURI和encodeURIComponent则使用UTF-8对非ASCII字符进行编码，然后再进行百分号编码，这是RFC推荐的。上面语句中的encodedString是一串字符乱码，使用escape()方法对其编码，由于其不能识别utf-8字节，只会在每个字节前加上百分号。然后再采用decodeURIComponent解码，就可以得到正确的utf-16字符了。
如果只需要在utf-16字符和utf-8字符之间简单转换，网友给出了更简洁的解法：
```
function encode_utf8(s) {
  return unescape(encodeURIComponent(s));
}

function decode_utf8(s) {
  return decodeURIComponent(escape(s));
}
```
最后，贴上字符和ArrayBuffer数组之间的转换：
```
// 字符串转为ArrayBuffer对象，借助ArrayBuffer对象的byteLength可以获取字节长度
const str2ab = function(str) {
  var buf = new ArrayBuffer(str.length * 2); // 每个字符占用2个字节
  var bufView = new Uint16Array(buf);
  for (var i = 0, strLen = str.length; i < strLen; i++) {
    bufView[i] = str.charCodeAt(i);
  }
  return buf;
}
// ArrayBuffer转为字符串
const ab2strU16 = function (buf) {
  return String.fromCharCode.apply(null, new Uint16Array(buf));
}
```

* 参考
[Decode UTF-8 with Javascript](https://stackoverflow.com/questions/13356493/decode-utf-8-with-javascript)