[TOC]





 # 开发插件






 # 上传插件


> 把插件上传到github 后 把插件的信息添加到数据库 字段如下 ：


```
{
    "msg": "成功",
    "err": 0,
    "plugin": {
        "id": 7,
        "name": "zoomImage",
        "ios_url": "https://github.com/farwolf2010/zoomImage-ios",
        "android_url": "https://github.com/farwolf2010/zoomImage",
        "desc": "图片放大查看器",
        "download": 1,
        "ios_author": 1,
        "android_author": 1,
        "ios_update_time": 1546522873000,
        "android_update_time": 1546522876000
    }
}
```

 # 安装插件

 > 执行的命令是 weexplus plugin add zoomImage
 > 安装插件的时候 先请求后台接口，得的以上字段后 在逐个 下载 github




##  要进行安装时候的json数据


    ```
    {
        "name": "zoomImage",
        "dir": "mycli",
        "platform": "all"
    }

    ```


##  读取接口数据

 - 插件命令 可以安装单个 所以进行了判断，默认 add 就是下载所有的
 - 根据上面的json格式进行提示和下载对应的插件


```
  getInfo(op,(res)=>{
    if(op.platform=='all'){

          if(res.ios_url!==''&&res.android_url!==''){
            op.callback=()=>{
              if(res.ios_url!==''){
                ios.add(op)
              }
            };
            android.add(op)
          }else {
            if(res.ios_url!==''){
              log.info('只检测到ios端插件，开始安装！');
              ios.add(op)
            }
            if(res.android_url!==''){
              log.info('只检测到android端插件，开始安装！');
              android.add(op)
            }
          }

    }
    else if(op.platform=='android'){
      if(res.android_url!==''){
        android.add(op)
      }else{
        log.fatal('未检测到android端插件，无法安装！');
      }
    }
    else if(op.platform=='ios'){
      if(res.ios_url!==''){
        ios.add(op)
      }else{
        log.fatal('未检测到ios端插件，无法安装！');
      }
    }
  })
```

## 下载git代码

> 拼接处理git地址进行下载

```
function download(op){
  const downloadRepo = require('download-repo');

  process.chdir('platforms/android');

  let url=op.android_url.replace('https://github.com/','');
  url=url.replace('.git','');
  let dinfo={};
  dinfo.target=op.name;
  if(op.tag)
  dinfo.tag=op.tag;

  downloadRepo(url, dinfo).then(() => {

      log.info('插件'+op.name+' android端下载完毕，开始安装！');
      process.chdir('../../');
      addSetting(op);
      addGradle(op);
      invokeScript(op);

  },(exp)=>{
      console.log('exp='+exp);
      log.fatal('插件'+op.name+' android端下载失败！');
  })

}
```

## 修改android端的主配置文件

> 目的是为了把此项目作为modulef方式


```
function changeSetting(op,add){
   var path=  process.cwd();
  path+='/platforms/android/'+op.dir+'/settings.gradle'
  var result=fs.readFileSync(path, 'utf8');

  let temp=result.split('\n');
  if(temp[0].indexOf(op.name)!=-1){
    log.fatal('项目下存在同名module，请先删除!');
    return
  }

  let out=[];
  for(let t in temp){
    if(temp[t].indexOf(op.name)==-1) {
      out.push(temp[t])
    }
  }
  if(add){
    out.push('');
    out.push('include \''+':'+op.name+'\'');
    out.push('project(\''+':'+op.name+'\').projectDir = new File(\'../'+op.name+'\')');
  }

  let s='';
  out.forEach((item)=>{
    s+=item+'\n'
  });

  fs.writeFileSync(path,s,{encode:'utf-8'})

}
```

> 添加到主项目的 gradle ，以: api project(':zoomImage')让weex端可以调用里面的activity


