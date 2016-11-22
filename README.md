# spring-boot-mvc-ueditor-qiniu
spring boot 、springMVC环境集成百度ueditor、七牛云存储


原料：

1、ueditor源码,版本1.4.3.3</br>
2、ueditor上传文件路径配置官方文档(http://fex.baidu.com/ueditor/#server-path)</br>
3、七牛sdk(版本：7.0.7)及文档</br>
特点：

    - 基于spring boot、spirngMVC环境，集成百度UEditor和七牛存储

    - 重写了存储图片的代码逻辑

    - 集成七牛sdk

    - 修改了多图上传弹框的图片列表相关js的代码

    - 将UEditor的config.json配置化

    - 内置springMVC上传接口

在spring boot环境中，只需引入jar，配置文件中添加config.json配置及七牛相关配置即可使用。
修改记录：

1、上传接口：
```
@Controller
@RequestMapping("/ueditor")
public class UeditorController {
    @Autowired 
    private ActionEnter actionEnter;
    @ResponseBody
    @RequestMapping("/upload")
    public String exe(HttpServletRequest request){
        return actionEnter.exec(request);
    }
}
```
2、基于spring boot配置：
```
@ConfigurationProperties(prefix = "ueditor")
public class UeditorProperties {

    private String config = "{}";

    private String accessKey;

    private String secretKey;

    private String bucket;

    private String baseUrl;

    private String uploadDirPrefix = "ueditor/file/";

    //......
}
```
3、重写StorageManager类，只使用七牛上传图片，去除了旧代码存储到本地的功能。
```
public static String accessKey;
public static String secretKey;
public static String baseUrl;
public static String bucket;
public static String uploadDirPrefix;

public static State saveBinaryFile(byte[] data, String path) {
        State state = null ;
        String key = uploadDirPrefix + getFileName(path);
        try {
            String uploadToken = Auth.create(accessKey, secretKey).uploadToken(bucket);
            Response response = new UploadManager().put(data, key, uploadToken);
            if (response.statusCode == 200) {
                state = new BaseState(true);
                state.putInfo("size", data.length);
                state.putInfo("title", path);
                state.putInfo("url", baseUrl + key);
            } else {
                state = new BaseState(false, AppInfo.IO_ERROR);
            }
        } catch (QiniuException e) {
            state = new BaseState(false, AppInfo.IO_ERROR);
        }
        return state;
    }
```
4、重写多个类ImageHunter、ActionEnter、FileManager等
5、修改ueditor/dialogs/image/image.js的857行getImageData()方法,此方法是向后台拉取图片列表数据；为了能分页，需要迎合七牛的sdk规范，此处添加了参数：marker ；相对应，后台的FileManager类listFile方法也做了修改。
```
 /* 初始化第一次的数据 */
 initData: function () {
     /* 拉取数据需要使用的值 */
     this.state = 0;
     this.listSize = editor.getOpt('imageManagerListSize');
     this.listIndex = 0;
     this.listEnd = false;
     this.marker =  undefined;
     /* 第一次拉取数据 */
     this.getImageData();
 },
 //......
 /* 向后台拉取图片列表数据 */
 getImageData: function () {
            var _this = this;

            if(!_this.listEnd && !this.isLoadingData) {
                this.isLoadingData = true;
                var url = editor.getActionUrl(editor.getOpt('imageManagerActionName')),
                    isJsonp = utils.isCrossDomainUrl(url);
                ajax.request(url, {
                    'timeout': 100000,
                    'dataType': isJsonp ? 'jsonp':'',
                    'data': utils.extend({
                            start: this.listIndex,
                            size: this.listSize,
                            marker:this.marker
                        }, editor.queryCommandValue('serverparam')),
                    'method': 'get',
                    'onsuccess': function (r) {
                        try {
                            var json = isJsonp ? r:eval('(' + r.responseText + ')');
                            if (json.state == 'SUCCESS') {
                                _this.pushData(json.list);
                                _this.listIndex = parseInt(json.start) + parseInt(json.list.length);
                                _this.marker = json.marker;
                                if(_this.listIndex >= json.total) {
                                    _this.listEnd = true;
                                }
                                if("true"==json.isLast){
                                    _this.listEnd = true;
                                }
                                _this.isLoadingData = false;
                            }
                        } catch (e) {
                            if(r.responseText.indexOf('ue_separate_ue') != -1) {
                                var list = r.responseText.split(r.responseText);
                                _this.pushData(list);
                                _this.listIndex = parseInt(list.length);
                                _this.listEnd = true;
                                _this.isLoadingData = false;
                            }
                        }
                    },
                    'onerror': function () {
                        _this.isLoadingData = false;
                    }
                });
            }
        },
```
使用：

1、引入jar (源码在github，请自行编译后再引入)：
```
<dependency>
    <groupId>com.baidu</groupId>
    <artifactId>spring-boot-ueditor-mvc</artifactId>
    <version>1.4.3.3-SNAPSHOT</version>
</dependency>
```
2、spring boot启动类添加扫描: @ComponentScan(basePackages = {“com.baidu”})：
```
@Controller
@ComponentScan(basePackages = {"com.zrk","com.baidu"})
@SpringBootApplication
public class SpringBootMvcUeditorQiniuDemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringBootMvcUeditorQiniuDemoApplication.class, args);
    }

    @RequestMapping("/")
    public String index(){
        return "ueditor";
    }

}
```
3、application.properties中添加ueditor的config.json配置，和七牛相关配置：
```
#开启thymeleaf
spring.thymeleaf.enabled=true
#上传文件大小
spring.http.multipart.max-file-size=500MB
spring.http.multipart.max-request-size=20MB

#ueditor的config.json配置，原配置在ueditor资源目录jsp/config.json，拷到此处时请将原json去除掉注释，并压缩为一行，参考 “config纯净版.json”
ueditor.config={"imageActionName":"uploadimage","imageFieldName":"upfile","imageMaxSize":2048000,"imageAllowFiles":[".png",".jpg",".jpeg",".gif",".bmp"],"imageCompressEnable":true,"imageCompressBorder":1600,"imageInsertAlign":"none","imageUrlPrefix":"","imagePathFormat":"/ueditor/jsp/upload/image/{yyyy}{mm}{dd}/{time}{rand:6}","scrawlActionName":"uploadscrawl","scrawlFieldName":"upfile","scrawlPathFormat":"/ueditor/jsp/upload/image/{yyyy}{mm}{dd}/{time}{rand:6}","scrawlMaxSize":2048000,"scrawlUrlPrefix":"","scrawlInsertAlign":"none","snapscreenActionName":"uploadimage","snapscreenPathFormat":"/ueditor/jsp/upload/image/{yyyy}{mm}{dd}/{time}{rand:6}","snapscreenUrlPrefix":"","snapscreenInsertAlign":"none","catcherLocalDomain":["127.0.0.1","localhost","img.baidu.com"],"catcherActionName":"catchimage","catcherFieldName":"source","catcherPathFormat":"/ueditor/jsp/upload/image/{yyyy}{mm}{dd}/{time}{rand:6}","catcherUrlPrefix":"","catcherMaxSize":2048000,"catcherAllowFiles":[".png",".jpg",".jpeg",".gif",".bmp"],"videoActionName":"uploadvideo","videoFieldName":"upfile","videoPathFormat":"/ueditor/jsp/upload/video/{yyyy}{mm}{dd}/{time}{rand:6}","videoUrlPrefix":"","videoMaxSize":102400000,"videoAllowFiles":[".flv",".swf",".mkv",".avi",".rm",".rmvb",".mpeg",".mpg",".ogg",".ogv",".mov",".wmv",".mp4",".webm",".mp3",".wav",".mid"],"fileActionName":"uploadfile","fileFieldName":"upfile","filePathFormat":"/ueditor/jsp/upload/file/{yyyy}{mm}{dd}/{time}{rand:6}","fileUrlPrefix":"","fileMaxSize":51200000,"fileAllowFiles":[".png",".jpg",".jpeg",".gif",".bmp",".flv",".swf",".mkv",".avi",".rm",".rmvb",".mpeg",".mpg",".ogg",".ogv",".mov",".wmv",".mp4",".webm",".mp3",".wav",".mid",".rar",".zip",".tar",".gz",".7z",".bz2",".cab",".iso",".doc",".docx",".xls",".xlsx",".ppt",".pptx",".pdf",".txt",".md",".xml"],"imageManagerActionName":"listimage","imageManagerListPath":"/ueditor/jsp/upload/image/","imageManagerListSize":20,"imageManagerUrlPrefix":"","imageManagerInsertAlign":"none","imageManagerAllowFiles":[".png",".jpg",".jpeg",".gif",".bmp"],"fileManagerActionName":"listfile","fileManagerListPath":"/ueditor/jsp/upload/file/","fileManagerUrlPrefix":"","fileManagerListSize":20,"fileManagerAllowFiles":[".png",".jpg",".jpeg",".gif",".bmp",".flv",".swf",".mkv",".avi",".rm",".rmvb",".mpeg",".mpg",".ogg",".ogv",".mov",".wmv",".mp4",".webm",".mp3",".wav",".mid",".rar",".zip",".tar",".gz",".7z",".bz2",".cab",".iso",".doc",".docx",".xls",".xlsx",".ppt",".pptx",".pdf",".txt",".md",".xml"]}

#七牛云存储配置
ueditor.access-key=8nU0zA9aTvfHBZs0fPZZWd8gpnFRtOkOPkiTB6M0
ueditor.secret-key=400iGAeaeJyjgSm26-wT8R-HQYZbBR1el_cDiRIq
ueditor.bucket=zrk-test
#域名，可使用请你提供的域名，或自己绑定的域名
ueditor.base-url=http://od710rrnd.bkt.clouddn.com
#文件上传到七牛的目录，默认为‘ueditor/file/’，请使用‘/’结尾
ueditor.upload-dir-prefix=biz/img/
```
