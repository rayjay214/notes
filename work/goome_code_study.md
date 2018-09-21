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


# GLogin

# MysqlSync






