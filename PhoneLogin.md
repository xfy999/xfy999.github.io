java后台实现用户密码登录和手机短信登录 

1.账号密码登录:获取用户名、密码，检验是否存在该账号，以及该账号是否有效（未冻结、未删除），检验密码是否正确

```java 
public Result<JSONObject> login(@RequestBody SysLoginModel sysLoginModel) throws Exception {
    Result<JSONObject> result = new Result<JSONObject>();
    String username = sysLoginModel.getUsername();
    String password = sysLoginModel.getPassword();
    //update-begin--Author:scott  Date:20190805 for：暂时注释掉密码加密逻辑，有点问题
    //前端密码加密，后端进行密码解密
    //password = AesEncryptUtil.desEncrypt(sysLoginModel.getPassword().replaceAll("%2B", "\\+")).trim();//密码解密
    //update-begin--Author:scott  Date:20190805 for：暂时注释掉密码加密逻辑，有点问题

    //1. 校验用户是否有效
    SysUser sysUser = sysUserService.getUserByName(username);
    result = sysUserService.checkUserIsEffective(sysUser);
    if(!result.isSuccess()) {
        return result;
    }

    //2. 校验用户名或密码是否正确
    String userpassword = PasswordUtil.encrypt(username, password, sysUser.getSalt());
    String syspassword = sysUser.getPassword();
        if (!syspassword.equals(userpassword)) {
            result.error500("用户名或密码错误");
        return result;
    }

    //用户登录信息
    userInfo(sysUser, result);
    sysBaseAPI.addLog("用户名: " + username + ",登录成功！", CommonConstant.LOG_TYPE_1, null);

    return result;
}
```

2.短信验证码登录

2.1获取验证码：获取手机号、短信模板号-->随机产生验证码-->根据模板号发送登录模板，设置有效时间

```java
public Result<String> sms(@RequestBody JSONObject jsonObject) {
    Result<String> result = new Result<String>();
    String phone = jsonObject.get("phone").toString();
    String smsmode=jsonObject.get("smsmode").toString();
    log.info(phone);
    Object object = redisUtil.get(phone);
    if (object != null) {
        result.setMessage("验证码10分钟内，仍然有效！");
        result.setSuccess(false);
        return result;
    }

    //随机数
    String captcha = RandomUtil.randomNumbers(6);
    JSONObject obj = new JSONObject();
    obj.put("code", captcha);
    try {
        boolean b = false;
        //登录模板
        if (CommonConstant.SMS_TPL_TYPE_0.equals(smsmode)) {
            b = DySmsHelper.sendSms(phone, obj, DySmsEnum.LOGIN_TEMPLATE_CODE);
        }
        if (b == false) {
            result.setMessage("短信验证码发送失败,请稍后重试");
            result.setSuccess(false);
            return result;
        }
        //验证码10分钟内有效
        redisUtil.set(phone, captcha, 600);
        //update-begin--Author:scott  Date:20190812 for：issues#391
        //result.setResult(captcha);
        //update-end--Author:scott  Date:20190812 for：issues#391
        result.setSuccess(true);

    } catch (ClientException e) {
        e.printStackTrace();
        result.error500(" 短信接口未配置，请联系管理员！");
        return result;
    }
    return result;
}
```

2.2 开通阿里云短信服务，得到accessKeyId和accessKeySecret

```java

// TODO 此处需要替换成开发者自己的AK(在阿里云访问控制台寻找)
static  String accessKeyId;
static  String accessKeySecret;
```

2.3 查看是否存在该用户，若不存在，登录即注册，将该手机号存入数据库，检验验证码是否正确，错误输出提示信息，正确则登录成功

```java
public Result UserPhoneLogin(@Valid  @RequestBody ApiLoginModel loginModel,BindingResult bindingResult) {
    Result<JSONObject> result = new Result<JSONObject>();
    String phone = loginModel.getPhone();
    //校验用户有效性
    MemberUser memberUser = memberUserService.getUserByPhone(phone);
    if(oConvertUtils.isEmpty(memberUser)) {
        //添加
        memberUserService.add(phone);
    }
    if(!result.isSuccess()) {
        return result;
    }
    String smscode = loginModel.getCaptcha();
    Object code = redisUtil.get(phone);
    if (!smscode.equals(code)) {
        result.setMessage("手机验证码错误");
        return result;
    }
    //添加日志
    sysBaseAPI.addLog("手机号: " + memberUser.getPhone() + ",登录成功！", CommonConstant.LOG_TYPE_1, null);
    result.setMessage("登录成功");
    return result;
}
```