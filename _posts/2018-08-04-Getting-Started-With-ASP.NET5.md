### 目录

* [新建项目](#新建项目)
* [Gulp使用说明](#Gulp使用说明)

#### 新建项目

新建 ASP.NET Web 空项目，添加以下Nuget包（其它依赖的包会自动引用）：

    Microsoft.AspNet.Mvc
    Microsoft.Owin
    Microsoft.Owin.StaticFiles

添加Startup类

```cs
[assembly: OwinStartup(typeof(Startup))]

public class Startup
{
    public void Configuration(IAppBuilder app)
    {
        app.UseStaticFiles(new StaticFileOptions()
        {
            RequestPath = new PathString("/wwwroot"),
            OnPrepareResponse = ctx =>
            {
                const int durationInSeconds = 60 * 60 * 24;
                ctx.OwinContext.Response.Headers[HeaderNames.CacheControl] =
                    "public,max-age=" + durationInSeconds;
            }
        });
    }
}
```

在项目目录下新建wwwroot目录，修改.csproj添加以下内容

```xml
<ItemGroup>
    <Compile Remove="wwwroot\**" />
    <EmbeddedResource Remove="wwwroot\**" />
    <None Remove="wwwroot\**" />
    <Content Include="wwwroot\**" />
    <Content Update="wwwroot\**">
        <CopyToPublishDirectory>PreserveNewest</CopyToPublishDirectory>
    </Content>
</ItemGroup>
```

wwwroot目录不会在VS项目资源管理器当中显示，但是发布时会被包括在内。后续所有的静态文件都会放到这里。

#### Gulp使用说明

VS默认使用Bower管理JQuery、Bootstrap之类的库，然而Bower实在不好用，所有这里用npm来代替Bower。为了方便使用命令行，还需要添加一个VS扩展：**Open Command Line**。

打开命令行到项目根文件夹下，运行命令

```ps
npm init
```

根据需要输入package name、version等等内容，然后安装以下包

```ps
npm install gulp gulp-concat gulp-uglify gulp-cssmin merge-stream --save-dev
```

按项目需要安装需要的包，这里安装以下包：

```ps
npm install jquery jquery-validation jquery-validation-unobtrusive bootstrap@3.3.7 -save
```

在VS项目资源管理器当中添加文件gulpfile.js，复制粘贴以下内容

```js
/// <binding BeforeBuild='clean, minify' Clean='clean' />
/*
This file is the main entry point for defining Gulp tasks and using Gulp plugins.
Click here to learn more. https://go.microsoft.com/fwlink/?LinkId=518007
*/

var gulp = require('gulp');
var uglify = require('gulp-uglify');
var concat = require('gulp-concat');
var cssmin = require('gulp-cssmin');
var del = require('del');
var merge = require('merge-stream');

var paths = {
    webroot: './wwwroot/',
};

paths.lib = paths.webroot + 'lib/';
paths.js = paths.webroot + 'js/**/*.js';
paths.minJs = paths.webroot + 'js/**/*.min.js';
paths.css = paths.webroot + 'css/**/*.css';
paths.minCss = paths.webroot + 'css/**/*.min.css';
paths.concatJsDest = paths.webroot + 'js/site.min.js';
paths.concatCssDest = paths.webroot + 'css/site.min.css';

// Dependency Dirs
var deps = {
    'bootstrap': {
        'dist/**/*': ''
    },
    'jquery': {
        'dist/*': ''
    },
    'jquery-validation': {
        'dist/**/*': ''
    },
    'jquery-validation-unobtrusive': {
        'dist/*': ''
    }
};

gulp.task('clean', function (cb) {
    return del([
        paths.lib,
        paths.minJs,
        paths.minCss,
        paths.concatJsDest,
        paths.concatCssDest
    ], cb);
});

gulp.task('minify:site', function () {

    var streams = [
        gulp.src([paths.js, '!' + paths.minJs], { base: '.' })
            .pipe(uglify())
            .pipe(concat(paths.concatJsDest))
            .pipe(gulp.dest('.'))
    ];

    streams.concat = [
        gulp.src([paths.css, '!' + paths.minCss])
            .pipe(cssmin())
            .pipe(concat(paths.concatCssDest))
            .pipe(gulp.dest('.'))
    ]

    return merge(streams);
});

gulp.task('minify:lib', function () {

    var streams = [];

    for (var prop in deps) {
        console.log('Prepping Scripts for: ' + prop);
        for (var itemProp in deps[prop]) {
            streams.push(gulp.src('node_modules/' + prop + '/' + itemProp)
                .pipe(gulp.dest(paths.lib + prop + '/' + deps[prop][itemProp])));
        }
    }

    return merge(streams);

});

gulp.task('minify', ['minify:site', 'minify:lib']);
```

[https://blog.bitscry.com/2018/03/13/using-npm-and-gulp-in-visual-studio-2017/](https://blog.bitscry.com/2018/03/13/using-npm-and-gulp-in-visual-studio-2017/)
