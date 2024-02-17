---
pubDatetime: 2017-06-20
title: Android 图片拍摄一二
featured: false
draft: false
tags:
  - Android
description: 关于在Android开发中调用摄像头进行图片拍摄，保存到指定路径，通过FileProvider获取原始图片以及提供图片给其它使用方等过程的细节进行梳理。
---

关于在Android开发中调用摄像头进行图片拍摄，保存到指定路径，通过FileProvider获取原始图片以及提供图片给其它使用方等过程的细节进行梳理。

<!-- more -->

### 1. 请求Camera权限

如果拍照是app的主要功能，那Google Play就需要知道目标设备是否有摄像头。
android:required是告诉Google Play是否强制下载该app的设备需要硬件支持；如果false，则需要在运行时进行判断是否有摄像头`hasSystemFeature(PackageManager.FEATURE_CAMERA)`。

```xml
<manifest ... >
    <uses-feature android:name="android.hardware.camera"
                  android:required="true" />
    ...
</manifest>
```

### 2. 使用Camera App拍照

```java
static final int REQUEST_IMAGE_CAPTURE = 1;

private void dispatchTakePictureIntent() {
    Intent takePictureIntent = new Intent(MediaStore.ACTION_IMAGE_CAPTURE);
    if (takePictureIntent.resolveActivity(getPackageManager()) != null) {
        startActivityForResult(takePictureIntent, REQUEST_IMAGE_CAPTURE);
    }
}
```

### 3. 获取缩略图

注意：这里从"data"中获取出来的仅是照片的缩略图，可能足够满足小图展示的需求；默认情况下Android Camera App会把照片保存在外部公共存储的相册`Environment.getExternalStoragePublicDirectory(Environment.DIRECTORY_DCIM)`；此时需要声明or动态申请 `WRITE_EXTERNAL_STORAGE` 权限。
如果需要取得全尺寸的完整图片，`data.getData()` 即可得到原图的Uri，形如“`content://media/external/images/media/6363`”，绝对路径如`/storage/emulated/0/DCIM/Camera/20170516_114052.jpg`（PS：目前仅在samsung上测试）。

```java
@Override
protected void onActivityResult(int requestCode, int resultCode, Intent data) {
    if (requestCode == REQUEST_IMAGE_CAPTURE && resultCode == RESULT_OK) {
        Bundle extras = data.getExtras();
        Bitmap imageBitmap = (Bitmap) extras.get("data");
        mImageView.setImageBitmap(imageBitmap);
    }
}
```

### 4. 保存全尺寸图片

如果需要将照片保存到App的私有路径，其它App无法读取，需要给Camera指定一个完整的保存路径`getExternalFilesDir()`。4.4开始写入这个路径将不再需要手动申请`WRITE_EXTERNAL_STORAGE`权限，因为这个路径是绝对私有，对其它app来说是不可访问的。所以可以这样声明权限：

```xml
<manifest ...>
    <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"
                     android:maxSdkVersion="18" />
    ...
</manifest>
```

指定照片的保存路径，然后还需要生成一个无冲突的唯一文件名，顺手再留下一个图片路径以备后用。如下：

```java
String mCurrentPhotoPath;

private File createImageFile() throws IOException {
    // Create an image file name
    String timeStamp = new SimpleDateFormat("yyyyMMdd_HHmmss").format(new Date());
    String imageFileName = "JPEG_" + timeStamp + "_";
    File storageDir = getExternalFilesDir(Environment.DIRECTORY_PICTURES);
    File image = File.createTempFile(
        imageFileName,  /* prefix */
        ".jpg",         /* suffix */
        storageDir      /* directory */
    );

    // Save a file: path for use with ACTION_VIEW intents
    mCurrentPhotoPath = image.getAbsolutePath();
    return image;
}
```

发起Intent的方法也要调整：

