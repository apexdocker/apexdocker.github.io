---
title: "U2自动安装APK"
date: 2020-11-25T17:23:32+08:00
tags: ['自动化']
---

```
import uiautomator2 as u2
import apkutils
from loguru import logger

class AndroidClient(object):
    d:u2
    @classmethod
    def start(cls, udid):
        cls.d = u2.connect(udid)
        cls.d.healthcheck()
        cls.d.implicitly_wait(5)
        cls.d.set_fastinput_ime(True)
        return cls.d

class Base(object):
    def __init__(self):
        self.d = AndroidClient.d

class ApkInstall(Base):
    def watch_device(self, keyword):
        '''
        如果存在元素则自动点击
        :param keyword: exp: keyword="yes|允许|好的|跳过"
        '''
        for i in keyword.split("|"):
            self.d.watcher.when(i).click()
        self.d.watcher.start(1.0)
        time.sleep(2)

    def unwatch_device(self):
        '''关闭watcher '''
        self.d.watcher.stop()
        time.sleep(2)
    
    def get_apk_info(self, path):
        '''从apk获取信息
        '''
        tmp = apkutils.APK(path).get_manifest()
        info = {}
        info['versionCode'] = str(tmp.get('@android:versionCode'))
        info['versionName'] = str(tmp.get('@android:versionName'))
        info['package'] = str(tmp.get('@package'))
        return info

    def local_install(self, apk_path):
        '''
        安装本地apk 覆盖安装，不需要usb链接
        :param apk_path: apk文件本地路径
        '''
        pkg_name = self.get_apk_info(apk_path)['package']
        file_name = os.path.basename(apk_path)[:12] + '.apk'
        logger.info('start install %s' % file_name)

        self.d.shell('rm -rf /sdcard/*.apk')
        self.d.shell('rm -rf /sdcard/apks/*.apk')
        self.d.shell(['/data/local/tmp/atx-agent', 'server', '-d'])
        if self.d.device_info['brand'] == 'vivo':
            '''Vivo 手机通过打开文件管理 安装app'''
            dst = '/sdcard/apks/' + file_name
            self.d.push(apk_path, dst)
            self.d.app_stop('com.android.packageinstaller')
            self.d.app_stop('com.android.filemanager')
            self.d.app_start('com.android.filemanager')
            time.sleep(4)
            if self.d(text="本地文件").exists(timeout=3):
                self.d(text="本地文件").click()
            if self.d(text="手机存储").exists(timeout=3):
                self.d(text="手机存储").click()
            time.sleep(1)
            # self.d(scrollable=True).scroll.toEnd()
            # time.sleep(1)
            for i in range(10):
                if self.d(textContains="apks").exists:
                    self.d(textContains="apks").click()
                    time.sleep(1)
                    break
                else:
                    # self.d(scrollable=True).scroll.vert.forward(steps=100)
                    self.d.swipe_ext("up", scale=0.8)
                    time.sleep(1)
            for i in range(40):
                if self.d(textContains=file_name).exists:
                    self.d(textContains=file_name).click()
                    time.sleep(5)
                    break
                else:
                    # self.d(scrollable=True).scroll.vert.forward(steps=100)
                    self.d.swipe_ext("up", scale=0.8)
                    print(i)
                    time.sleep(1)

            # self.d(text=u"继续安装").click(timeout=15)
            if self.d(resourceId="android:id/button1").exists:
                self.d(resourceId="android:id/button1").click()
            else:
                self.d(resourceId="com.android.packageinstaller:id/continue_button").click()
                self.d(resourceId="com.android.packageinstaller:id/ok_button").click()
            time.sleep(8)

        elif self.d.device_info['brand'] == 'OPPO':
            dst = '/sdcard/apks/' + file_name
            self.d.push(apk_path, dst)
            self.d.app_stop('com.coloros.filemanager')
            self.d.app_start('com.coloros.filemanager')
            time.sleep(4)
            if self.d(text="手机存储").exists(timeout=3):
                self.d(text="手机存储").click()
            if self.d(text="所有文件").exists(timeout=3):
                self.d(text="所有文件").click()
            time.sleep(1)
            self.d(scrollable=True).scroll.to(textContains="apks")
            time.sleep(2)
            self.d(textContains="apks").click()
            time.sleep(2)
            # self.d(scrollable=True).scroll.to(textContains=file_name)
            # time.sleep(2)
            self.d(textContains=file_name).click()
            time.sleep(5)
            btn_done = self.d(className="android.widget.Button", text="完成")
            while not btn_done.exists:
                self.d(text="继续安装旧版本").click_exists()
                self.d(text="重新安装").click_exists()
                if self.d(resourceId="com.android.packageinstaller:id/install_confirm_panel").exists:
                    # R11Plus
                    if self.d(resourceId="com.android.packageinstaller:id/cancel_button").exists:
                        self.d(resourceId="com.android.packageinstaller:id/bottom_button_layout") \
                            .click(offset=(0.5, 0.3))
                    # R9s
                    elif self.d(resourceId="com.android.packageinstaller:id/bottom_button_layout").exists:
                        self.d(resourceId="com.android.packageinstaller:id/bottom_button_layout") \
                            .click(offset=(0.75, 0.5))
                    # R17
                    elif not self.d(resourceId="com.android.packageinstaller:id/bottom_button_layout").exists:
                        self.d(resourceId="com.android.packageinstaller:id/install_confirm_panel") \
                            .click(offset=(0.5, 0.95))
                time.sleep(3)
            btn_done.click()

        else:
            dst = '/data/local/tmp/' + file_name
            self.d.push(apk_path, dst)
            self.watch_device('允许|继续安装|允许安装|始终允许|安装|重新安装')
            r = self.d.shell(['pm', 'install', '-r', dst], stream=True)
            id = r.text.strip()
            print(time.strftime('%H:%M:%S'), id)
            self.unwatch_device()
        packages = list(map(lambda p: p.split(':')[1], self.d.shell('pm list packages').output.splitlines()))
        if pkg_name in packages:
            self.d.shell(['rm', dst])
        else:
            raise Exception('%s 安装失败' % apk_path)

    def install_apk(self):
        apk = 'xxx.apk'
        pkg_name = self.get_apk_info(apk)['package']
        self.d.app_stop(pkg_name)
        self.d.app_uninstall(pkg_name)
        self.local_install(apk)
        self.d.app_start(pkg_name)
        time.sleep(2)

        # 授权
        self.d.shell('pm grant {} android.permission.READ_PHONE_STATE'.format(pkg_name))
        self.d.shell('pm grant {} android.permission.READ_EXTERNAL_STORAGE'.format(pkg_name))
        self.d.shell('pm grant {} android.permission.ACCESS_COARSE_LOCATION'.format(pkg_name))
        self.d.shell('pm grant {} android.permission.ACCESS_FINE_LOCATION'.format(pkg_name))
        self.d.shell('pm grant {} android.permission.CAMERA'.format(pkg_name))
        self.d.shell('pm grant {} android.permission.RECORD_AUDIO'.format(pkg_name))
        self.d.shell('pm grant {} android.permission.WRITE_EXTERNAL_STORAGE'.format(pkg_name))
        self.d.shell('pm grant {} android.permission.READ_CONTACTS'.format(pkg_name))
```
