domo地址：https://github.com/BuglyDevTeam/Bugly-Android-Demo

集成步骤：

1.工程根目录下“build.gradle”文件中添加：

	buildscript {
	    repositories {
		jcenter()
	    }
	    dependencies {
		classpath 'com.android.tools.build:gradle:2.2.3'
		// 只需配置tinker-support插件依赖，无需再依赖tinker插件
		classpath 'com.tencent.bugly:tinker-support:1.0.8'
	    }
	}

2.在app module的“build.gradle”文件中添加（示例配置）：

	dependencies {

	    // 多dex配置
	    compile 'com.android.support:multidex:1.0.1'
	    // 远程依赖集成方式（推荐）
	    compile 'com.tencent.bugly:crashreport_upgrade:1.3.1'
	}
	// 依赖插件脚本
	apply from: 'tinker-support.gradle'

3.在app目录下创建tinker-support.gradle这个文件


	apply plugin: 'com.tencent.bugly.tinker-support'

	def bakPath = file("${buildDir}/bakApk/")

	/**
	 * 此处填写每次构建生成的基准包目录  **注意每次构建基准包或生成差分包时修改。。
	 */
	def baseApkDir = "app-0717-11-31-54"

	/**
	 * 对于插件各参数的详细解析请参考
	 */
	tinkerSupport {

    // 开启tinker-support插件，默认值true
    enable = true

    // 自动生成tinkerId, 你无须关注tinkerId，默认为false
    autoGenerateTinkerId = true

    // 指定归档目录，默认值当前module的子目录tinker
    autoBackupApkDir = "${bakPath}"

    // 是否启用覆盖tinkerPatch配置功能，默认值false
    // 开启后tinkerPatch配置不生效，即无需添加tinkerPatch
    overrideTinkerPatchConfiguration = true

    // 编译补丁包时，必需指定基线版本的apk，默认值为空
    // 如果为空，则表示不是进行补丁包的编译
    // @{link tinkerPatch.oldApk }
    baseApk =  "${bakPath}/${baseApkDir}/app-release.apk"

    // 对应tinker插件applyMapping
    baseApkProguardMapping = "${bakPath}/${baseApkDir}/app-release-mapping.txt"

    // 对应tinker插件applyResourceMapping
    baseApkResourceMapping = "${bakPath}/${baseApkDir}/app-release-R.txt"

    // 构建基准包跟补丁包都要修改tinkerId，主要用于区分
    //  tinkerId = "1.0.3-patch"

    // 打多渠道补丁时指定目录
    // buildAllFlavorsDir = "${bakPath}/${baseApkDir}"

    // 是否使用加固模式，默认为false
    // isProtectedApp = true

    // 是否采用反射Application的方式集成，无须改造Application
    enableProxyApplication = true

	}

	/**
	 * 一般来说,我们无需对下面的参数做任何的修改
	 * 对于各参数的详细介绍请参考:
	 * https://github.com/Tencent/tinker/wiki/Tinker-%E6%8E%A5%E5%85%A5%E6%8C%87%E5%8D%97
	 */
	tinkerPatch {
	    tinkerEnable = true
	    ignoreWarning = false
	    useSign = true
	    dex {
		dexMode = "jar"
		pattern = ["classes*.dex"]
		loader = []
	    }
	    lib {
		pattern = ["lib/*/*.so"]
	    }

	    res {
		pattern = ["res/*", "r/*", "assets/*", "resources.arsc", "AndroidManifest.xml"]
		ignoreChange = []
		largeModSize = 100
	    }

	    packageConfig {
	    }
	    sevenZip {
		zipArtifact = "com.tencent.mm:SevenZip:1.1.10"
	//        path = "/usr/local/bin/7za"
	    }
	    buildConfig {
		keepDexApply = false
	//      tinkerId = "base-2.0.1"
	    }
	}