```java
static final int REQUEST_TAKE_PHOTO = 1;

private void dispatchTakePictureIntent() {
    Intent takePictureIntent = new Intent(MediaStore.ACTION_IMAGE_CAPTURE);
    // Ensure that there's a camera activity to handle the intent
    if (takePictureIntent.resolveActivity(getPackageManager()) != null) {
        // Create the File where the photo should go
        File photoFile = null;
        try {
            photoFile = createImageFile();
        } catch (IOException ex) {
            // Error occurred while creating the File
            ...
        }
        // Continue only if the File was successfully created
        if (photoFile != null) {
            Uri photoURI = FileProvider.getUriForFile(this,
                                                  "com.example.android.fileprovider",
                                                  photoFile);
            takePictureIntent.putExtra(MediaStore.EXTRA_OUTPUT, photoURI);
            startActivityForResult(takePictureIntent, REQUEST_TAKE_PHOTO);
        }
    }
}
```

> 注意：这里使用的`getUriForFile(Context, String, File) `返回的是形如`content://`的URI，对于target Android 7.0（API 24）及以上的app，跨越package传递`file://`URI会抛出`FileUriExposedException`，需要统一使用**FileProvider**来处理。关于FileProvider的详细分析见附录。

首先要在manifest中配置FileProvider：

```xml
<application>
   ...
   <provider
        android:name="android.support.v4.content.FileProvider"
        android:authorities="com.example.android.fileprovider"
        android:exported="false"
        android:grantUriPermissions="true">
        <meta-data
            android:name="android.support.FILE_PROVIDER_PATHS"
            android:resource="@xml/file_paths"></meta-data>
    </provider>
    ...
</application>
```

这里要确保`android:authorities`设置的字符串与`getUriForFile(Context, String, File)`的第二个参数authorities一致。然后，如meta-data配置的一样，provider需要从资源文件 res/xml/file_paths.xml中读取路径来做为配置。

```xml
<?xml version="1.0" encoding="utf-8"?>
<paths xmlns:android="http://schemas.android.com/apk/res/android">
    <external-path name="my_images" path="Android/data/com.example.package.name/files/Pictures" />
</paths>
```

这里的path组件与`getExternalFilesDir( Environment.DIRECTORY_PICTURES)`返回的路径是相符的。[关于FileProvider的文档](https://developer.android.google.cn/reference/android/support/v4/content/FileProvider.html)

### 5. 添加照片到相册

（Don't know why在我的samsung测试机上不生效）
在创建照片的时候我们已经指定了照片的第一保存位置，这对当前app使用来说是最方便且可控的，而且保存在`getExternalFilesDir()`路径下的文件不会被media scanner扫描到。但是对于其它app来说，访问到这些照片最简单的方式就是通过系统的**Media Provider**。

```java
private void galleryAddPic() {
    Intent mediaScanIntent = new Intent(Intent.ACTION_MEDIA_SCANNER_SCAN_FILE);
    File f = new File(mCurrentPhotoPath);
    Uri contentUri = Uri.fromFile(f);
    mediaScanIntent.setData(contentUri);
    this.sendBroadcast(mediaScanIntent);
}
```

### 6. 动态调整图片

操作或显示全尺寸图对内存的损耗巨大，所以我们可以通过把jpeg图片按照目标显示视图的大小来动态的调整，以减少内存的使用。

```java
private void setPic() {
    // Get the dimensions of the View
    int targetW = mImageView.getWidth();
    int targetH = mImageView.getHeight();

    // Get the dimensions of the bitmap
    BitmapFactory.Options bmOptions = new BitmapFactory.Options();
    bmOptions.inJustDecodeBounds = true;
    BitmapFactory.decodeFile(mCurrentPhotoPath, bmOptions);
    int photoW = bmOptions.outWidth;
    int photoH = bmOptions.outHeight;

    // Determine how much to scale down the image
    int scaleFactor = Math.min(photoW/targetW, photoH/targetH);

    // Decode the image file into a Bitmap sized to fill the View
    bmOptions.inJustDecodeBounds = false;
    bmOptions.inSampleSize = scaleFactor;
    bmOptions.inPurgeable = true;

    Bitmap bitmap = BitmapFactory.decodeFile(mCurrentPhotoPath, bmOptions);
    mImageView.setImageBitmap(bitmap);
}
```

### 附录：关于File Provider

File Provider是Content Provider的子类，通过创建content:// Uri来代替file:// Uri进而提升文件访问的安全性。
Content URI允许授予临时读写权限。当一个包含Content URI的Intent发送到目标app时（也可以通过 Intent.setFlags()来添加权限）。这些权限在目标app接受该Intent的Activity active期间一致可用；如果是Service，权限在Servce Running期间可用。
相反，如果需要控制 file:/// Uri就必须修改指定路径或文件的文件系统权限，同时这个权限针对所有其它app可用，这种情况会持续到你再次修改它。这样的文件访问是完全不安全的。
FileProvider通过Content URI提供的更高安全性的文件访问控制是Android系统安全的关键部分。

#### a. 定义FileProvider

在app的manifest文件中添加`<provider>`即可，设置`android:name`属性为`android.support.v4.content.FileProvider`；`android:authorities`取决与你所控制的域名（app package name)；FileProvider不需要公开，所以`android:exported`设置为false；`android:grantUriPermissions`设置为true去允许为文件授予临时访问权限。
一个完整的FileProvider定义如下：

