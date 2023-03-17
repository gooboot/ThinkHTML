# Web前端思考笔记

## 包含块

什么是包含块？



```java
package com.gemmy;

import android.Manifest;
import android.content.Intent;
import android.net.Uri;
import android.os.Bundle;
import android.os.Handler;
import android.os.Message;
import android.view.WindowManager;

import com.common.CacheHandler;
import com.common.Configs;
import com.common.DataLoader;
import com.common.LocationService;
import com.common.api.ApiManager;
import com.common.api.ApiService;
import com.common.api.ApiType;
import com.common.api.CourierApiService;
import com.gemmy.acount.LoginActivity;
import com.gemmy.bean.AppBaseConfig;
import com.gemmy.bean.Login;
import com.gemmy.bean.MaterialPic;
import com.tbruyelle.rxpermissions2.RxPermissions;
import com.utils.LogUtil;
import com.utilslibrary.Utils;
import com.utilslibrary.widget.MsgDialog;

import org.json.JSONObject;

import java.util.HashMap;
import java.util.List;

import io.reactivex.functions.Consumer;

/**
 * 启动页
 * Created by sam on 17/8/17.
 */

public class StartupActivity extends BaseActivity{
    //判断是否通过网络监听广播跳转到主页，当为true时表示网络不可用，需要通过广播跳转
    private boolean isIntentHomeByReceiver;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        //防止第一次安装退出后台后会重复打开多个实例的问题
        if ((getIntent().getFlags() & Intent.FLAG_ACTIVITY_BROUGHT_TO_FRONT) != 0) {
            finish();
            return;
        }
        super.onCreate(savedInstanceState);
        this.getWindow().setFlags(WindowManager.LayoutParams.FLAG_FULLSCREEN, WindowManager.LayoutParams.FLAG_FULLSCREEN);//去掉信息栏
        setContentView(R.layout.activity_startup);
        this.isIntentHomeByReceiver = !Utils.isNetworkAvailable(this);

        new RxPermissions(this).request(Manifest.permission.READ_EXTERNAL_STORAGE,
                Manifest.permission.WRITE_EXTERNAL_STORAGE, Manifest.permission.ACCESS_COARSE_LOCATION,
                Manifest.permission.ACCESS_FINE_LOCATION
                ).subscribe(new Consumer<Boolean>() {
            @Override
            public void accept(Boolean aBoolean) throws Exception {
                if (aBoolean)
                    showToast("申请权限成功！");
                else
                    showToast("申请权限失败！");
            }
        });

    }

    @Override
    public void hasPermsDoAction() {
        //启动定位服务，定期上传位置
//        if(Utils.getPackNmae(this).equals(Configs.Courier_PackageName)){
//            LocationService.getInstance().startService();
//        }

        if(Utils.getPackNmae(this).equals(Configs.Courier_PackageName)
                && null != DataLoader.getInstance(mContext).getLoginInfo()
                && !Utils.isInternetAvaiable(mContext)){
            //如果是配送端，&& 已登陆 && 没有网络
            intentToHomePage();
            return;
        }

        loadOrderBgData();
        new Handler().postDelayed(new Runnable() {
            @Override
            public void run() {
                loadAppBaseConfig();
            }
        }, 1000);

        //配送端物料图
        if(Utils.getPackNmae(this).equals(Configs.Courier_PackageName)){
            ApiManager.getInstance(this).loadData(ApiService.getCourierMaterialPic(), null, null);
        }else{
            //要登录成功后才能获取数据，所以在这个地方获取物料图片将失败
            //ApiManager.getInstance(this).loadData(ApiService.getMaterialPic(), this, null);
        }
    }

    //应用配置信息
    private void loadAppBaseConfig(){
        showCustomDialog(Dialog_Normal);
        ApiManager.getInstance(this).loadData(CourierApiService.getAppBaseConfig(), this, null);
    }

    //获取订单背影颜色
    private void loadOrderBgData(){
        if(Utils.getPackNmae(this).equals(Configs.Courier_PackageName)){
            ApiManager.getInstance(this).loadData(CourierApiService.getOrderBgColor(), null, null);
        }
    }

    /**
     * 自动登陆，如果有用户信息，调登陆接口登陆
     */
    private void autoLogin(){
        JSONObject loginAccountJson = CacheHandler.getInstance().getLoginAccount(this);
        int loginType = 0;
        if(null != loginAccountJson){
            loginType = loginAccountJson.optInt("loginType",0);
        }

        if(DataLoader.getInstance(this).getLoginInfo() != null && loginType != 0){  // 自动登入
            //调登陆接口

            String account = loginAccountJson.optString("account");
            String pass = loginAccountJson.optString("pass");

            HashMap dataMap = new HashMap();
            dataMap.put("logintype",loginType);
            switch (loginType){
                case 1:// weichat
                    dataMap.put("openid",account);
                    break;
                case 3://短信验证码登录
                case 2:// common
                    dataMap.put("account",account);
                    dataMap.put("pass",pass);
                    break;
            }
            showCustomDialog(Dialog_Normal);
            ApiManager.getInstance(mContext).loadData(ApiService.getAccessLogin(dataMap), StartupActivity.this, null);
        }else {
            intentToHomePage();
        }
    }

    @Override
    public void onApiFailure(ApiType type, Object result, Object tag) {
        super.onApiFailure(type, result, tag);
        removeCustomDialog();

        if(type == ApiType.Type_AccessLogin){
            LoginActivity.deleteOauth(StartupActivity.this); // 没有登录过或者因为其他问题导致无法正常登录都要清除登录信息
            intentToHomePage();
            return;
        }

        if(type == ApiType.Type_AppBaseConfig
                && Utils.getPackNmae(this).equals(Configs.Courier_PackageName)
                && null != DataLoader.getInstance(mContext).getLoginInfo()){
            //如果是配送端，&& 已登陆 && 没有网络
            intentToHomePage();
            return;
        }

        if(result instanceof Error){
            //showTipDialog(((Error)result).getMessage());
            LogUtil.e(this.getLocalClassName(),((Error)result).getMessage());
        }

    }

    @Override
    public void onApiResult(ApiType type, Object result, Object tag) {
        super.onApiResult(type, result, tag);
        //removeDialog(Dialog_Normal);

        switch (type){
            case Type_AccessLogin:
                //自动登陆
                if(result instanceof Login){
                    intentToHomePage();
                }else{
                    showToast("登录异常");
                }
                break;
            case Type_GetMaterialPic:
                if(result instanceof MaterialPic){
                    DataLoader.getInstance(this).saveMaterialPic((MaterialPic)result);
                }
                break;
            case Type_AppBaseConfig:
                //获取基础配置信息
                //1、判断是否有应用更新
                AppBaseConfig config = null;
                if(result instanceof AppBaseConfig){
                    //保存配置信息
                    config = (AppBaseConfig)result;
                    DataLoader.getInstance(this).saveAppBaseConfig(config);
                }

                List<AppBaseConfig.Version> versionlist = config.versionlist;
                if(null != versionlist && versionlist.size() > 0 && null != versionlist.get(0)){
                    removeCustomDialog();
                    AppBaseConfig.Version version = versionlist.get(0);
                    float deviceVersion = Utils.getVersionNumber(this);//当前设备上的app版本号
                    //有更新
                    if(deviceVersion < version.versionNum){
                        if(deviceVersion < version.minNum){
                            //强制更新
                            showUpdateDialog(getString(R.string.version_force_update), true, version.versionUrl);
                            return;
                        }else{
                            //可选择更新
                            showUpdateDialog(getString(R.string.version_update), false, version.versionUrl);
                        }
                    }else{
                        //无更新,进行自动登陆
                        autoLogin();
                    }
                }else{
                    autoLogin();
                }
                break;
        }
    }

    /**
     * 提示更新
     * @param updateMsg 更新信息
     * @param isForce 是否强制更新
     */
    private void showUpdateDialog(String updateMsg, boolean isForce, final String url){
        MsgDialog msgDialog = new MsgDialog(this, null, getString(R.string.confirm),
                isForce ?  null : getString(R.string.cancel), updateMsg);
        msgDialog.setLeftBtnClickListener(new MsgDialog.LeftBtnClickListener() {
            @Override
            public void onClick() {
                try{
                    Intent intent = new Intent(Intent.ACTION_VIEW);
                    intent.setData(Uri.parse(url));
                    startActivity(intent);
                }
                catch(Exception e){
                    showTipToast(getString(R.string.version_url_error));
                }
                finish();
            }
        });
        if(!isForce){
            msgDialog.setRightClickListener(new MsgDialog.RightBtnClickListener() {
                @Override
                public void onClick() {
                    autoLogin();
                }
            });
        }

        msgDialog.showDialog();
    }

    // 跳去主页
    private void intentToHomePage(){
        if(Utils.getPackNmae(this).equals(Configs.Courier_PackageName) && Utils.isInternetAvaiable(mContext)){
            if(!locationAvailable()){
                //TODO 防止获取定位失败而进入不到主页
                //return;
            }
        }

        startActivity(new Intent(StartupActivity.this, UserGuideActivity.class));
        finish();
    }

    private boolean locationAvailable(){
//        if(!LocationService.getInstance().locationAvailable()){
//            MsgDialog msgDialog = new MsgDialog(mContext, null,
//                    getString(R.string.confirm), null, getString(R.string.location_available));
//            msgDialog.setLeftBtnClickListener(new MsgDialog.LeftBtnClickListener() {
//                @Override
//                public void onClick() {
//                    finish();
//                    LocationService.getInstance().stopService();
//                    System.exit(0);
//                }
//            });
//            msgDialog.showDialog();
//            return false;
//        }
        return true;
    }

    @Override
    public void onNetworkConnectedSuccess(Intent intent) {
        super.onNetworkConnectedSuccess(intent);
        if (this.isIntentHomeByReceiver)this.intentToHomePage();
    }
}

```

