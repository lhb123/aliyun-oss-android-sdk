## 简介

本文档主要介绍OSS Android SDK的安装和使用。本文档假设您已经开通了阿里云OSS 服务，并创建了Access Key ID 和Access Key Secret。文中的ID 指的是Access Key ID，KEY 指的是Access Key Secret。如果您还没有开通或者还不了解OSS，请登录[OSS产品主页](http://www.aliyun.com/product/oss)获取更多的帮助。

### 环境要求：
- Android系统版本：2.3 及以上
- 必须注册有Aliyun.com用户账户，并开通OSS服务。

-----
## 安装

OSS Android SDK依赖于[okhttp](https://github.com/square/okhttp)。

在项目中使用时，您可以把下载得到的jar包引入工程，也可以通过maven依赖。

### 直接引入jar包

当您下载了OSS Android SDK的zip包后，进行以下步骤(对Android studio或者Eclipse都适用):

* 在官网[点击查看](https://help.aliyun.com/document_detail/oss/sdk/sdk-download/android.html)下载sdk包
* 解压后在libs目录下得到jar包，目前包括aliyun-oss-sdk-android-2.0.3.jar、okhttp-2.7.0.jar、okio-2.6.0.jar
* 将以上3个jar包导入工程的libs目录

### Maven依赖

```
<dependency>
	<groupId>com.aliyun.dpa</groupId>
	<artifactId>oss-android-sdk</artifactId>
	<version>2.0.3</version>
</dependency>
```

### 自行从源码编译jar包

可以clone下工程源码之后，运行gradle命令打包：

```bash

# clone工程
$ git clone https://github.com/aliyun/aliyun-oss-android-sdk.git

# 进入目录
$ cd aliyun-oss-android-sdk/oss-android-sdk/

# 执行打包脚本，要求gradle 2.7+, jdk 1.7
$ gradle releaseJar

# 进入打包生成目录，jar包生成在该目录下
$ cd build/libs && ls
```

### Javadoc

[点击查看](http://aliyun.github.io/aliyun-oss-android-sdk/)

### 权限设置

以下是OSS Android SDK所需要的Android权限，请确保您的AndroidManifest.xml文件中已经配置了这些权限，否则，SDK将无法正常工作。

```
<uses-permission android:name="android.permission.INTERNET"></uses-permission>
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE"></uses-permission>
<uses-permission android:name="android.permission.READ_PHONE_STATE"></uses-permission>
<uses-permission android:name="android.permission.ACCESS_WIFI_STATE"></uses-permission>
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"></uses-permission>
```

### 对SDK中同步接口、异步接口的一些说明

考虑到移动端开发场景下不允许在UI线程执行网络请求的编程规范，SDK大多数接口都提供了同步、异步两种调用方式，同步接口调用后会阻塞等待结果返回，而异步接口需要在请求时需要传入回调函数，请求的执行结果将在回调中处理。

同步接口不能在UI线程调用。遇到异常时，将直接抛出ClientException或者ServiceException异常，前者指本地遇到的异常如网络异常、参数非法等；后者指OSS返回的服务异常，如鉴权失败、服务器错误等。

异步请求遇到异常时，异常会在回调函数中处理。

此外，调用异步接口时，函数会直接返回一个Task，Task可以取消、等待直到完成、或者直接获取结果。如：

```java
OSSAsyncTask task = oss.asyncGetObject(...);

task.cancel(); // 可以取消任务

task.waitUntilFinished(); // 等待直到任务完成

GetObjectResult result = task.getResult(); // 阻塞等待结果返回
```

-----
## 快速入门

以下演示了上传、下载文件的基本流程。更多细节用法可以参考本工程的：

test目录：[点击查看](https://github.com/aliyun/aliyun-oss-android-sdk/tree/master/oss-android-sdk/src/androidTest/java/com/alibaba/sdk/android)

或者：

sample目录: [点击查看](https://github.com/aliyun/aliyun-oss-android-sdk/tree/master/app)。

### STEP-1. 初始化OSSClient

初始化主要完成Endpoint设置、鉴权方式设置、Client参数设置。其中，鉴权方式包含明文设置模式、自签名模式、STS鉴权模式。鉴权细节详见后面的`访问控制`章节。

```java
String endpoint = "http://oss-cn-hangzhou.aliyuncs.com";

// 明文设置secret的方式建议只在测试时使用，更多鉴权模式请参考后面的`访问控制`章节
OSSCredentialProvider credentialProvider = new OSSPlainTextAKSKCredentialProvider("<accessKeyId>", "<accessKeySecret>");

OSS oss = new OSSClient(getApplicationContext(), endpoint, credentialProvider);
```

### STEP-2. 上传文件

这里假设您已经在控制台上拥有自己的Bucket，这里演示如何从把一个本地文件上传到OSS:

```java
// 构造上传请求
PutObjectRequest put = new PutObjectRequest("<bucketName>", "<objectKey>", "<uploadFilePath>");

// 异步上传时可以设置进度回调
put.setProgressCallback(new OSSProgressCallback<PutObjectRequest>() {
	@Override
	public void onProgress(PutObjectRequest request, long currentSize, long totalSize) {
		Log.d("PutObject", "currentSize: " + currentSize + " totalSize: " + totalSize);
	}
});

OSSAsyncTask task = oss.asyncPutObject(put, new OSSCompletedCallback<PutObjectRequest, PutObjectResult>() {
	@Override
	public void onSuccess(PutObjectRequest request, PutObjectResult result) {
		Log.d("PutObject", "UploadSuccess");
	}

	@Override
	public void onFailure(PutObjectRequest request, ClientException clientExcepion, ServiceException serviceException) {
		// 请求异常
		if (clientExcepion != null) {
			// 本地异常如网络异常等
			clientExcepion.printStackTrace();
		}
		if (serviceException != null) {
			// 服务异常
			Log.e("ErrorCode", serviceException.getErrorCode());
			Log.e("RequestId", serviceException.getRequestId());
			Log.e("HostId", serviceException.getHostId());
			Log.e("RawMessage", serviceException.getRawMessage());
		}
	}
});

// task.cancel(); // 可以取消任务

// task.waitUntilFinished(); // 可以等待直到任务完成
```

### STEP-3. 下载指定文件

下载一个指定`object`，返回数据的输入流，需要自行处理:

```java
// 构造下载文件请求
GetObjectRequest get = new GetObjectRequest("<bucketName>", "<objectKey>");

OSSAsyncTask task = oss.asyncGetObject(get, new OSSCompletedCallback<GetObjectRequest, GetObjectResult>() {
	@Override
	public void onSuccess(GetObjectRequest request, GetObjectResult result) {
		// 请求成功
		Log.d("Content-Length", "" + getResult.getContentLength());

		InputStream inputStream = result.getObjectContent();

		byte[] buffer = new byte[2048];
		int len;

		try {
			while ((len = inputStream.read(buffer)) != -1) {
				// 处理下载的数据
			}
		} catch (IOException e) {
			e.printStackTrace();
		}
	}

	@Override
	public void onFailure(GetObjectRequest request, ClientException clientExcepion, ServiceException serviceException) {
		// 请求异常
		if (clientExcepion != null) {
			// 本地异常如网络异常等
			clientExcepion.printStackTrace();
		}
		if (serviceException != null) {
			// 服务异常
			Log.e("ErrorCode", serviceException.getErrorCode());
			Log.e("RequestId", serviceException.getRequestId());
			Log.e("HostId", serviceException.getHostId());
			Log.e("RawMessage", serviceException.getRawMessage());
		}
	}
});

// task.cancel(); // 可以取消任务

// task.waitUntilFinished(); // 如果需要等待任务完成

```

-----
## 完整文档

SDK提供进阶的上传、下载功能、断点续传，以及文件管理、Bucket管理等功能。详见官方完整文档：[点击查看]()

-----
## 联系我们

* 阿里云OSS官方网站：http://oss.aliyun.com
* 阿里云OSS官方论坛：http://bbs.aliyun.com
* 阿里云OSS官方文档中心：http://www.aliyun.com/product/oss#Docs
* 阿里云官方技术支持 登录OSS控制台 https://home.console.aliyun.com -> 点击"工单系统"

-----
## License

Copyright (c) 2015 Aliyun.Inc.

Licensed under the Apache License, Version 2.0 (the "License");

you may not use this file except in compliance with the License.

You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software

distributed under the License is distributed on an "AS IS" BASIS,

WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.

See the License for the specific language governing permissions and

limitations under the License.