```java
<provider
    android:name="android.support.v4.content.FileProvider"
    android:authorities="com.mydomain.fileprovider"
    android:exported="false"
    android:grantUriPermissions="true">
    <meta-data
        android:name="android.support.FILE_PROVIDER_PATHS"
        android:resource="@xml/file_paths" />
</provider>
```

#### b. 指定可用文件

FileProvider仅对指定的路径生成Content URI；这个指定工作在xml中完成。
**在工程中添加`res/xml/file_paths.xml`**（在<provider>的<meta-data>中指定的path资源），代码如下：

```xml
<paths xmlns:android="http://schemas.android.com/apk/res/android">
    <files-path name="my_images" path="images/"/>
    <files-path name="my_docs" path="docs/"/>
</paths>
```

<paths>必须包含一个或多个子元素：

- `<files-path name="name" path="path" />`：app内部存储的`files/`子目录；与`Context.getFilesDir()`一致。
- `<cache-path name="name" path="path" />`：app内部存储的缓存子目录；与`getCacheDir()`一致。
- `<external-path name="name" path="path" />`：外部存储的根目录；与`Environment.getExternalStorageDirectory()`一致。
- `<external-files-path name="name" path="path" />`：app的外部存储根目录，与`Context#getExternalFilesDir(String type) Context.getExternalFilesDir(null)`一致。
- `<external-cache-path name="name" path="path" />`：app的外部缓存根目录，与`Context.getExternalCacheDir()`一致。

所有以上子元素都包含两个属性：

- name="name"：URI路径片段。为了安全，生成的Uri会以name值隐藏正在共享的目录的真实路径。
- path="path"：共享的真实目录。path值永远指向一个子目录，而不是特定的文件；不能通过文件名来共享指定的某个文件，也不能使用通配符来指定一部分文件。

#### c. 生成文件的Content URI

APP必须生成Content URI来与其它app共享数据。生成URI的代码如下：

```java
File imagePath = new File(Context.getFilesDir(), "images");
File newFile = new File(imagePath, "default_image.jpg");
Uri contentUri = getUriForFile(getContext(), "com.mydomain.fileprovider", newFile);
```

APP可以将这个URI通过Intent发送给Client app，接受方通过调用` ContentResolver.openFileDescriptor`来得到一个`ParcelFileDescriptor`进而访问文件内容。具体接收方app的接收处理见Demo。
[关于ContentResolver.openFileDescriptor](https://developer.android.google.cn/reference/android/content/ContentResolver.html#openFileDescriptor(android.net.Uri, java.lang.String))

#### d. 为URI授予临时权限

为通过`getUriForFile()`得到的content URI授予访问权限：

1. Content Uri设置到Intent.setData()
2. 调用Intent.setFlags()，入参 FLAG_GRANT_READ_URI_PERMISSION or FLAG_GRANT_WRITE_URI_PERMISSION or both。
3. 发送Intent到另一个app，通常是setResult(intent)。

```java
Intent intent = new Intent();
Uri imageUri = FileProvider.getUriForFile(this, FILE_PROVIDER_AUTHORITY, new File(currentPhotoPath));
intent.setData(imageUri);
intent.setFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
setResult(RESULT_OK, intent);
```

#### e. DEMO

- [FileProvider图片拍摄及提供方示例](https://github.com/ponywei/MyFileProviderDemo)
- [FileProvider图片接受方示例](https://github.com/ponywei/MyFileProviderClient)

### 参考资料

- [“Taking Photos Simply” training class.](https://developer.android.google.cn/training/camera/photobasics.html)
