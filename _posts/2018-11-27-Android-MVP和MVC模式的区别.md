---
layout: post
title: Android MVP和MVC模式的区别
key: 20181127 MVP和MVC模式的区别
tags: Android MVP MVC
---

# 1.MVC模式

M 指模型层（网络IO、文件IO等操作）

V 指视图层（对应Android中的Layout或者自定义View）

C 指控制层（对应Android中的Activity/Fragment）

<!--more-->

在Android中，Activity/Fragment既充当控制层又充当视图层，这就导致了V和C这两层耦合在一起，当业务比较复杂时，Activity/Fragment文件就很庞大，导致难以维护和测试，这时就可以MVP模式

![8697741-9990956472185fb5](https://ws3.sinaimg.cn/large/006tNbRwgy1fxmv6hl8cyj309n0b075x.jpg)

下面看一个MVC模式的示例：

```
public class LoginActivity extends AppCompatActivity implements LoginModel.CallBack {

    private Button btnLogin;
    private EditText editUser;
    private EditText editPass;
    private LoginModel mModel;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_login);
        mModel = new LoginModel(this);

        btnLogin = findViewById(R.id.btnLogin);
        btnLogin.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                mModel.doLogin(editUser.getText().toString(),
                        editPass.getText().toString());

            }
        });
        editUser = findViewById(R.id.editUser);
        editPass = findViewById(R.id.editPass);
    }

    @Override
    public void onLoginResult(boolean isSuccess) {
        Toast.makeText(getApplicationContext(), isSuccess ? "login success" : "login fail",
                Toast.LENGTH_SHORT).show();
    }
}

public class LoginModel implements ILoginModel {
    private CallBack mCallBack;

    public LoginModel(CallBack callBack) {
        mCallBack = callBack;
    }

    @Override
    public void doLogin(String user, String pass) {
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        mCallBack.onLoginResult(true);
    }

    interface CallBack {
        void onLoginResult(boolean isSuccess);
    }
}
```

# 2.MVP模式

M 指模型层（网络IO、文件IO等操作）

V 指视图层（Activity）

P 指业务层（业务逻辑）

Activity/Fragment只充当视图层，不做任何的业务逻辑，将业务逻辑全部放在业务层，由Presenter和Model进行交互，避免Model直接操作View。MVP的优点：将业务从Activity/Fragment分离，便于后期维护和测试。MVP使用特点是面向接口编程（View/Presenter/Model都定义一套接口）。

![8697741-d3b1643733d68629](https://ws3.sinaimg.cn/large/006tNbRwgy1fxmvbhn1c5j309s0b1gnc.jpg)

示例代码：

```
public class LoginActivity extends AppCompatActivity implements ILoginView {

    private Button btnLogin;
    private EditText editUser;
    private EditText editPass;

    private ILoginPresenter mPresenter;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_login);
        mPresenter = new LoginPresenter(this);

        btnLogin = findViewById(R.id.btnLogin);
        btnLogin.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                mPresenter.doLogin(editUser.getText().toString(),
                        editPass.getText().toString());
            }
        });
        editUser = findViewById(R.id.editUser);
        editPass = findViewById(R.id.editPass);
    }

    @Override
    public void onLoginResult(boolean isSuccess) {
        Toast.makeText(getApplicationContext(), isSuccess ? "login success" : "login fail",
                Toast.LENGTH_SHORT).show();
    }

public class LoginPresenter implements ILoginPresenter {
    private ILoginModel mModel;
    private ILoginView mView;

    public LoginPresenter(ILoginView view) {
        mModel = new LoginModel();
        mView = view;
    }

    @Override
    public void doLogin(String user, String pass) {
        boolean ret = mModel.reqLogin(user, pass);
        mView.onLoginResult(ret);
    }
}

public class LoginModel implements ILoginModel {
    @Override
    public boolean reqLogin(String user, String pass) {
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return true;
    }
}
```



