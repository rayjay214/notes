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
Processor(ServantProcessor)是继承自BaseProcessor的，BaseProcessor实现了两个重要的虚函数preHandle和process，preHandle中做一些基本的校验，如:session是否过期，preHandle中还会根据cookie中的sign值或者access_token值解析出login_type和login_id放到qmap中去。
process就是通过其成员变量(m_ProxyBoPrx)调用Proxy服务Ice接口的过程。

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
```
std::string loginByEnterprise(store_type &param, http_header_t &m_httpHeader, 
                              session_content_t &sessionVO, 
                              const std::string _locateStr, bool axaxLoginFlat)
{
    std::string _retStr;
    std::string _cookieStr;
    int ret = commitPrxBo(param, _retStr);
    
    enterprise_t eVO;

    param["cookie_sign"] = eVO.sign;
    _cookieStr = setCookie("session", m_httpHeader.sessionKey, m_httpHeader); 
    _cookieStr += setCookie("user", "\""+eVO.id + "," + UrlEncode(UrlEncode(eVO.name))+"\"", m_httpHeader); 
    _cookieStr += setCookie("sign", eVO.sign, m_httpHeader);
    _cookieStr += setCookie("group", g_gns_shard_id, m_httpHeader);
    _cookieStr += setCookie("psip", m_httpHeader.httpHost, param["accessUrl"], "/");

    sessionVO.eUserID    = eVO.id;
    sessionVO.eUserIDing = eVO.id;
    sessionVO.loginURL   = param["url"];
    sessionVO.ecarServiceURL = ECAR_SERVICE_MWO;
    sessionVO.sessionKey = m_httpHeader.sessionKey;

    std::string lang;
    lang = param["locale"].empty() ? m_httpHeader.locale : param["locale"] ;
    if( lang.empty() )
    {
        lang = "zh-cn";
        std::size_t found = m_httpHeader.httpHost.find("cootrack");
        if( found!=std::string::npos )
            lang = "en-us";
    }
    sessionVO.locale = ( lang == "en-us") ? "en" : "cn";

    // 客户类型(1.特殊类型，4.经销商 8.终端客户 6.租赁 7.物流 5.汽修厂)
    sessionVO.grade ="8";
    if( !eVO.grade.empty() )
    {
         sessionVO.grade = eVO.grade;
    }
    MYLOG_INFO(logger, "loginByEnterprise Session Grade is [%s]", sessionVO.grade.c_str()); 

    // addr字段存放的是登陆次数
    if( eVO.addr.empty() ==  false )
         sessionVO.permission = eVO.addr;

    // 看来自的请求，如果来自警察专用的区域回放系统，那么跳转到 poliz文件夹
    std::size_t found = param["url"].find(JC_AREA_PLAY_BACK_DOMAIN);
    if( found!=std::string::npos )
    {
        sessionVO.eVODealer = "OK";
        sessionVO.homePage = REDIRECTOR_PATH_POLIZ;
        setSession(param, m_httpHeader.sessionKey, sessionVO);
        MYLOG_INFO(logger, "loginByEnterprise [%s] Login Redirect To /poliz", eVO.id.c_str()); 
        return _cookieStr + retRedirect(REDIRECTOR_PATH_POLIZ, _locateStr);
    }

    std::string homePage = REDIRECTOR_PATH_USER;
    // 客户级别：6:租赁 7:物流 8:终端客户
    if( sessionVO.grade == "6" || sessionVO.grade == "7" || sessionVO.grade == "8")
    {
        sessionVO.eID = eVO.id;
        homePage = REDIRECTOR_PATH_USER;        
        MYLOG_INFO(logger, "loginByEnterprise [%s] Login Redirect To /user", eVO.id.c_str()); 
    }
    ...
```

