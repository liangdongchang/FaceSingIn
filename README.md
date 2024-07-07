# FaceSingIn
人脸识别签到系统:课堂签到，上班打卡，进出门身份验证。 功能：人脸录入，打卡签到，声音提醒，打卡信息导出，打包exe文件
## 人脸识别签到系统

### 1、运用场景

课堂签到，上班打卡，进出门身份验证。

### 联系我
微信 1257309054
![image](https://github.com/liangdongchang/FaceSingIn/assets/29998120/6e79912e-315b-4eb1-a27d-5da8ed8092e9)


### 2、功能架构

人脸录入，打卡签到，声音提醒，打卡信息导出：

![image](https://github.com/liangdongchang/FaceSingIn/assets/29998120/214525bb-6522-46c9-8574-89c6015f38bb)




### 3、技术栈

python3.8，sqlite3，opencv，face_recognition，PyQt5，csv

第三方库：

```
asgiref==3.8.1
click==8.1.7
colorama==0.4.6
comtypes==1.4.4
dlib @ file:////dlib-19.19.0-cp38-cp38-win_amd64.whl.whl#sha256=89a19fe0003e2fa2ff33264b5abba82443056f915b2339feb167569b4446a460
et-xmlfile==1.1.0
face-recognition==1.3.0
face_recognition_models==0.3.0
numpy==1.24.4
opencv-python==4.10.0.84
openpyxl==3.1.5
pillow==10.3.0
pypiwin32==223
PyQt5==5.15.10
PyQt5-Qt5==5.15.2
PyQt5-sip==12.13.0
pyttsx3==2.90
pytz==2024.1
pywin32==306
sqlparse==0.5.0
typing_extensions==4.12.2

```



### 4、人脸识别流程图

```
1、导入库
2、编写UI界面
3、打开摄像头录入人脸信息
4、比对人脸信息并发出声音提醒
5、导出打卡信息
6、打包成exe可执行文件
```
![image](https://github.com/liangdongchang/FaceSingIn/assets/29998120/8b19682b-c99f-4f71-88be-399c33fc8180)



### 5、数据库

#### 5.1、用户表

| 字段         | 描述         | 类型    | 大小 |
| ------------ | ------------ | ------- | ---- |
| id           | 主键         | integer |      |
| name         | 姓名         | text    | 256  |
| account      | 账号         | text    | 256  |
| password     | 密码         | text    | 256  |
| is_admin     | 是否为管理员 | integer |      |
| icon_feature | 人脸特征     | blob    | 1024 |
| addtime      | 添加时间     | text    | 256  |

sql语句：

```
PRAGMA foreign_keys = false;

-- ----------------------------
-- Table structure for user
-- ----------------------------
DROP TABLE IF EXISTS "user";
CREATE TABLE "user" (
  "id" INTEGER NOT NULL,
  "name" TEXT(256),
  "account" TEXT(256),
  "password" TEXT(256),
  "is_admin" integer,
  "icon_feature" blob(1024),
  "addtime" TEXT(256),
  PRIMARY KEY ("id")
);

PRAGMA foreign_keys = true;
```



#### 5.2、记录表

| 字段     | 描述                                         | 类型    | 大小 |
| -------- | -------------------------------------------- | ------- | ---- |
| id       | 主键                                         | integer |      |
| userid   | 用户id                                       | integer |      |
| name     | 用户姓名                                     | text    | 256  |
| is_login | 是否登录：0 ：登录，1：上班打卡，2：下班打卡 | integer |      |
| content  | 打卡日志                                     | text    | 256  |
| addtime  | 添加时间                                     | text    | 256  |

sql语句：

```
PRAGMA foreign_keys = false;

-- ----------------------------
-- Table structure for logs
-- ----------------------------
DROP TABLE IF EXISTS "logs";
CREATE TABLE "logs" (
  "id" INTEGER NOT NULL PRIMARY KEY AUTOINCREMENT,
  "userid" INTEGER,
  "name" TEXT(256),
  "is_login" integer,
  "content" TEXT(256),
  "addtime" TEXT
);

-- ----------------------------
-- Auto increment value for logs
-- ----------------------------
UPDATE "sqlite_sequence" SET seq = 12 WHERE name = 'logs';

PRAGMA foreign_keys = true;
```



### 6、UI界面

#### 6.1、主界面

```
主要功能：
1、摄像头实时采集人脸数据
2、账号密码打卡
3、输出打卡结果
4、语音播放
```
![image](https://github.com/liangdongchang/FaceSingIn/assets/29998120/dcbe33a8-bfd5-4a45-a6eb-bcdfd8f5c999)




#### 6.2、人脸录入功能

```
功能：
1、管理员登录
2、账号密码修改
3、人脸信息更改
4、用户信息更改
5、用户信息录入
6、摄像头采集人脸数据
```
![image](https://github.com/liangdongchang/FaceSingIn/assets/29998120/0436d850-abd9-4146-af46-6e5ef504befd)




#### 6.3、打卡记录

```python
功能：
1、打卡记录查询
2、打卡记录输出
3、打卡记录导出excel文件，并使用橙色标记异常情况，可在输出框直接打开文件
```
![image](https://github.com/liangdongchang/FaceSingIn/assets/29998120/15a67b15-d7b8-4bdb-8c3c-fdd54fbbcc72)




#### 6.4、代码

```python
# -*- coding: utf-8 -*-

"""
@contact: 微信 1257309054
@file: main.py
@time: 2024/6/30 11:09
@author: LDC
"""
class OpenLoginCameraThread(QThread):
    # 摄像头打开多线程
    _signal_thread = pyqtSignal(str)

    def __init__(self, parent=None):
        super(OpenLoginCameraThread, self).__init__(parent)
        self.window = parent
        self.qmut = QMutex()  # 互斥量
        self.is_exit_run = False  # 默认人脸识别里面的摄像头是循环打开的
        self.cap = None

    def get_capture(self):
        '''
        打开摄像头
        :return:
        '''
        try:
            self.cap = cv2.VideoCapture(0)
            w, h = 640, 360

            self.cap.set(3, w)
            self.cap.set(4, h)
            return True
        except Exception as e:
            self.cap = None
            self.window.sign_in_info.setText('打开摄像头失败，{}'.format(e))
            return False

    def open_login_camera(self):
        '''
        打开摄像头
        :return:
        '''

        self.is_close_camera = False  # 设备摄像头打开
        while 1:
            if self.is_exit_run:
                break
            if not self.cap:
                if not self.get_capture():
                    time.sleep(3)
                    continue
            try:
                success, img = self.cap.read()  # 读取图片
                mirrow = cv2.flip(img, 1)
                width, height = mirrow.shape[:2]  # 行:宽，列:高

                # 显示图片
                image_show = cv2.cvtColor(mirrow, cv2.COLOR_BGR2RGB)  # opencv读的通道是BGR,要转成RGB
                camera_img = QtGui.QImage(image_show.data, height, width, QImage.Format_RGB888)

                self.window.label_video.setPixmap(QPixmap.fromImage(camera_img))  # 往显示视频的Label里显示QImage

                is_face, icon_feature = self.window.get_face_feature(camera_img)
                if is_face:
                    user = self.window.compare_feature(icon_feature)
                    if user:
                        self._signal_thread.emit(json.dumps({'user': user}))  # 把人脸信息传回主进程
                        time.sleep(1)
                    else:
                        self._signal_thread.emit(json.dumps({'info': '人脸未录入！'}))
                else:
                    self._signal_thread.emit(json.dumps({'info': '获取人脸失败，请打开摄像头，对准人脸！'}))
            except:
                pass

        # 释放摄像头 release camera
        self.cap.release()
        self.cap = None

    def run(self):
        self.is_exit_run = False
        print('人脸识别进入循环，打开摄像头')
        self.open_login_camera()
        print('人脸识别已退出循环，关闭摄像头')

    def stop(self):
        # 改变线程状态与终止
        self.is_exit_run = True
        self.wait()
```



#### 6.5、效果

![](D:\我的\学习\我的博客\imgs\人脸识别打卡系统.png)



