4.定义自己的Application类并集成buglysdk

    public class MyApp extends Application {

    @Override
    public void onCreate() {
        super.onCreate();

        setTinker();
    }

    @Override
    protected void attachBaseContext(Context base) {
        super.attachBaseContext(base);
        // you must install multiDex whatever tinker is installed!
        MultiDex.install(base);

        // 安装tinker
        Beta.installTinker();
    }

    private void setTinker() {
        // 设置是否开启热更新能力，默认为true
        Beta.enableHotfix = true;
        // 设置是否自动下载补丁
        Beta.canAutoDownloadPatch = true;
        // 设置是否提示用户重启
        Beta.canNotifyUserRestart = true;
        // 设置是否自动合成补丁
        Beta.canAutoPatch = true;

        /**
         * 补丁回调接口，可以监听补丁接收、下载、合成的回调
         */
        Beta.betaPatchListener = new BetaPatchListener() {
            @Override
            public void onPatchReceived(String patchFileUrl) {
//                Toast.makeText(getApplicationContext(), patchFileUrl, Toast.LENGTH_SHORT).show();
            }

            @Override
            public void onDownloadReceived(long savedLength, long totalLength) {
		//                Toast.makeText(getApplicationContext(), String.format(Locale.getDefault(),
		//                        "%s %d%%",
		//                        Beta.strNotificationDownloading,
		//                        (int) (totalLength == 0 ? 0 : savedLength * 100 / totalLength)), Toast.LENGTH_SHORT).show();
            }

            @Override
            public void onDownloadSuccess(String patchFilePath) {
	//                Toast.makeText(getApplicationContext(), patchFilePath, Toast.LENGTH_SHORT).show();
                Beta.applyDownloadedPatch();
            }

            @Override
            public void onDownloadFailure(String msg) {
	//                Toast.makeText(getApplicationContext(), msg, Toast.LENGTH_SHORT).show();
            }

            @Override
            public void onApplySuccess(String msg) {
	//                Toast.makeText(getApplicationContext(), msg, Toast.LENGTH_SHORT).show();
            }

            @Override
            public void onApplyFailure(String msg) {
//                Toast.makeText(getApplicationContext(), msg, Toast.LENGTH_SHORT).show();
            }

            @Override
            public void onPatchRollback() {
	//                Toast.makeText(getApplicationContext(), "onPatchRollback", Toast.LENGTH_SHORT).show();
            }
		    };
			// 这里实现SDK初始化，appId替换成你的在Bugly平台申请的appId,调试时将第三个参数设置为true
			Bugly.init(this, "aa20469483", true);

	    }
	}


5. 清单文件添加权限和组件

		<----------------------------添加-------------------------------------->

		<uses-permission android:name="android.permission.READ_PHONE_STATE" />
		<uses-permission android:name="android.permission.INTERNET" />
		<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
		<uses-permission android:name="android.permission.ACCESS_WIFI_STATE" />
		<uses-permission android:name="android.permission.READ_LOGS" />
		<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />

		<------------------------------------------------------------------>

	    <application
		android:allowBackup="true"
		android:icon="@mipmap/ic_launcher"
		android:label="@string/app_name"
		android:roundIcon="@mipmap/ic_launcher_round"
		android:supportsRtl="true"
		android:name=".MyApp" //**修改为自己定义的application类
		android:theme="@style/AppTheme">
		<activity android:name=".MainActivity">
		    <intent-filter>
			<action android:name="android.intent.action.MAIN" />

			<category android:name="android.intent.category.LAUNCHER" />
		    </intent-filter>
		</activity>

			<----------------------------添加-------------------------------------->

		<activity
		    android:name="com.tencent.bugly.beta.ui.BetaActivity"
		    android:configChanges="keyboardHidden|orientation|screenSize|locale"
		    android:theme="@android:style/Theme.Translucent" />

		<provider
		    android:name="android.support.v4.content.FileProvider"
		    android:authorities="${applicationId}.fileProvider"
		    android:exported="false"
		    android:grantUriPermissions="true">
		    <meta-data
			android:name="android.support.FILE_PROVIDER_PATHS"
			android:resource="@xml/provider_paths"/>
		</provider>

			<------------------------------------------------------------------>
	    </application>


6.在res目录创建xml目录，新建provider_paths.xml文件

	<?xml version="1.0" encoding="utf-8"?>
	<paths xmlns:android="http://schemas.android.com/apk/res/android">
	    <!-- /storage/emulated/0/Download/com.bugly.upgrade.demo/.beta/apk-->
	    <external-path name="beta_external_path" path="Download/"/>
	    <!--/storage/emulated/0/Android/data/com.bugly.upgrade.demo/files/apk/-->
	    <external-path name="beta_external_files_path" path="Android/data/"/>
	</paths>

7.根据基准包生成差分包，在bugly平台上下发，完成热修复流程。