- 生成sessionid，并设置session到redis中去
```
//sessionKey是登录预处理时设置的，每次调这个方法获取的值都是唯一的
sessionKey = IceUtil::generateUUID();

setSession(param, m_httpHeader.sessionKey, sessionVO);

int setSession(store_type _setSessionParam, const std::string &sessionKey, session_content_t &sessionValues)
{
    sessionValues.permission = transChars(sessionValues.permission, std::string(";"), std::string("；"));

    std::string _sessionStr;
    _sessionStr  = sessionValues.loginURL       + ";";
    _sessionStr += sessionValues.ecarServiceURL + ";";
    _sessionStr += sessionValues.homePage       + ";";
    _sessionStr += sessionValues.grade          + ";";
    _sessionStr += sessionValues.virtualName    + ";";
    _sessionStr += sessionValues.userNameORG    + ";";
    _sessionStr += sessionValues.permission     + ";";
    _sessionStr += sessionValues.eUserID        + ";";
    _sessionStr += sessionValues.eUserIDing     + ";";
    _sessionStr += sessionValues.eID            + ";";
    _sessionStr += sessionValues.viewType       + ";";
    _sessionStr += sessionValues.QQConnectState + ";";
    _sessionStr += sessionValues.openID         + ";";
    _sessionStr += sessionValues.token          + ";";
    _sessionStr += sessionValues.QQUserInfo     + ";";
    _sessionStr += sessionValues.userVO         + ";";
    _sessionStr += sessionValues.eVODealer      + ";";
    _sessionStr += sessionValues.locale         + ";";
 
    std::string _retStr;
    _setSessionParam["ICE_PARAM_SERVLET"] = "LoginService";
    _setSessionParam["method"] = "setSession";
    _setSessionParam["sessionID"] = sessionKey;
    _setSessionParam["value"] = _sessionStr;

    CSessionBO sBO;
    _retStr = sBO.setSession(_setSessionParam);
    ...
 }
 
int CSessionBO::putSessionRedis(const std::string& sid, const std::string& value)
{
    std::string key = RedisSessionKeyPrefix + sid;
    if(!m_redis->SetEx(key, value, g_redis_expire_sec))
    {		
        return ERR_CALL_REDIS_FAIL;	
    }
    return ERR_OK;
}
```

- session主要是给前端使用的，通过get_meta_js方法，就能获取很多session中保存的有用信息
```
nRet = getSessionRedis(sid, value);	
if (nRet != ERR_OK)
{
    return nRet;
}
value = b64_decode_string(value);

nRet = getSessionValue(value, stSession);

//User
GetUserVo(stSession);

//Ent Delaer
GetEnterpriseDealerVO(stSession);

//Ent
GetEnterpriseVO(stSession);

GetEnterpriseGpsDayInfo(eid, stSession);
```

- 返回给浏览器是重定向后的地址，并带上刚才设置的cookie
```
...
return _cookieStr + retRedirect(homePage, _locateStr, axaxLoginFlat);
std::string retRedirect(const std::string path, const std::string host, bool axaxLoginFlat)
{
    std::string retRedir = "Location:" + host + path + "\r\n\r\n";

    if( axaxLoginFlat == true )
    {
        retRedir = "\r\n" + host + path;
    }
    return retRedir;
}
```

# Tracelog

## 杂记
浏览器通过tracecgi发送配置信息到logproxy，包含的信息有要监控服务的ip:port，以及浏览器生成的session_id。logproxy收到配置信息后，调用CConfigInfo::Set(const string& ip, int port, const ::Log::cfg2& c, SessionType stype)，将配置信息保存在缓存中，此处如果配置信息存在（ip,port,session_id,p1,p2,p3都一样），则更新配置，否则新增配置信息。缓存更新完成


# 设备协议相关

## 离线指令发送流程
- 客户端发送请求（method=set&order_id=xxx&content=xxx）给平台。
  OrderHandle.cpp中根据order_id在t_user_menu中找到指令的详细内容（url和remark字段），以及用户发送的content，构造指令完整内容。先将指令具体详情插入t_cmd_history表，得到sn（自动生成的id），再构造消息发送到downlink。
```
std::string OrderContent;
std::string OrderName;
if ( buildOrderContent(orderId, orderParam, OrderContent, OrderName) )
{
    return ERR_INVALIDATE_PARAM;
}

std::string login_id = ("0"==param["LOGIN_TYPE"])?param["LOGIN_ID"]:"1";
long orderGenId;

if ( insertOrderRecord(user_id, login_id, OrderContent, OrderName, orderId, platform, orderGenId) )
{
    return ERR_CALL_GUD_FAIL;
}
std::string downLinkOrderMsg = pack2DownlinkOrderMsg(orderId, ToString(orderGenId), imei, OrderContent);
g_pSyncToDownlinkThread->write(downLinkOrderMsg);
```

