
## 修正 Webm 檔案丟失時間進度的問題

Javascript MediaStream Recording API 所錄製產生的影片會遺失時間進度

會導致一些影片播放器，無法拖拉進度條到後方進行播放

此方法採用 RecordRTC.js 當中的 getSeekableBlob() 來解決影片丟失時間的問題

## 具體方法

實際上用不到 RecordRTC.js 本身

需要的是 EBML.js 和 getSeekableBlob() 這個function就夠了

先在 HTML 頁面引入

```html
<script src="https://www.webrtc-experiment.com/EBML.js"></script>
```

之後把 RecordRTC.js 中 getSeekableBlob() 的內容把它搬出來

```js
/**
 * @param {Blob} file - File or Blob object.
 * @param {function} callback - Callback function.
 * @example
 * getSeekableBlob(blob or file, callback);
 * @see {@link https://github.com/muaz-khan/RecordRTC|RecordRTC Source Code}
 */
function getSeekableBlob(inputBlob, callback) {
    // EBML.js copyrights goes to: https://github.com/legokichi/ts-ebml
    if (typeof EBML === 'undefined') {
        throw new Error('Please link: https://www.webrtc-experiment.com/EBML.js');
    }

    var reader = new EBML.Reader();
    var decoder = new EBML.Decoder();
    var tools = EBML.tools;

    var fileReader = new FileReader();
    fileReader.onload = function(e) {
        var ebmlElms = decoder.decode(this.result);
        ebmlElms.forEach(function(element) {
            reader.read(element);
        });
        reader.stop();
        var refinedMetadataBuf = tools.makeMetadataSeekable(reader.metadatas, reader.duration, reader.cues);
        var body = this.result.slice(reader.metadataSize);
        var newBlob = new Blob([refinedMetadataBuf, body], {
            type: 'video/webm'
        });

        callback(newBlob);
    };
    fileReader.readAsArrayBuffer(inputBlob);
}

```

## 實際使用

```js
...
  var chunks = [];
  mediaRecorder.onstop = function(e) {
    var notFixedWebmBlob = new Blob(chunks, { 'type': 'video/webm' });
    getSeekableBlob(notFixedWebmBlob , function(isFixedWebmBlob){
        // isFixedWebmBlob is result
    })
  }
  mediaRecorder.ondataavailable = function(e) {
    chunks.push(e.data);
  }
...
```
