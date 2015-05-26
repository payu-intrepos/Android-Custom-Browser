## For Android Studio (without sdk)

######1. Download the [CustomBrowser-release.aar](https://github.com/payu-intrepos/Android-Custom-Browser/blob/master/CustomBrowser-release.aar) file from github. 

######2. Import CustomBrowser as a module. 

######3. Make sure that app has a module dependency on CustomBrowser. 

######4. Make sure that you granted permission to read sms in your AndroidManifest.xml file. 

        <uses-permission android:name="android.permission.RECEIVE_SMS" />
        <uses-permission android:name="android.permission.READ_SMS" />

######5. Assign the layout as given below to your activity

        <RelativeLayout  
            xmlns:android="http://schemas.android.com/apk/res/android"  
            android:layout_width="match_parent"  
            android:layout_height="match_parent"  
            android:gravity="center_horizontal">

            <FrameLayout
                android:id="@+id/parent"
                android:visibility="gone"
                android:layout_alignParentBottom="true"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"/>

            <WebView
                android:id="@+id/webview"
                android:layout_width="match_parent"
                android:layout_height="match_parent"/>

            <LinearLayout
                android:layout_width="match_parent"
                android:id="@+id/trans_overlay"
                android:layout_height="match_parent"
                android:orientation="horizontal"
                android:background="@drawable/background_drawable">
            </LinearLayout>
        </RelativeLayout>

**Note:**  
i. id of frame layout should be parent  
ii. default visibility of frame layout should be “gone”.  CustomBrowser will handle the same.

######6. Make sure that your Activity is able to support Fragments (v7 support library)  

######7. Inside your Activiy onCreate() function add the following (Please make sure to change R.id.webiew to the id of your webview) 

        Class.forName("com.payu.custombrowser.Bank");

        final Bank bank = new Bank() {
            @Override
            public void registerBroadcast(BroadcastReceiver broadcastReceiver, IntentFilter filter) {
                mReceiver = broadcastReceiver;
                registerReceiver(broadcastReceiver, filter);
            }

            @Override
            public void unregisterBroadcast(BroadcastReceiver broadcastReceiver) {
                if(mReceiver != null){
                    unregisterReceiver(mReceiver);
                    mReceiver = null;
                }
            }

            @Override
            public void onHelpUnavailable() {
                findViewById(R.id.parent).setVisibility(View.GONE);
                findViewById(R.id.trans_overlay).setVisibility(View.GONE);
            }

            @Override
            public void onBankError() {
                progressBarVisibilityPayuChrome(View.GONE);
                findViewById(R.id.parent).setVisibility(View.GONE);
                findViewById(R.id.trans_overlay).setVisibility(View.GONE);
            }

            @Override
            public void onHelpAvailable() {
                findViewById(R.id.parent).setVisibility(View.VISIBLE);
            }

            @Override
            public void updateSet(Set<String> urlSet,String check){
                if (urlSet != null && urlSet.size() > 0 && webviewUrl!=null && !urlSet.contains(webviewUrl)) {
                    progressBarVisibilityPayuChrome(View.GONE);
                }
                set=urlSet;
                checkValue=check;
            }

            @Override
            public void communicationError(){
                progressBarVisibilityPayuChrome(View.GONE);
            }
        };

        Bundle args = new Bundle();
        args.putInt("webView", R.id.webview);
        args.putInt("tranLayout",R.id.trans_overlay);
        String [] list =  getIntent().getExtras().getString("postData").split("&");
        HashMap<String , String> intentMap = new HashMap<String , String>();

        for (String item : list) {
            String [] list1 =  item.split("=");
            intentMap.put(list1[0], list1[1]);
        }

        if(getIntent().getExtras().containsKey("txnid")) {
            args.putString(Bank.TXN_ID, getIntent().getStringExtra("txnid"));
        } else {
            args.putString(Bank.TXN_ID, intentMap.get("txnid"));
        }

        if(getIntent().getExtras().containsKey("showCustom")) {
            args.putBoolean("showCustom", getIntent().getBooleanExtra("showCustom", false));
        }

        args.putBoolean("showCustom", true);
        bank.setArguments(args);
        findViewById(R.id.parent).bringToFront();
        getSupportFragmentManager().beginTransaction().setCustomAnimations(R.anim.fade_in,R.anim.face_out).add(R.id.parent, bank).commit();

        webView.getSettings().setJavaScriptEnabled(true);
        webView.getSettings().setDomStorageEnabled(true); webView.postUrl(Constants.PAYMENT_URL,EncodingUtils.getBytes(getIntent().getExtras().getString("postData"), "base64"));

        webView.setWebChromeClient(new PayUWebChromeClient(bank) {
            public void onProgressChanged(WebView view, int newProgress) {

                super.onProgressChanged(view, newProgress);
                progressBarVisibilityPayuChrome(View.VISIBLE);

                if(newProgress>=95) {
                    if (checkProgress)
                        progressBarVisibilityPayuChrome(View.GONE);
                }
            }
        });

        webView.setWebViewClient(new WebViewClient() {
            @Override
            public void onPageFinished(WebView view, String url) {
                super.onPageFinished(view, url);
            }

            @Override
            public void onPageStarted(WebView view, String url, Bitmap favicon)
            {
                super.onPageStarted(view, url, favicon);
                webviewUrl=url;
                if(set!=null && set.size()>0 && !set.contains(url)) {
                    checkProgress = true;
                }

                if(checkValue!=null && url.contains(checkValue)) {
                    bank.update();
                }
            }
        });