- downlink接收到指令并尝试发送给设备，若无响应（设备不在线），则将指令发给ggw缓存
handlerThread接收到ServantBo发来的消息后, 先将之前保存的指令改为已执行，之后将其放入离线指令处理线程发送队列（SetSendMap），然后将其放进自己的m_onlineMsgVec，并尝试发送消息。
```
msg_content_t msg_cnt;
if (ParseMsg(m_msg, msg_cnt))
{					
    SetGgwEnd(strImei, EMPTY_GGW_IP);

    std::string strKey = "gpsbox:OffLineInstr:" + strImei;
    std::string strInst = GetInstrFromRedis(strKey);
    //如果是指令，则先将数据库中已保存的离线指令置为已执行，保证新指令一定能覆盖旧指令
    if (!strInst.empty() && msg_cnt.msgType == MSG_TYPE_INSTRCTION)
    {
        m_updateThread->UpdateInstrStateInDB(msg_cnt);
    }
    if (msg_cnt.msgType == MSG_TYPE_INSTRCTION)
    {
        instrInfo stInstrInfo = {m_msg, (uint64_t)time(0), "CPPBO"};
        SetSendMap(devId, stInstrInfo);
    }
    std::string strType = GetDevType(strImei, devId);
    StringSeq::iterator fIter = find(g_conf.commonInstrDevType.begin(), g_conf.commonInstrDevType.end(), strType);
    if (fIter != g_conf.commonInstrDevType.end())
    {						
        m_commonInstrVec.push_back(m_msg);						
    }
    else
    {						
        IceUtil::Monitor<IceUtil::RecMutex>::Lock lock(m_mutex_online);
        m_onlineMsgVec.push_back(m_msg);
        m_mutex_online.notify();
    }
}  
if (time(0) - m_tLastSendCommonInstrTime > 3 && !m_commonInstrVec.empty())
{
    SendCommonInstrMsg();
    m_tLastSendCommonInstrTime = time(0);
}
if (time(0) - lastTimeSendOnline > 3)
{
    if (!m_onlineMsgVec.empty())
    {
        SendOnlineMsg();					
        lastTimeSendOnline = time(0);					
    }				
}
```

