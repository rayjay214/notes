## Slice2cpp
### ice客户端动态使用ice服务端的proto结构
代码看起来有如下不同  
注意，以下的修改点都是针对于服务端来说的，对ice的客户端没用  
官方教程中ice客户端还需要引入自动生成的头文件，但若使用pb字节流传传参数的方式，就不用引入该头文件了
1. 官方slice2cpp生成的cpp文件
```
::Ice::Int
IceDelegateM::Servant::Service::Call(const ::Servant::ParamMap& params, ::std::string& result, const ::Ice::Context* __context)
{
    ::IceInternal::Outgoing __og(__handler.get(), __Servant__Service__Call_name, ::Ice::Normal, __context);
    try
    {
        ::IceInternal::BasicStream* __os = __og.os();
        ::Servant::__writeParamMap(__os, params);
    }
    catch(const ::Ice::LocalException& __ex)
    {
        __og.abort(__ex);
    }
    bool __ok = __og.invoke();
    ::Ice::Int __ret;
    try
    {
        if(!__ok)
        {
            try
            {
                __og.throwUserException();
            }
            catch(const ::Ice::UserException& __ex)
            {
                ::Ice::UnknownUserException __uue(__FILE__, __LINE__, __ex.ice_name());
                throw __uue;
            }
        }
        ::IceInternal::BasicStream* __is = __og.is();
        __is->startReadEncaps();
        __is->read(result);
        __is->read(__ret);
        __is->endReadEncaps();
        return __ret;
    }
    catch(const ::Ice::LocalException& __ex)
    {
        throw ::IceInternal::LocalExceptionWrapper(__ex, false);
    }
}

::Ice::DispatchStatus
Servant::Service::___Call(::IceInternal::Incoming& __inS, const ::Ice::Current& __current)
{
    __checkMode(::Ice::Normal, __current.mode);
    ::IceInternal::BasicStream* __is = __inS.is();
    __is->startReadEncaps();
    ::Servant::ParamMap params;
    ::Servant::__readParamMap(__is, params);
    __is->endReadEncaps();
    ::IceInternal::BasicStream* __os = __inS.os();
    ::std::string result;
    ::Ice::Int __ret = Call(params, result, __current);
    __os->write(result);
    __os->write(__ret);
    return ::Ice::DispatchOK;
}
```
2. 修改slice2cpp后生成的文件
```
::Ice::Int
IceProxy::slxk::SLXKUserDao::AddUser(const ::slxk::SLXKUserDaoAddUserReq& __p_objReq, ::slxk::SLXKUserDaoAddUserResp& __p_objResp, const ::Ice::Context* __ctx)
{
    ZBNS_API::ZBNS_API_RAII report(m_caller, m_callee, 100);
    __checkTwowayOnly(__slxk__SLXKUserDao__AddUser_name);
    ::IceInternal::Outgoing __og(this, __slxk__SLXKUserDao__AddUser_name, ::Ice::Normal, __ctx);
    try
    {
        ::IceInternal::BasicStream* __os = __og.startWriteParams(::Ice::DefaultFormat);
        __os->write(__p_objReq);
        __og.endWriteParams();
    }
    catch(const ::Ice::LocalException& __ex)
    {
        report.set_return_code(-1);
        __og.abort(__ex);
    }
    if(!__og.invoke())
    {
        try
        {
            __og.throwUserException();
        }
        catch(const ::Ice::UserException& __ex)
        {
            ::Ice::UnknownUserException __uue(__FILE__, __LINE__, __ex.ice_name());
            throw __uue;
        }
    }
    ::Ice::Int __ret;
    ::IceInternal::BasicStream* __is = __og.startReadParams();
    __is->read(__p_objResp);
    __is->read(__ret);
    __og.endReadParams();
    return __ret;
}

::Ice::DispatchStatus
slxk::SLXKUserDao::___AddUser(::IceInternal::Incoming& __inS, const ::Ice::Current& __current)
{
    __checkMode(::Ice::Normal, __current.mode);
    ::Ice::Int __ret = 0;
    ::slxk::SLXKUserDaoAddUserReq __p_objReq;
    ::slxk::SLXKUserDaoAddUserResp __p_objResp;
    try
    {
        ::IceInternal::BasicStream* __is = __inS.startReadParams();
        __is->read(__p_objReq);
        __inS.endReadParams();
    }
    catch(...)
    {
        __ret=MMOcComm::LogicError::SLXK_COMMON_STD_EXCEPTION;
        SKGetPBMetaInfoWithImportedFile(::slxk::SLXKUserDaoSKBuiltinEmpty(), *__p_objResp.mutable_meta_info());
    }
    if(__ret == 0)
    {
         __ret = AddUser(__p_objReq, __p_objResp, __current);
    }
    ::IceInternal::BasicStream* __os = __inS.__startWriteParams(::Ice::DefaultFormat);
    __os->write(__p_objResp);
    __os->write(__ret);
    __inS.__endWriteParams(true);
    return ::Ice::DispatchOK;
}
```
以上的修改点就在于，read(req)的时候捕获异常，比如此时客户端传来的pb和服务端的有冲突，这里就可以防止服务端挂掉。  
另一个修改点就是当发生异常的时候，在响应中构建服务端此时的pb结构供客户端去解析。

## Protobuf
### 几个关键类的作用
- [Descriptor](https://developers.google.com/protocol-buffers/docs/reference/cpp/google.protobuf.descriptor)  
You can use a message's descriptor to learn at runtime what fields it contains and what the types of those fields are.
获取该Message所在的proto文件类: const FileDescriptor * file() const

- [FileDescriptor](https://developers.google.com/protocol-buffers/docs/reference/cpp/google.protobuf.descriptor)  
To get the FileDescriptor for a compiled-in file, get the descriptor for something defined in that file and call descriptor->file().

- [Message](https://developers.google.com/protocol-buffers/docs/reference/cpp/google.protobuf.message)  
获取Message的descriptor：const Descriptor* descriptor = foo->GetDescriptor();  
获取message某一个field的descriptor: const FieldDescriptor* text_field = descriptor->FindFieldByName("text");  
获取message的反射类（可以用来查看每个字段的具体值）：const Reflection* reflection = foo->GetReflection();  
根据反射类获取某个字段的值: assert(reflection->FieldSize(*foo, numbers_field) == 3);  
