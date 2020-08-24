## 关于OpenALPR
OpenALPR是一种使用C ++编写的开源自动车牌识别库，支持多个国家多个地区的车牌。而最近公司想做车牌识别这一块业务，要支持全球多个国家的车牌，于是写了个Demo测试OpenAlpr的接口（有2000次免费机会）。
## 功能实现
```
import android.app.Activity;
import android.content.ContentValues;
import android.content.Intent;
import android.graphics.Bitmap;
import android.graphics.BitmapFactory;
import android.net.Uri;
import android.os.Build;
import android.os.Bundle;
import android.os.Environment;
import android.provider.MediaStore;
import android.support.v7.app.AppCompatActivity;
import android.text.format.DateFormat;
import android.view.View;
import android.widget.AdapterView;
import android.widget.ArrayAdapter;
import android.widget.Button;
import android.widget.ImageView;
import android.widget.Spinner;
import android.widget.TextView;
import android.widget.Toast;

import com.openalpr.api.DefaultApi;
import com.openalpr.api.invoker.ApiException;
import com.openalpr.api.models.InlineResponse200;
import com.yalantis.ucrop.UCrop;
import com.yalantis.ucrop.UCropActivity;

import java.io.File;
import java.io.FileInputStream;
import java.io.IOException;
import java.util.Calendar;
import java.util.Locale;

public class MainActivity extends AppCompatActivity {
    private final DefaultApi api = new DefaultApi();
    private static final String TAG = "MainActivity";
    Button reconize,select;
    TextView result,plate;
    ImageView showImage;
    Spinner spinner;
    String secretKey = "sk_960d6f4c62d1c20452c7613e";
    String url = "https://qnwww2.autoimg.cn/youchuang/g6/M0A/FA/29/autohomecar__wKgH3FkZCr-AFH7zAAFPFgtP6pc751.jpg?imageView2/2/w/752|watermark/2/text/TEkzMw0K5rG96L2m5LmL5a62/font/5b6u6L2v6ZuF6buR/fontsize/270/fill/d2hpdGU=/dissolve/100/gravity/SouthEast/dx/5/dy/5";
    File cameraFile,cropFile;
    private boolean isFromfile = false;
    String country = "us";
    final Integer recognizeVehicle = 0;
    final String state = "";
    final Integer returnImage = 1;
    final Integer topn = 10;
    final String prewarp = "";
    private String cropPath = "";
    String[] countrys ;
    String[] countryCodes ;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        reconize = (Button) findViewById(R.id.reconize);
        select = (Button) findViewById(R.id.selectImage);
        result = (TextView) findViewById(R.id.result);
        showImage = (ImageView) findViewById(R.id.imageShow);
        plate = (TextView) findViewById(R.id.plate);
        spinner = (Spinner) findViewById(R.id.spinner);
//        GlideUtils.setUrlImage(this,url,showImage);
        countrys = getResources().getStringArray(R.array.countrys);
        countryCodes = getResources().getStringArray(R.array.country_codes);

        select.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                String filename = DateFormat.format("yyyyMMdd_hhmmss", Calendar.getInstance(Locale.CHINA)) + ".jpg";
                cameraFile = new File(Environment.getExternalStorageDirectory() + "/Pictures/Camera/" + filename);
                if(!cameraFile.exists()){
                    try {
                        cameraFile.createNewFile();
                    } catch (IOException e) {
                        e.printStackTrace();
                    }
                }
                Intent intent0 = new Intent(MediaStore.ACTION_IMAGE_CAPTURE);

                if (Build.VERSION.SDK_INT < 24) {
                    intent0 = new Intent(MediaStore.ACTION_IMAGE_CAPTURE);
                    intent0.putExtra("android.intent.extras.CAMERA_FACING", 0);
                    intent0.putExtra(MediaStore.EXTRA_OUTPUT, Uri.fromFile(cameraFile));
                } else {
                    ContentValues contentValues = new ContentValues(1);
                    intent0.putExtra("android.intent.extras.CAMERA_FACING", 0);
                    contentValues.put(MediaStore.Images.Media.DATA, cameraFile.getAbsolutePath());
                    Uri uri = MainActivity.this.getContentResolver().insert(MediaStore.Images.Media.EXTERNAL_CONTENT_URI, contentValues);
                    intent0.putExtra(MediaStore.EXTRA_OUTPUT, uri);
                }
                isFromfile = true;
				startActivityForResult(intent0, 200);
            }
        });
        reconize.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                result.setText("正在识别。。。。。");
                plate.setText("");
                try {
                    new Thread(){
                        @Override
                        public void run() {
                            super.run();
                            InlineResponse200 response = null;
                            try {
                                if(isFromfile){
                                    response = api.recognizeFile(cropFile, secretKey, country, recognizeVehicle, state, returnImage, topn, prewarp);

                                }else {
                                    response = api.recognizeUrl(url, secretKey, country, recognizeVehicle, state, returnImage, topn, prewarp);

                                }
                            } catch (ApiException e) {
                                e.printStackTrace();
                            }
                            final InlineResponse200 finalResponse = response;
                            runOnUiThread(new Runnable() {
                                @Override
                                public void run() {
                                    if(finalResponse == null){
                                        Toast.makeText(MainActivity.this, "识别失败", Toast.LENGTH_SHORT).show();
                                        return;
                                    }
                                    Toast.makeText(MainActivity.this, "识别成功", Toast.LENGTH_SHORT).show();
                                    if(finalResponse.getResults()!=null && finalResponse.getResults().size()>=1)
                                    plate.setText("车牌号："+finalResponse.getResults().get(0).getPlate());
                                    result.setText("返回结果:"+finalResponse.toString());
                                }
                            });
                        }
                    }.start();
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        });

        ArrayAdapter<String> product_adapter = new ArrayAdapter<String>(this, android.R.layout.select_dialog_singlechoice, countrys);
        spinner.setAdapter(product_adapter);
        spinner.setSelection(12);

        spinner.setOnItemSelectedListener(new AdapterView.OnItemSelectedListener() {
            @Override
            public void onItemSelected(AdapterView<?> parent, View view, int position, long id) {
                country = countryCodes[position];
            }

            @Override
            public void onNothingSelected(AdapterView<?> parent) {

            }
        });

    }

    @Override
    protected void onActivityResult(int requestCode, int resultCode, Intent data) {
        super.onActivityResult(requestCode, resultCode, data);
        if(resultCode != RESULT_OK){
            return;
        }
        switch (requestCode) {
            case 200:

                if (!cameraFile.exists()) {
                    return ;
                }
                Bitmap bitmap = null;
                if (resultCode== Activity.RESULT_OK){
                        try {
                            FileInputStream fis = new FileInputStream(cameraFile);
                            bitmap = BitmapFactory.decodeStream(fis);
                            fis.close();
                        } catch (Exception e) {
                            e.printStackTrace();
                        }
                        if (bitmap!=null) {
                            showImage.setImageBitmap(bitmap);
                        }
                    cropPath = startUCrop(this,cameraFile.getPath(),UCrop.REQUEST_CROP,16,9);
                }
                break;
            case UCrop.REQUEST_CROP:
                final Uri resultUri = UCrop.getOutput(data);
                showImage.setImageURI(resultUri);
                cropFile = new File(cropPath);
                break;

            default:
                break;
        }
    }

    /**
     * 启动裁剪
     * @param activity 上下文
     * @param sourceFilePath 需要裁剪图片的绝对路径
     * @param requestCode 比如：UCrop.REQUEST_CROP
     * @param aspectRatioX 裁剪图片宽高比
     * @param aspectRatioY 裁剪图片宽高比
     * @return
     */
    public static String startUCrop(Activity activity, String sourceFilePath,
                                    int requestCode, float aspectRatioX, float aspectRatioY) {
        Uri sourceUri = Uri.fromFile(new File(sourceFilePath));
        File outDir = Environment.getExternalStoragePublicDirectory(Environment.DIRECTORY_PICTURES);
        if (!outDir.exists()) {
            outDir.mkdirs();
        }
        File outFile = new File(outDir, System.currentTimeMillis() + ".jpg");
        //裁剪后图片的绝对路径
        String cameraScalePath = outFile.getAbsolutePath();
        Uri destinationUri = Uri.fromFile(outFile);
        //初始化，第一个参数：需要裁剪的图片；第二个参数：裁剪后图片
        UCrop uCrop = UCrop.of(sourceUri, destinationUri);
        //初始化UCrop配置
        UCrop.Options options = new UCrop.Options();
        //设置裁剪图片可操作的手势
        options.setAllowedGestures(UCropActivity.SCALE, UCropActivity.ROTATE, UCropActivity.ALL);
        //是否隐藏底部容器，默认显示
        options.setHideBottomControls(true);
//        //设置toolbar颜色
//        options.setToolbarColor(ActivityCompat.getColor(activity, R.color.colorPrimary));
//        //设置状态栏颜色
//        options.setStatusBarColor(ActivityCompat.getColor(activity, R.color.colorPrimary));
        //是否能调整裁剪框
        options.setFreeStyleCropEnabled(true);
        //UCrop配置
        uCrop.withOptions(options);
        //设置裁剪图片的宽高比，比如16：9
        uCrop.withAspectRatio(aspectRatioX, aspectRatioY);
        //uCrop.useSourceImageAspectRatio();
        //跳转裁剪页面
        uCrop.start(activity, requestCode);
        return cameraScalePath;
    }

}
```
主要分为拍照，裁剪（主要是为了区域更小识别更准确），选取区域，上传图片，获取解析结果。