- OffLineInstrHandler线程从finishSend队列中取出刚才设置的指令（到了检查周期之后），将其放入待检查的vector，之后在待检查vector中通过t_cmd_history获取该指令的status，因为指令还未执行，这时status应该为0。若status=0，则将这条指令上报给所有的ggw去缓存，并且放入ready2SaveDB队列，之后replace into t_off_line_instr表。
```
bool COffLineHandlerThread::ProcessSentInstruction(const tbb::concurrent_unordered_map<uint32_t, instrInfo>& finishSendMap)
{
	tbb::concurrent_unordered_map<uint32_t, instrInfo> finishOfflineSendMap;
	std::vector<std::string> strCheckIdVec;
	
	//判断已下发的指令是否支持离线指令，若支持离线指令，在规定时间内是否收到回复，若未收到回复，则通知GGW在下次设备上线时通知downlink下发该指令
	for (tbb::concurrent_unordered_map<uint32_t, instrInfo> ::const_iterator itr = finishSendMap.begin(); itr!=finishSendMap.end(); itr++)
	{
		std::string strType;
		std::string strSn;
		//若获取不到相应的type和sn，则认为指令非法，不做离线指令的判断，对非指令不做离线指令判断
		if (!GetJsonStringItem(itr->second.strContent, "type", strType) ||strType != MSG_TYPE_INSTRCTION || !GetJsonStringItem(itr->second.strContent, "sn", strSn)||strSn == "")
		{
			continue;
		}
	
		std::string strDevType;
		std::string strImei;
		GetJsonStringItem(itr->second.strContent, "imei", strImei);
		strDevType = GetDevType(strImei);
	
		//对GM08等udp设备，指令下发隔3s检查是否收到结果(在配置文件中配置)，其他的tcp连接设备隔15s检查
		uint64_t checkInterval = g_conf.checkOfflineInterval;
		std::string iotType;
		bool iotForOfflineInstr = false;
		if (GetJsonStringItem(itr->second.strContent, "iottype", iotType) &&
			(iotType == "IOT"))
		{
			iotForOfflineInstr = true;
		}
		if (std::find(g_conf.udpOfflineInstr.begin(), g_conf.udpOfflineInstr.end(), strDevType) != g_conf.udpOfflineInstr.end())
		{
			checkInterval = g_conf.checkUdpOfflineInterval;
		}
	
		time_t now = time(0);								
		if(now - itr->second.recvCmdTime > checkInterval)
		{
			//仅对支持离线指令的设备类型判断指令下发是否成功，其他设备的指令一律视为在线已下发成功

			//改为在线指令也要保存后，这条就可以不用判断了
			if (!IsOfflineInstructionDevType(strDevType) && (iotForOfflineInstr == false))
			{			
				continue;
			}
			
			strCheckIdVec.push_back(strSn);
			finishOfflineSendMap.insert(*itr);
		}
		else
		{
			//未到规定检查间隔的，则继续等待
			m_handlerThread->InsertIntoSendMap(itr);
		}
	}
	
	std::map<uint32_t, std::string> insStatusMap;
	if (!strCheckIdVec.empty())
	{
		//查找数据库，获取指令执行结果
		GetInstrStatusFromDB(strCheckIdVec,insStatusMap,m_gudPrx);
	}
	
	std::map<std::string, msg_content_t> ready2SaveDB;
	std::vector<msg_content_t> ready2UpdateDB;
	
	for(tbb::concurrent_unordered_map<uint32_t, instrInfo>::iterator itr = finishOfflineSendMap.begin(); itr != finishOfflineSendMap.end(); itr++)
	{				
		std::string strImei;
		GetJsonStringItem(itr->second.strContent, "imei", strImei);
		std::string strKey = "gpsbox:OffLineInstr:" + strImei;				
		std::string strInstr = GetInstrFromRedis(strKey);
		std::string strDevType = GetDevType(strImei);
	
		std::map<uint32_t, std::string>::iterator find_itr = insStatusMap.find(itr->first);
	
		//在t_cmd_history里找得到指令下发记录的离线指令才处理，否则一律认为指令下发已过期，不再处理(t_cmd_history表可能会定期清理)
		MYLOG_DEBUG(logger,"ready to get cmd result for devId[%u]", itr->first);
		if (find_itr != insStatusMap.end())
		{
			if (find_itr->second != "0")
			{						
				if (strInstr!="**nonexistent-key**")
				{				
					msg_content_t tmsg;
					ParseMsg(strInstr,tmsg);
					ready2UpdateDB.push_back(tmsg);
					m_handlerThread->SetGgwEnd(strImei, EMPTY_GGW_IP);
				}
				else
				{
					MYLOG_DEBUG(logger,"invalid redis value found for key[%s], value[%s]", strKey.c_str(), strInstr.c_str());
				}
			}
			else
			{

				//初始化的数据只用于通知ggw，刷新ggw缓存中的指令内容，不作重发处理
				//只有支持离线指令的设备需要做重发，其他设备每次上报数据的时候发一次就足够了
				if (itr->second.strCmdSrc.compare("INIT") != 0 && IsOfflineInstructionDevType(strDevType))
				{
					//未收到回复的离线指令加入重发队列，连续发三次
					resendInfo rsi;
					IceUtil::Monitor<IceUtil::RecMutex>::Lock lock(m_monitor);
					m_resendMap[strImei] = rsi;
					m_monitor.notify();
				}
										
				//通知GGW进入监控状态，检测保存离线指令的设备是否再次上线
				if (!SendUnsucOfflineInfoToGGW(strImei))
				{
					MYLOG_DEBUG(logger,"failed in sending request to GGW for imei[%s]", strImei.c_str());
				}
				MYLOG_DEBUG(logger,"finish sending unsuccessful offline info to ggw, cmd src is:%s", itr->second.strCmdSrc.c_str());
				if (itr->second.strCmdSrc.compare("CPPBO") == 0)
				{
					msg_content_t tmsg;
					ParseMsg(itr->second.strContent,tmsg);
					ready2SaveDB[itr->second.strContent] = tmsg;
					// 新来的离线数据，存完数据库，往GGW发一份，用来更新GGW中的缓存
					MYLOG_DEBUG(logger,"ready to send new offline to ggw");
					SendNewCmd2GGW(tmsg);
				}
				else if (itr->second.strCmdSrc.compare("INIT") == 0)
				{
					//若配置文件中配置了需要初始化ggw中的离线指令缓存的话，就给ggw发送一份数据
					msg_content_t tmsg;
					ParseMsg(itr->second.strContent,tmsg);
					MYLOG_DEBUG(logger,"ready to init new offline to ggw");
					SendNewCmd2GGW(tmsg);
				}
			}
		}
		else
		{
			MYLOG_DEBUG(logger,"cannot find instruction for devid[%u]", itr->first);
		}
	}
	
	m_updateThread->SaveInDB(ready2SaveDB);
	// 执行完的离线指令，也要给GGW发一份，用来清除GGW中的缓存
	for (uint64_t i = 0; i < ready2UpdateDB.size(); i++)
	{
		m_updateThread->UpdateInstrStateInDB(ready2UpdateDB[i]);				
		MYLOG_DEBUG(logger,"ready to send finished offline to ggw");
		SendFinishedCmd2GGW(ready2UpdateDB[i]);
	}		

	return true;
}
```

