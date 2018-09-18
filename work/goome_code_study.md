# CGI Framework
## CGI Framework中使用到的库
- [FCGI](http://www.nongnu.org/fastcgi/)
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
    cgicc::const_file_iterator  iterF; 
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

fcgi在初始化的时候（FCGI_Processor构造函数），会通过cgicc提取http请求的各种数据放在
m_httpHeader, m_queryMap, m_postdata中供后续FCGI_Processor的子类使用



# GLogin

# MysqlSync