![主要界面](https://img-blog.csdnimg.cn/20181218201735688.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mzg0NzA5Mw==,size_16,color_FFFFFF,t_70)
## 测试结果
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181218202101851.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mzg0NzA5Mw==,size_16,color_FFFFFF,t_70)
识别速度这一块与图片体积有关，识别准确率主要跟地区选取和图像清晰度这一块有关，总体来说功能非常强大，支持多个地区，而且识别率还非常高，支持的地区主要有
```
        <item>阿根廷</item>
        <item>澳大利亚</item>
        <item>巴西</item>
        <item>中国</item>
        <item>欧洲</item>
        <item>英国</item>
        <item>印度</item>
        <item>印度尼西亚</item>
        <item>日本</item>
        <item>韩国</item>
        <item>中东</item>
        <item>新西兰</item>
        <item>北美</item>
        <item>俄罗斯</item>
        <item>沙特阿拉伯</item>
        <item>南非</item>
        <item>泰国</item>
        <item>阿拉伯联合酋长国</item>
 对应的code值为
        <item>ar</item>
        <item>au</item>
        <item>br</item>
        <item>cn</item>
        <item>eu</item>
        <item>en</item>
        <item>in</item>
        <item>id</item>
        <item>jp</item>
        <item>kr</item>
        <item>me</item>
        <item>nz</item>
        <item>us</item>
        <item>ru</item>
        <item>sa</item>
        <item>za</item>
        <item>th</item>
        <item>ae</item>
```
代码地址
[GitHub地址](https://github.com/Participanta/OpenAlprTest)