（最古老的方式，没用ggw缓存指令）
- 在设备上线了之后，比如ggw收到了0x80包，除了给设备回响应之外，如果在缓存中发现了离线指令，还要给gpp发一个0x1111包。
```
if (hasOffline && reply)
{
    SendGm08ReplyMsg();
    Send1111Package2Gpp(id, PROTNO_SEND_OFFLINE_INSTRUCTION);
}
```

- Gpp收到0x1111消息之后，检查是否有离线指令，有的话发送json消息给downlink。
```
else if (INNER_1111_HEAD == *(uint16_t*)&str[0])
{
    if (*(uint8_t*)&str[3] == 0xA1 || *(uint8_t*)&str[3] == 0xA2)
    {
        return CheckAndSendOfflineInstruction(str);
    }
}

bool CConsumerThread::CheckAndSendOfflineInstruction(const std::string& str)
{
    const INNER_LOGIN_MSG* pDestMsg = (const INNER_LOGIN_MSG*) str.c_str();

	uint64_t imei = 0;
	uint64_t devId = 0;
	imei = ntoh64(pDestMsg->imei);
	if(!GetDevId(imei, &devId))
	{
		return false;
	}

    if((uint64_t)g_pTracer->m_tracer.p2 == devId)
    {
        g_pTracer->DLog("uid==%lld, imei=%s, CheckAndSendOfflineInstruction 1111:", g_pTracer->m_tracer.p2, g_pTracer->m_tracer.p3.c_str());
        DumpHex((uint8_t*)&str[0], str.size(), g_pTracer);    
    }

	std::string strImei;	
    std::string strOfflineInstr;
    MYCOM::ImeiDecToHex(imei, strImei);

	MYLOG_DEBUG(logger,"ready to get offline instruction for strImei[%s]", strImei.c_str());

	bool hasOfflineInstr = GetOffLineInstrFromRedis(strImei, strOfflineInstr);
	if (hasOfflineInstr)
	{
		MYLOG_INFO(logger,"offline instruction found for imei[%s], instruction[%s], send to downlink", strImei.c_str(), strOfflineInstr.c_str());
		std::string strGgwEnd = GetGgwIp();		

		rapidjson::StringBuffer strbuf;
		rapidjson::Writer<rapidjson::StringBuffer> writer(strbuf);
		rapidjson::Document root;
		rapidjson::Document::AllocatorType& allocator = root.GetAllocator();
		root.SetObject();
		rapidjson::Value val;
		OBJ_ADD_STR_MEMBER(root, "OfflineInstrImei", strImei);
		OBJ_ADD_STR_MEMBER(root, "GgwEndpoint", strGgwEnd);
		root.Accept(writer);
		std::string strSendDown = strbuf.GetString();

		SendMsgToDownlinkByGMQ(devId,strSendDown.c_str(),strSendDown.size());
	}
	else
	{
		MYLOG_DEBUG(logger,"found no offline instrcution for strImei[%s]", strImei.c_str());
	}

	if (*(uint8_t*)&str[3] == 0xA2)
	{
		return SendOfflineRplToGgw(hasOfflineInstr, imei);
	}

	return true;
}
```

