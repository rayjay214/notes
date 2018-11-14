# CGI Framework
## CGI Framework中使用到的库
- [FCGI](http://www.nongnu.org/fastcgi/)
- [FCGI Alternatives](https://stackoverflow.com/questions/1774647/fastcgi-for-c)
- [CGICC](http://www.gnu.org/software/cgicc/doc/index.html)<br>
cgicc是一个用来提取传过来的表单数据，cookie数据的工具库, 解析过程如下:<br>
```
FCGI_Processor::FCGI_Processor(void): m_queryMap()
{
    cgicc::FCgiIO IO(FCGI_Processor::fcgi_request);
    cgicc::Cgicc cgi(&IO);
    cgicc::CgiEnvironment cgiEnv = cgi.getEnvironment();
    m_httpHeader.queryURI      = cgiEnv.getQueryString();
    m_httpHeader.scriptName    = cgiEnv.getScriptName();
    m_httpHeader.contentType   = cgiEnv.getContentType();
    m_httpHeader.requestMethod = cgiEnv.getRequestMethod(); 
    m_httpHeader.remoteAddr    = cgiEnv.getRemoteAddr();
    m_httpHeader.remoteHost    = cgiEnv.getRemoteHost();
    m_httpHeader.serverName    = cgiEnv.getServerName();
    m_httpHeader.pathInfo      = cgiEnv.getPathInfo();
    m_postdata                 = cgiEnv.getPostData();
    HttpCookies cookies        = cgiEnv.getCookieList();
    char* _httpHost = FCGI_Processor::Fgetenv("HTTP_HOST");   
    if (_httpHost) 
        m_httpHeader.httpHost  = _httpHost;

    char* _httpRefer = FCGI_Processor::Fgetenv("HTTP_REFERER");   
    if (_httpRefer) 
        m_queryMap["ICE_PARAM_REFERER"]  = _httpRefer;

    char* _httpRealAddr = FCGI_Processor::Fgetenv("HTTP_X_FORWARDED_FOR");   
    if (_httpRealAddr) 
        m_httpHeader.remoteAddr  = _httpRealAddr;

    cgicc::const_form_iterator iterE;
    for(iterE = cgi.getElements().begin(); iterE != cgi.getElements().end(); ++iterE) 
    {
        m_queryMap.insert(store_type::value_type(iterE->getName(), iterE->getValue()));
    }
    cgicc::const_file_iterator iterF;
    for(iterF = cgi.getFiles().begin(); iterF != cgi.getFiles().end(); ++iterF) 
    {
        m_queryMap.insert(store_type::value_type(iterF->getName(), iterF->getData()));
    }

    HttpCookies::iterator iterV;
    for(iterV = cookies.begin(); iterV != cookies.end(); ++iterV)
    {
        if( Atoa(iterV->getName()) == "session")
        {
            if( iterV->getValue().length() == 36 )
            {
                m_httpHeader.sessionKey = iterV->getValue();
            }
        }
        if( Atoa(iterV->getName()) == "sign")
            m_httpHeader.signValue = iterV->getValue();

        if( Atoa(iterV->getName()) == "locale")
            m_httpHeader.locale = iterV->getValue();
    }
}
```

**注意，这里的cgicc::const_file_iterator，它会将用户上传的文件内容以二进制的方式提取出来**
- getName: Get the name of the form element.The name of the form element is specified in the HTML form that called the CGI application.
- getData: Get the file data.This returns the raw file data as a string

FCGI_Accepter是一个模板类，在其静态的dispatch方法里，会接收http的请求
```
template <typename TPROC>
class FCGI_Accepter 
{
public:
    static void dispatch(const bool bIsTboCgi=false)
    {
        FCGX_Request request;
        FCGX_Init();
        FCGX_InitRequest(&request, 0, 0);
        int iRet;
        while ((iRet = FCGX_Accept_r(&request)) == 0)
        {
            FCGI_stdin->stdio_stream = NULL;
            FCGI_stdin->fcgx_stream = request.in;
            FCGI_stdout->stdio_stream = NULL;
            FCGI_stdout->fcgx_stream = request.out;
            FCGI_stderr->stdio_stream = NULL;
            FCGI_stderr->fcgx_stream = request.err;
            // counter
            ++_reqs;
            FCGI_Processor::fcgi_request = request;
            // invoke function object
            TPROC TboProcObj(bIsTboCgi);
            TboProcObj();
        }
    }
    ...
}
```
每接收到一个请求，框架就会创建一个TPROC, 这个TPROC是继承自FCGI_Processor的，在FCGI_Processor的构造函数中，就完成了上述的使用cgicc解析表单数据，并填进q_map的过程。

在CgiMain.cpp中，通过g_funcs关联http请求中script和proxyName以及Processor之间的关系。这样可以达到，不同的script_name可以通过不同的processor去处理的目的。
```
typedef struct _FUNCITEM
{
    std::string strMethod;
    std::string strPrxBoName;
    BaseProcessor *pProcessor;
} FUNC_ITEM;

FUNC_ITEM g_funcs[] = 
{
    {"efence", "goocarProxy", &g_ServantProcessor},
    {"order", "goocarProxy", &g_ServantProcessor},
    {"monitor", "goocarProxy", &g_ServantProcessor},
    {"tracking", "goocarProxy", &g_ServantProcessor},
    {"location", "goocarProxy", &g_ServantProcessor},
    {"message", "goocarProxy", &g_ServantProcessor},
    {"search", "goocarProxy", &g_ServantProcessor},
    {"address", "goocarProxy", &g_ServantProcessor},
    {"lnglat", "goocarProxy", &g_ServantProcessor},
    {"rectsearch", "goocarProxy", &g_ServantProcessor},
        
    //UserGroup
    {"GetGroupInfoService", "goocarProxy", &g_ServantProcessor},
    //cppbo
    {"GetDataService", "goocarProxy", &g_ServantProcessor},

    //IoTBo
    {"IoTquery", "goocarProxy", &g_ServantProcessor},

    //configbo
    {"device", "goocarProxy", &g_ServantProcessor},

    //openapi
    {"info", "goocarProxy", &g_ServantProcessor},
    {"devinfo", "goocarProxy", &g_ServantProcessor},
    {"getchannelid", "goocarProxy", &g_ServantProcessor},
    {"appupdate", "goocarProxy", &g_ServantProcessor},
    {"modify_pwd", "goocarProxy", &g_ServantProcessor},

    //weixinshared
    {"weixin_shared", "goocarProxy", &g_ServantProcessor}
};
```
Processor(ServantProcessor)是继承自BaseProcessor的，BaseProcessor实现了两个重要的虚函数preHandle和process，preHandle中做一些基本的校验，如:session是否过期，process就是通过其成员变量(m_ProxyBoPrx)调用Proxy服务Ice接口的过程。

## 疑问
关于cgi框架，一直有一个未解的疑问，为什么编译的时候要编译FCgiIO.cpp和fcgio.cpp这两个文件呢，这两个文件中FCgiIO.cpp是cgicc这个库的，fcgio.cpp是fcgi这个库的，重新编译想必是对源码有修改，在苦查资料之后，发现作者当时可能遇到了这个问题[fcgi编译错误](https://stackoverflow.com/questions/4577453/fcgio-cpp50-error-eof-was-not-declared-in-this-scope), 查看我们自己的fcgio.cpp文件，确实发现增加了一行#include <stdio.h>。在去github上找这个项目的源码对照，发现fcgio.cpp这个文件已经增加了#include <stdio.h>这行，看这个文件的历史修改记录，原来这行是2017年新加上去的[fcgio.cpp修改](https://github.com/FastCGI-Archives/fcgi2/commit/122e55cc354dd4a78849aed8d36c61ed9edeaeb2#diff-5302ed059d26979ddd727918b70ae46e), 现在若要使用最新的libfcgi库，想必就不用重新编译这两个文件了。


# GLogin

## 浏览器登录
- 请求到loginCgi，浏览器登录对应的方法名是loginSystem(app端登录/1/auth/access_token在获取sign的时候，执行的其实也是proxy的loginSystem)
- ice调用loginproxy，对应执行的方法是LoginBO::login，其中要设置sign的几个基本标志
```
SIGN_MEMBER_T signType;
// 初始化SIGN_MEMBER_T结构，并赋值
{
    // Set signType.signVersion
    signType.signVersion = DEFAULT_SIGN_VERSION; //2
    // Set signType.traceLog
    signType.traceLog = TRACE_LOG_UNSET; //0
    if ( !param["tracelog"].empty() && param["tracelog"] == TRACE_LOG_SET )
    {
        signType.traceLog = TRACE_LOG_SET; //1
    }
    // Set signType.time
    signType.time = _time;
    // Set signType.retainByte
    signType.retainByte =  DEFAULT_RETAIN_BYTE;
}
```
- 调用Glname的方法，获取登录账户的基本信息，账户类型可以包含，账号名，设备名，虚拟账户名，imei号,车牌号这几种
```
// 尝试设备别名，用户名和虚拟账户名登陆，失败不退出
LNAME_CARBINET_INFO lnameRecord;
int ret = GlnameOp::GetRecordByLname(loginParam.loginName, lnameRecord);
// type: 1: t_user; 2: t_enterprise; 3: t_account
// 设备别名登陆
if( lnameRecord.type == 1 )
{
    checkUserIdentity(ToString(lnameRecord.uid), loginParam.domain, loginParam.pwd, signType, resValue);
}
// 用户名登陆
else if( lnameRecord.type == 3 )
{
    checkEntIdentity(ToString(lnameRecord.uid), loginParam.domain, loginParam.pwd, signType, resValue);
}
// 虚拟账户名登陆
else if( lnameRecord.type == 2 )
{
    checkAccountIdentity(ToString(lnameRecord.uid), loginParam.domain, loginParam.pwd, signType, resValue);
}

if ("USER" == loginParam.loginType)
{
if( loginParam.loginName.length() > IEMI_LENGTH_LIMIT )
{
    // 尝试IMEI号登陆
    IMEI_CARBINET_INFO imeiInfo;
    int ret = GimeiOp::GetRecordByImei(loginParam.loginName, imeiInfo);
    if( ret == ERR_OK )
    {
        checkUserIdentity(ToString(imeiInfo.uid), loginParam.domain, loginParam.pwd, signType, resValue);    
    }
}

// 全部数字，尝试设备Id登陆，失败不退出
if( isNumber(loginParam.loginName) == true )
{
    checkUserIdentity(loginParam.loginName, loginParam.domain, loginParam.pwd, signType, resValue);
}
else
// 非数字，尝试车牌号登陆，失败不退出
{
    checkPlateIdentity(loginParam.loginName, loginParam.domain ,loginParam.pwd, signType, resValue);
}
```
- 通过checkXXXIdentity方法获取账号的基本信息和sign并返回给cgi，已checkEntIdentity为例
首先校验pwd，自定义域名等信息check_pwd_old，judgeLocalUserLogin，之后通过FormatEntValue方法构造sign以及账号基本信息
```
std::string LoginBO::FormatEntValue(::pb::Customer &eInfo, SIGN_MEMBER_T &signType)
{
    std::string eid    = ToString(eInfo.mutable_base()->id());
    std::string grade  = eInfo.mutable_extend()->grade();

    // Set signType.userType
    signType.userType = ENT_TYPE_FLAT;
    // Set signType.userID
    signType.userID = eid;
    // 用户类型 + 用户账号 + 时间（10字节） + 32位md5 
    // Set signType.sign
    std::string rawData = signType.userType + eid + signType.time + g_conf.signKey;
    if ( MakeMD5(rawData, signType.sign) != ERR_OK )
    {
        MYLOG_WARN(g_logger, "FormatUserValue rawData MD5 Failed[%s]", rawData.c_str());
    }
    // Set signType.userGrade
    signType.userGrade = grade;
    std::string signValue;
    SIGN_MEMBER_T2String(signType, signValue);

    std::string strBuf;
    strBuf.append(eid); 
    strBuf.append("#"); 
    strBuf.append(ToString(eInfo.mutable_base()->pid())); 
    strBuf.append("#"); 
    strBuf.append(eInfo.mutable_base()->name()); 
    strBuf.append("#"); 
    strBuf.append(eInfo.mutable_base()->phone()); 
    strBuf.append("#"); 
    strBuf.append(eInfo.mutable_base()->login_name()); 
    strBuf.append("#"); 
    strBuf.append(eInfo.mutable_base()->email()); 
    strBuf.append("#"); 
    strBuf.append(eInfo.mutable_base()->addr()); 
    strBuf.append("#"); 
    strBuf.append("***"); 
    strBuf.append("#"); 
    // status字段没什么用，直接改成grade
    strBuf.append(eInfo.mutable_extend()->grade());
    strBuf.append("#");  
    strBuf.append(eInfo.mutable_base()->create_time() + ".000");
    strBuf.append("#"); 

    Goome::ReplaceStr(strBuf, "#NULL#", "##");
    Goome::ReplaceStr(strBuf, "#null#", "##");
    return strBuf + "#" + signValue;
}
//以TLV方式保存sign信息，长度固定占两个字节，如长度为7则表示为07
int SIGN_MEMBER_T2String(const SIGN_MEMBER_T &signValue, std::string &tag)
{
    char sz[128] = ""; 
    snprintf(sz,sizeof(sz), "%s%s%s%s%s%s%s%s%s%s%s%s%s%s%s%s%s%s",
             signValue.signVersion.c_str(), signValue.userType.c_str(), 
             getLengthString(signValue.userID).c_str(), signValue.userID.c_str(), 
             getLengthString(signValue.time).c_str(), signValue.time.c_str(),
             getLengthString(signValue.sign).c_str(), signValue.sign.c_str(), 
             getLengthString(signValue.accountID).c_str(), signValue.accountID.c_str(), 
             getLengthString(signValue.virtrualAccount).c_str(), signValue.virtrualAccount.c_str(), 
             getLengthString(signValue.traceLog).c_str(), signValue.traceLog.c_str(),
             getLengthString(signValue.userGrade).c_str(), signValue.userGrade.c_str(), 
             getLengthString(signValue.retainByte).c_str(), signValue.retainByte.c_str());
    tag = std::string(sz);

    return ERR_OK;
}
```
- loginCgi收到loginProxy发回的请求，解析出账号的基本信息如eid，login_name等，根据账号的grade以及eid，设置homepage。之后设置返回给浏览器的cookie。

# MysqlSync






