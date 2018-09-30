+++
title = "Font-Awesome-SVG-PNG 思路分析"
date = 2018-06-09T00:03:55+08:00
tags = ["SVG"]
categories = ["前端"]
draft = false
+++

# Font-Awesome-SVG-PNG 思路分析

微软俱乐部的 Hackathon 的时候写了 [iCon4U](https://github.com/agrimonia/iCon4U) 这个项目。因为需要出 png 格式的图，参考了 [Font-Awesome-SVG-PNG](https://github.com/encharm/Font-Awesome-SVG-PNG/) 这个项目，大致看了一下它的源码，写一写我的理解。

现在的代码比较复杂，我回滚到了14年。发现前几个 commit 除了简单地上传了一堆 svg, png 格式的 icons以外，只有一个 js 文件，核心代码就这几行：

```javascript
if (argv.color && argv.png) {
  commandChecks.push(new Promise(function (resolve, reject) {
// @todo: surely there's a module in npm that allows to do command checks automagically?
    var convertTest = require('child_process').spawn('rsvg-convert', ['--help']);
    convertTest.once('error', function () {
      throw Error("Error: cannot start `rsvg-convert` command. Please install it or verify that it is in your PATH.");
    });
    convertTest.once('exit', function (code) {
      if (code) return reject();
      resolve();
    });
  }));
}

Promise.all(commandChecks).then(function () {
  return libFontAwesome.generate(argv);
}).catch(function (err) {
  console.error("Caught error", err);
});
```

可以看到转换出 svg 是通过 `libFontAwesome.generate(argv)`实现的，png 则额外开了一个 node 子进程，通过`spawn`执行了`rsvg-convert`命令，这是用来将 svg 转化成 png 的。

接下来的一个 commit 进行了一次重构，剥离出了`generateSvg()`这个函数

```javascript
var template =
'<svg width="1792" height="1792" viewBox="{shiftX} -256 1792 1792">' +
'<g transform="scale(1 -1) translate(0 -1280)">' +
'<path d="{path}" fill="{color}" />' +
'</g>' +
'</svg>';
/***********************/
function generateSvg(name, params) {
  var svgFolder = pathModule.join(params.destFolder, params.color, 'svg');
  mkdirp.sync(svgFolder);

  return new Promise(function(resolve, reject) {
    var outSvg = fs.createWriteStream(pathModule.join(svgFolder, name + '.svg'));
    svgo.optimize(getIconSvg(params), function(result) {
      outSvg.write('<?xml version="1.0" encoding="utf-8"?>' +'\n');	
      outSvg.end(result.data);
      resolve();
    });
  });
}
```

思路是用`replace`去修改一个 svg 模板。

目前的 `getIconsvg()`函数：

```javascript
function getIconSvg(params, size) {
  let {path, color, advWidth} = params;
  const PIXEL_WIDTH = advWidth > 2048 ? 18 : (advWidth > 1792 ? 16 : 14);
  const PIXEL_HEIGHT = 14;
  
  const BASE_WIDTH = PIXEL_WIDTH * PIXEL;
  const BASE_HEIGHT = PIXEL_HEIGHT * PIXEL;
  
  var options = optionsForSize(PIXEL_WIDTH, PIXEL_HEIGHT, 
    parseInt((BASE_WIDTH / BASE_HEIGHT) * size), size,
    params.addPadding);

  let {paddingLeft, paddingTop, paddingRight, paddingBottom} = options;  
  let shiftX = -(-(BASE_WIDTH - advWidth)/2 - paddingLeft);
  let shiftY = -(-2*PIXEL - paddingTop);  
  let width = BASE_WIDTH + paddingLeft + paddingRight;
  let height = BASE_HEIGHT + paddingBottom + paddingTop;

  const result =
`<svg width="${width}" height="${height}" viewBox="0 0 ${width} ${height}" xmlns="http://www.w3.org/2000/svg">
  <g transform="translate(${shiftX} ${shiftY})">
  <g transform="scale(1 -1) translate(0 -1280)">
  <path d="${path}" fill="${color}" />
  </g></g>
</svg>`;
  
  return result;
}
```

最终使用模板字符串，但是原理没有很大变化，就是通过一个 svg 模板，传入 params 解构之后拼成 icon 对应的 svg 标签。

我想知道path是怎么来的，怎么从 font-awesome 中得出对应 icon 的 path？

继续往下看：

```javascript
console.log("Downloading latest icons.yml ...");
request('https://raw2.github.com/FortAwesome/Font-Awesome/master/src/icons.yml', function(error, response, iconsYaml) {
  console.log("Downloading latest fontawesome-webfont.svg ...");
  request('https://raw2.github.com/FortAwesome/Font-Awesome/master/fonts/fontawesome-webfont.svg', function(error, response, fontData) {
    fontData = fontData.toString('utf8');
    var icons = yaml.safeLoad(iconsYaml).icons;
    icons.forEach(function(icon) {
      code2name[icon.unicode] = icon.id;
    });
    lines = fontData.split('\n');
    async.eachLimit(lines, 4, function(line, cb) {
      var m = line.match(/^<glyph unicode="&#x([^"]+);"\s*(?:horiz-adv-x="(\d+)")?\s*d="([^"]+)"/);
      if(m) {
        var str = m[1];
        if(code2name[str]) {
          generateIcon(code2name[str], m[3], {advWidth: m[2]?m[2]:1536, color: argv.color }, cb);
        } else {
          cb();
        }
      } else
        cb();
    }, function(err, cb) {//略});
```

到这里思路就很明显了：

1. 获取到了 icons.yml 后，在`icons.forEach` 中用`code2name`拿到 icons name。
2. 用`line.match`一行正则拿到 path 和 尺寸。

------

所以它的思路是一行`foreach`一次转换完，但是我们做单 icon 的转换可以参考这种正则匹配路径的思路，拿到 icon 的路径传给 path。