- downlink收到gpp的消息后，再次将其放入sendmap，以及m_onlineMsgVec，之后发送具体的指令(0x5858包)给ggw。
```
if (isOfflineInstruction(m_msg))
{
    //触发离线指令下发（在离线指令全切换到保存在GGW中后，只有在线指令的下发会触发这个逻辑）
    GetJsonStringItem(m_msg,"OfflineInstrImei",strImei);
    uint32_t devId = 0;
    if (!GetDevId(strImei, &devId, m_gudPrx, m_pRedisClient))
    {
        MYLOG_WARN(logger, "cannot find device id for imei[%s]", strImei.c_str());
        continue;
    }

    std::string strGgwEnd;
    if (GetJsonStringItem(m_msg,"GgwEndpoint",strGgwEnd))
    {
        SetGgwEnd(strImei, strGgwEnd);
    }
    else
    {					
        SetGgwEnd(strImei, EMPTY_GGW_IP);
    }

    std::string strKey = "gpsbox:OffLineInstr:" + strImei;
    std::string strInst = GetInstrFromRedis(strKey);
    if (strInst.empty())
    {
        MYLOG_DEBUG(logger,"empty off line instruction for imei[%s] in redis", strImei.c_str());
        continue;
    }

    std::string strSn;
    if (GetJsonStringItem(strInst,"sn",strSn))
    {
        MYLOG_DEBUG(logger,"instruction get for imei[%s], devId[%d], sn is [%s]", strImei.c_str(), devId, strSn.c_str());

        instrInfo stInstrInfo = {strInst, (uint64_t)time(0), "OFFLINE"};

        MYLOG_INFO(logger, "rayjay set send map for %ld", devId);
        SetSendMap(devId, stInstrInfo);

        std::string strType = GetDevType(strImei, devId);
        StringSeq::iterator fIter = find(g_conf.commonInstrDevType.begin(), g_conf.commonInstrDevType.end(), strType);
        if (fIter != g_conf.commonInstrDevType.end())
        {
            m_commonInstrVec.push_back(strInst);
        }
        else
        {
            IceUtil::Monitor<IceUtil::RecMutex>::Lock lock(m_mutex_online);
            m_onlineMsgVec.push_back(strInst);
            m_mutex_online.notify();
        }
    }
}
```

- ggw收到0x5858包，将其发送给设备。


- 设备收到了指令之后，正常情况下，会发送0x85包给ggw，ggw将其转发给gpp


- gpp收到了0x85设备响应包之后，将消息发送给warning_processor，由其改变t_cmd_history表的status以及response等字段。
```
//定时回传和下发调度文本信息downlink回复
if (uProtId == 0x85)
{
    char mqbuf[1024] = {0};
    std::string strContent = "Responce Succeed!";
    snprintf(mqbuf, 1023,"0001%%%ld##%d#%s",uDevId, 0, strContent.c_str());

    MYLOG_INFO(vul_logger,"downlink response: %s", mqbuf);
    std::string mq_string(mqbuf);
    ::GMQ::StrSeq tbl;
    tbl.push_back(mqbuf);
    m_pstConsumerThread->SendMsg2GmqStr(gConfData.vcAlarmCallee[0],tbl);
}
```

- downlink的offline线程会检测待发送指令的状态，即t_cmd_history的status字段，如果发现它不等于0之后，将t_off_line_instr表中的status改成1，表示此离线指令已发送成功。