######8. Add following variable to class
        private WebView webView;
        private ProgressDialog mProgressDialog;
        private BroadcastReceiver mReceiver = null;
        private ProgressDialog progressDialog;
        private String checkValue;
        private boolean checkProgress;
        private Set<String> set;
        private String webviewUrl;

######9. Add the following function to your Activity  
        public void progressBarVisibilityPayuChrome(int visibility)
        {
            if(visibility==View.GONE || visibility==View.INVISIBLE ) {
                if(progressDialog!=null && progressDialog.isShowing())
                    progressDialog.dismiss();
            }
            else if (progressDialog==null)
            {
                progressDialog=showProgress(this);
            }
        }

        public ProgressDialog showProgress(Context context) 
        {
            LayoutInflater mInflater = LayoutInflater.from(context);
            final Drawable[] drawables = {getResources().getDrawable(R.drawable.l_icon1),
                    getResources().getDrawable(R.drawable.l_icon2),
                    getResources().getDrawable(R.drawable.l_icon3),
                    getResources().getDrawable(R.drawable.l_icon4)
            };

            View layout = mInflater.inflate(R.layout.prog_dialog, null);
            final ImageView imageView; imageView = (ImageView) layout.findViewById(R.id.imageView);
            ProgressDialog progDialog = new ProgressDialog(context, R.style.ProgressDialog);

            Timer timer = new Timer();
            timer.scheduleAtFixedRate(new TimerTask() {
                int i = -1;
                @Override
                synchronized public void run() {
                    runOnUiThread(new Runnable() {
                        @Override
                        public void run() {
                            i++;
                            if (i >= drawables.length) {
                                i = 0;
                            }
                            imageView.setImageBitmap(null);
                            imageView.destroyDrawingCache();
                            imageView.refreshDrawableState();
                            imageView.setImageDrawable(drawables[i]);
                        }
                    });

                }
            }, 0, 500);

            progDialog.show();
            progDialog.setContentView(layout);
            progDialog.setCancelable(true);
            progDialog.setCanceledOnTouchOutside(false);
            return progDialog;
        }

######10. Add the following resources in corresponding folders/files. 
	
a.In _drawable-xxhdp_  
    l_icon1.png  
    l_icon2.png  
    l_icon3.png  
    l_icon4.png  

b. In _layout_  
prog_dialog.xml

c. In _values/styles.xml_  
Add style named ProgressDialog:  

        <style name="ProgressDialog" parent="@android:style/Theme.Dialog">
            <item name="android:windowFullscreen">true</item>
            <item name="android:windowFrame">@null</item>
            <item name="android:windowBackground">@android:color/transparent</item>
            <item name="android:windowIsFloating">true</item>
            <item name="android:windowContentOverlay">@null</item>
            <item name="android:windowTitleStyle">@null</item>
            <item name="android:windowAnimationStyle">@android:style/Animation.Dialog</item>
            <item name="android:windowSoftInputMode">stateUnspecified|adjustPan</item>
            <item name="android:backgroundDimEnabled">true</item>
        </style>
	
e. In _Animation_  
fade_in.xml  
face_out.xml  