```
function changeGradle(op,add){
  let path=  process.cwd();
  path+='/platforms/android/'+op.dir+'/app/build.gradle';
  let result=fs.readFileSync(path, 'utf8');
  let res=''+result.substr(result.indexOf('dependencies'),result.length)
  let temp=res.split('\n');
  let out=[];
  temp.forEach((item)=>{
    if(item.indexOf(':'+op.name)==-1){
      out.push(item)
    }
  });
  let weg=out[out.length-1];
  out=out.splice(0,out.length-2);
  if(add){
    out.push('    api project(\':'+op.name+'\')')
  }
  out.push('}');
   let px='';
  out.forEach((item)=>{
    px+=item+'\n'
  })
  result=result.replace(res,px)
  fs.writeFileSync(path,result,{encode:'utf-8'})
}
```



# 优化


> 不进行反射注册 ，直接修改文件内容进行注册 ，要在 json添加包名字段



```
{
    name:'zoomImage',
    android:{
        url:'xxx',
        page:'demo.class'
    },
    ios:{
        url:"xxx"
    }

}
```



```
  //找到文件注册的路径,进行直接添加
  path + ='/platforms/android/kjwx_plugin/src/main/java/com/kjwx_plugin/PluginManager.java';


```

> 问题是 activity怎么处理

![image](http://note.youdao.com/yws/res/60063/BFAC19B45B2D40ABAD157724763A9B6A)


> 在数据库上传的时候 进行标注是无activity的module还是存在activity的module
带activity的话当做单独的项目进行处理，不是activity的直接解压到module或component
解决编译的时候慢的问题(或者不考虑直接都按一个项目就没activity的问题)


**打成aar包是不是就没些问题呢**

 [优化arr](#)

> 三个步骤

1. 把项目解压到libs里面

2. 在插件项目添加      api(name: 'zoomimage-release', ext: 'aar')

3. 在PluginManager里面动态添加  WXSDKEngine.registerModule("img", WXZoomImageModule.class); 和 import com.kjwx_poject.zoomImage.module.WXZoomImageModule; 即可


包路径最好在添加插件的时候填写好 ： com.kjwx_poject.zoomImage.module.WXZoomImageModule



> 代码实现



```

要进行动态添加的类


public class PluginManager {


    public static void registerModule(Application application) throws WXException {

    }

    public static void registerComponent(Application application) throws WXException {

    }


}


插入包名

 let path = process.cwd();
    path += '/platforms/android/kjwx_plugin/src/main/java/com/kjwx_plugin/PluginManager.java';
    let result = fs.readFileSync(path, 'utf8');
     // let res=''+result.substr(result.indexOf('private'),result.length);
     let res=''+result.substr(result.indexOf('package'),result.length);
    let out=[];
    let temp=res.split('\n');
    temp.forEach((item)=>{
        //判断节点是否相同，不相同则拼接
        if(item.indexOf(':'+op.name)==-1){
            out.push(item)
        }
    });
    //此处应动态填写对应的 包名
    out.splice(1,0,'import com.kjwx_poject.zoomImage.module.WXZoomImageModule;');
   let px='';
    out.forEach((item)=>{
        px += item+'\n'
    });
    result=result.replace(res,px);
    fs.writeFileSync(path,result,{encode:'utf-8'});

  插入注册函数

   let path = process.cwd();
   path += '/platforms/android/kjwx_plugin/src/main/java/com/kjwx_plugin/PluginManager.java';
   let result = fs.readFileSync(path, 'utf8');
   // 可进行判断 添加 Component 或者 Module
   let res=''+result.substr(result.indexOf('registerComponent'),result.length);
    let out=[];
    let temp=res.split('\n');
    temp.forEach((item)=>{
     //判断节点是否相同，不相同则拼接
        if(item.indexOf(':'+op.name)==-1){
            out.push(item)
        }
    });
    //此处应该动态替换 对应的 类名 和 名称
    out.splice(1,0,'WXSDKEngine.registerModule("img", WXZoomImageModule.class);');
   let px='';
    out.forEach((item)=>{
        px += item+'\n'
    });
    result=result.replace(res,px);
    fs.writeFileSync(path,result,{encode:'utf-8'});


```





/Users/hanweiguang/Documents/weex_demo/aa/cc/android/