### 问题
既然所有ggw上都已经缓存了所有的离线指令，那么ggw发现设备上线了之后，直接发送指令给设备就可以了，不应该绕ggw-->gpp-->downlink-->ggw这么一个大圈，其实这个功能也是已经实现了的，但是通过灰度控制了，还没有启用。在目前的灰度机制下，只有百分之一的设备能采用新的机制。
```
bool GWReader::SendOfflineInstruction(uint64_t id, int fd, bool isUdp)
{
	if (m_cacheBlock == NULL)
	{
		MYLOG_ERROR(m_pLogger,"cache block has not been inited");
		return false;
	}


	uint64_t uiHexImei = HexToDec(ntoh64(id));
	int gs_dev = greyscale(uiHexImei, GGW_SERVICE_ID_OFFLINE_CMD);
	if (gs_dev <= 0)
	{
		MYLOG_DEBUG(m_pLogger,"imei[%lu] is not in the grey scale", uiHexImei);
		return true;
	}
	
	MYLOG_DEBUG(m_pLogger,"ready to get off line instruction for imei[%lu], fd[%d]", uiHexImei, fd);

	std::string strCmd;
	uint32_t sn = 0;
	if (m_cacheBlock->GetCmd(uiHexImei, sn, strCmd))
	{
		if(uiHexImei == g_pTracer->m_tracer.p2)
		{
			g_pTracer->DLog("get off line instrution in cacheblock for imei[%lu], sn[%u], fd[%d]", uiHexImei, sn, fd);	 
		}
	
		MYLOG_DEBUG(m_pLogger,"get off line instrution in cacheblock for imei[%lu], fd[%d]", uiHexImei, fd);		
        hexdump2(0,strCmd,__FILE__, __LINE__);
		if (isUdp)
		{			
			sendto(g_udpfd, &strCmd[0], strCmd.size(), 0, (struct sockaddr *)&m_stClient, sizeof(struct sockaddr_in));
		}
		else
		{
            if (writen(fd, (const void *)&strCmd[0], strCmd.size()) != strCmd.size())
            {
				if(uiHexImei == g_pTracer->m_tracer.p2)
				{
					g_pTracer->DLog("SendOfflineInstruction failed: WRITE_CTRL_ERROR cfd = %d", fd);	 
				}
			
                MYLOG_WARN(m_pLogger, "SendOfflineInstruction failed: WRITE_CTRL_ERROR cfd = %d", fd);
                return false;
            }		
		}
	}
	else
	{
		if(uiHexImei == g_pTracer->m_tracer.p2)
		{
			g_pTracer->DLog("cannot find off line instruction in cache block for imei[%lu], fd[%d]", uiHexImei, fd);	 
		}
	
		MYLOG_DEBUG(m_pLogger,"cannot find off line instruction in cache block for imei[%lu], fd[%d]", uiHexImei, fd);
		return false;
	}

	if(uiHexImei == g_pTracer->m_tracer.p2)
	{
		g_pTracer->DLog("SendOfflineInstruction successfully: imei[%lu]", uiHexImei);	 
	}

	NotifyDonwlink(id);
	return true;
}
```

# 运维
## gmq跨机房推送
公共组件内部模块要对另一个公共组件内部模块推送数据时, 比如支付BO(PAY_BO), 通过两个gmq转发消息, 两个gmq分别配置为to_proxy(布署在发送方的接口机上), 和as_proxy(布署在接收方的接口机上).
发送方在配置文件中配置to_proxy对应的gmq的IP与端口.在caller所属gns中配置caller为调用方, 比如PAY_BO, callee为gmq,比如GOOCAR_GMQ_PROXY.  
callee所属gns中配置caller为调用方, 比如PAY_BO, callee为被调方,比如CAROL_IOT_CARDPOOL_BO. (注意, GOOCAR_GMQ_PROXY所在机器, 要添加caller,否则调用关系无法同步到这台机器上,这时会对caller作为被调时产生干扰, 如果caller要作为被调, 必须使用不同的GNS名)

