Title: WebGL2 - 跨域图像
Description: 使用跨域图像
TOC: 跨域图像

此文上接 WebGL 系列文章，如果没读建议从[早期文章](webgl-fundamentals.html)开始。

在 WebGL 中通常将图像下载并上传到 GPU 当作纹理使用。这有几个相关的例子，
例如有关[图像处理](webgl-image-processing.html)的文章，
有关[纹理](webgl-3d-textures.html)的文章和有关
[二维 drawImage 实现](webgl-2d-drawimage.html)的文章。

基本上我们像这样下载图像

    // 创建纹理信息 { width: w, height: h, texture: tex }
    // 纹理起初为 1×1 像素
    // 当图像下载完成后加载图像
    function loadImageAndCreateTextureInfo(url) {
      var tex = gl.createTexture();
      gl.bindTexture(gl.TEXTURE_2D, tex);
      // 用 1x1 蓝色像素填充纹理
      gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGBA, 1, 1, 0, gl.RGBA, gl.UNSIGNED_BYTE,
                    new Uint8Array([0, 0, 255, 255]));

      gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_S, gl.CLAMP_TO_EDGE);
      gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_T, gl.CLAMP_TO_EDGE);

      var textureInfo = {
        width: 1,   // 在图像下载完之前不知道大小
        height: 1,
        texture: tex,
      };
      var img = new Image();
      img.addEventListener('load', function() {
        textureInfo.width = img.width;
        textureInfo.height = img.height;

        gl.bindTexture(gl.TEXTURE_2D, textureInfo.texture);
        gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGBA, gl.RGBA, gl.UNSIGNED_BYTE, img);
        gl.generateMipmap(gl.TEXTURE_2D);
      });
      img.src = url;

      return textureInfo;
    }

问题是图像可能含有一些私有信息（例如验证码，签名，一个裸照，...），
而一个网页通常含有一些广告之类的信息，这些信息并不是由页面直接控制的，
所以浏览器需要防止这些信息读取私有图片。

直接使用 `<img src="private.jpg">` 是没什么问题的，因为图像尽管被浏览器显示，
便签对象并不能获取图像的内部数据。[Canvas2D API](https://developer.mozilla.org/en-US/docs/Web/API/CanvasRenderingContext2D)
有一个方式可以读取图像信息，首先需要将图像绘制到画布

    ctx.drawImage(someImg, 0, 0);

然后就可以获取数据

    var data = ctx.getImageData(0, 0, width, heigh);

但是如果绘制的图像来自不同的域名，浏览器就会将画布标记为**被污染**，
然后当你调用 `ctx.getImageData` 时就会得到一个安全错误。

WebGL 则更进一步，在 WebGL 中 `gl.readPixels` 和 `ctx.getImageData` 是相似的，
所以你可能以为把这个接口封闭就好了，但事实是即使不能直接获取像素值，
也可以创建一个基于图像颜色的着色器，虽然效率低但是可以等同于获取到了图像信息。

所以 WebGL 直接禁止所有来自不同域名的图像。例如这里有个简单的例子，
绘制一个旋转的矩形，纹理是其他域名下的图像，会发现纹理永远都没有加载，
并且会得到一个错误信息

{{{example url="../webgl-cors-permission-bad.html" }}}

我们该怎么办呢？

## 介绍 CORS

CORS = Cross Origin Resource Sharing（跨域资源共享）。是一种网页向图像服务器请求使用图像许可的方式。

要实现这个我们需要设置 `crossOrigin` 属性然后浏览器会试图从服务器获取图像，如果不是相同的域名，
浏览器会请求 CORS 许可。

    ...
    +    img.crossOrigin = "";   // 请求 CORS 许可
        img.src = url;

您设置 `crossOrigin` 的字符串被发送到服务器。服务器可以检查该字符串并决定
是否给予您许可。 大多数支持 CORS 的服务器不检查这个字符串，他们只是
给予所有人许可。 这就是将其设置为空字符串的原因。 在这种情况下，它的意思是“请求许可”，而说`img.crossOrigin = "bob"` 的意思是“请求'bob'的许可”。

为什么不总是请求许可？ 因为请求许可需要发起 2 个 HTTP 请求，它比不请求要慢。如果我们知道我们在同一个域中或者我们知道除了 img 标签或 canvas2d 之外我们不会将图像用于任何其他内容，那么我们不设置 `crossOrigin`，因为它会使事情变慢。

我们可以定义一个方法，当图像资源在其他域名下时就设置 `crossOrigin` 属性。

    function requestCORSIfNotSameOrigin(img, url) {
      if ((new URL(url, window.location.href)).origin !== window.location.origin) {
        img.crossOrigin = "";
      }
    }

然后像这样使用

    ...
    +requestCORSIfNotSameOrigin(img, url);
    img.src = url;

{{{example url="../webgl-cors-permission-good.html" }}}

需要注意的是请求许可**不是**表示你一定会得到许可，结果取决于服务器。
Github Pages 提供许可，flickr.com 提供许可，imgur.com 提供许可，但大多数网站不提供许可。

<div class="webgl_bottombar">
<h3>使 Apache 服务器提供 CORS 许可</h3>
<p>如果你使用 Apache 服务器，并且安装了 mod_rewrite 插件，就可以将</p>
<pre class="prettyprint">
    Header set Access-Control-Allow-Origin "*"
</pre>
<p>
放在合适的 <code>.htaccess</code> 文件中来发放公共 CORS 许可。
</p>
</div>