### 例子，dc_114的user_servant_bo推送消息到dc7_132的carol_log_record服务
dj_114上user_servant_bo的配置如下
```
LOGRECORD_GMQ_ENDPOINT=192.168.2.131:22334 (gmq_to_proxy)
LOGRECORD_CALLER=COMMON_LOGRECORD_PROXY
LOGRECORD_CALLEE=CAROL_LOG_RECORD
```
servant_bo将数据写入dj接口机的gmq_to_proxy中去。写入此gmq中的消息的caller为COMMON_LOGRECORD_PROXY，callee为CAROL_LOG_RECORD
dj相关gns配置如下。
```
COMMON_LOGRECORD_PROXY-->CAROL_LOG_RECORD
COMMON_LOGRECORD_PROXY-->COMMON_GMQ_DJ2DC_1
```
gmq_to_proxy接收到的消息中，虽然callee配置为CAROL_LOG_RECORD，但是由于配置项gmq.proxyname=COMMON_GMQ_DJ2DC_1，gmq会将消息推送到COMMON_GMQ_DJ2DC_1(dc_131,gmq as proxy)。COMMON_GMQ_DJ2DC_1配置为183.60.142.149:22335, 由于是dj发往dc，所以这里要配置成公网ip，通过查看在物理机器上执行sudo iptables -L -nv -t nat |grep 22335可以看到
`1003K   60M DNAT       tcp  --  em1    *       113.105.139.0/24     0.0.0.0/0            tcp dpt:22335 to:10.0.1.131:22335`
这表明113.105.139.0子网发往dc_199:22335的消息都被转发到了dc_131:22335，这里正是部署dc机房gmq_as_proxy的位置。
dc的gns配置如下。
```
COMMON_LOGRECORD_PROXY(192.168.1.131:0)
COMMON_LOGRECORD_PROXY-->CAROL_LOG_RECORD
```
gmq_as_proxy接收到消息之后，消息的caller，callee仍然是之前的COMMON_LOGRECORD_PROXY，CAROL_LOG_RECORD，此时gmq_as_proxy就可以把消息推送到CAROL_LOG_RECORD了。

#### hint
gns的t_service_node中要配置COMMON_LOGRECORD_PROXY(192.168.1.131:0)，是因为要在131机器的GNS共享内存中生成COMMON_LOGRECORD_PROXY-->CAROL_LOG_RECORD这条记录，不然GMQ不知道要往哪里推送。

#### 疑问
gmq_as_proxy配置了proxy.name=COMMON_GMQ_PROXY, 并且GNS还配置了COMMON_GMQ_PROXY-->CAROL_LOG_RECORD, 不知道有什么作用。


# 设备接入层

## 基础

### 二进制（16进制）数据流和字符串之间的转换
我们定位问题时常说的原始数据就是二进制数据的16进制表示，但其实在代码中，数据是存储在string中的。string是由很多char组成的，其中每一个char的值，就是16进制的值。比如原始数据是2A 13，那实际存储的情况就是str[0]=42, str[1]=19。这种情况下，想通过string来查看原始数据，是不能通过cout打印string的值来实现的，因为cout会将string以ascii码的形式展示出来，如刚才的str打印出来结果就是*(设备控制3)。要想打印原始数据，需通过hexdump函数。
```
void hexdump(uint8_t* buf, int len)
{
    char myBuf[1024] = {0};

    if (len > 341)  // 341=1024/3: To avoid overflow
        len = 341;
    
    for (int i = 0; i < len; i++)
        sprintf(myBuf+3*i, "%02x ", buf[i]);

    MYLOG_INFO(logger, myBuf);
}

std::string strHex;
hexdump((uint8_t*)&strHex, strHex.size());
```
在测试程序中，有很多这样的场景，已知原始数据，如果将其保存在string中，可以使用如下的程序。
```
int StringHexToDec(const std::string& strHex)
{
    int result = 0;
	for (int i = 0; i < (int)(strHex.size()); i++)
	{
		int index = strHex.size()-1-i;
		if (strHex[index] >= '0' && strHex[index] <= '9')
		{
			result += (int)(strHex[index] - '0')*pow(16,i);
		}
		else if (strHex[index] >= 'A' && strHex[index] <= 'F')
		{
			result += (int)(strHex[index] - 'A' + 10)*pow(16,i);
		}
		else if (strHex[index] >= 'a' && strHex[index] <= 'f')
		{
			result += (int)(strHex[index] - 'a' + 10)*pow(16,i);
		}
	}

    return result;
}

std::string LongStringHexToStringDec(const std::string& strHex)
{
    string strResult;
    string strFirst;
    char c;
    for(string::size_type i = 0; i < strHex.size(); i+=2)
    {   
        strFirst = strHex.substr(i, 2); 
        c = StringHexToDec(strFirst);
        strResult.push_back(c);
    }   
    return strResult;
}
```



# GFS

## GFS简介



# MysqlSync






