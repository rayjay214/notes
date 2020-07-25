makefile详解，包含了大部分的场景
```
HOME_PATH:=$(shell echo ~)
OUT_BIN_PATH:=$(HOME_PATH)/bin
TARGET=$(OUT_BIN_PATH)/platform/comm/libmmcomm.a
ROOT_STR_PATH:=$(HOME_PATH)/platform
COMM_STR_PATH:=$(ROOT_STR_PATH)/comm
PUBLIC_STR_PATH:=$(ROOT_STR_PATH)/public
PROTO_CC:=protoc
PROTO_CC3:=protoc3
PROTO_FLAGS:=--cpp_out=$(OUT_BIN_PATH)/platform/comm

CC=g++

#如果需要调用其他RPC，请在这里添加文件夹列表,以空格隔开
CLIENT_PATH:=

CFLAGS= -g -Wall -std=c++11 -Wno-deprecated -Wno-write-strings -Wno-comment -Wcpp -fPIC
INCLUDE=-I./
INCLUDE+=-I/usr/include/mysql
INCLUDE+=-I/usr/local/openssl/include
INCLUDE+=-I/usr/local/protobuf/include
INCLUDE+=-I$(COMM_STR_PATH)
INCLUDE+=-I$(HOME_PATH)
INCLUDE+=-I$(OUT_BIN_PATH)/platform/comm
INCLUDE+=-I$(COMM_STR_PATH)/comm2 
INCLUDE+=-I$(PUBLIC_STR_PATH)
INCLUDE+=-I$(PUBLIC_STR_PATH)/zbns
INCLUDE+=-I$(OUT_BIN_PATH)
INCLUDE+=-I$(COMM_STR_PATH)/3rd/zeroc_ice/include
INCLUDE+=-I$(COMM_STR_PATH)/3rd/ice-protobuf
LINK_LIBS+= -lcoev_comm -lcoev_misc -lmysqlclient -ljson-c -L/usr/local/protobuf/lib -lprotobuf -lrt -L /usr/local/openssl/lib -lssl -lmysqlpp
LINK_LIBS+= -L$(COMM_STR_PATH)/appplatform -llibrary_log3 -llogclient3 -llibrary_the3_tinyxml 
LINK_LIBS+= -L/usr/local/zblib -lIce -lIceUtil -llog4cxx
LINK_LIBS+= -L$(OUT_BIN_PATH)/platform/public -lzbpublic
LINK_LIBS+= -lpthread -luuid -lcrypto -ldl -lbfd -liberty -lz
LINK_LIBS+= -lboost_regex -lboost_date_time -lboost_filesystem -lboost_system -lboost_thread

PROTOINCLUDE:=-I /usr/local/protobuf/include -I $(COMM_STR_PATH) -I . -I $(HOME_PATH)
DIRS:=$(shell find . -maxdepth 10 -type d)  #获取当前目录下的所有子目录

DIRS2:=$(shell find . -maxdepth 10 -type d | grep -v "3rd/protobuf")
DIRS3:=$(shell find . -maxdepth 10 -type d | grep "3rd/protobuf")

PROTOSRC:=$(foreach dir, $(DIRS2), $(wildcard $(dir)/*.proto))
PROTOSRC+=$(foreach dir, $(CLIENT_PATH), $(wildcard $(dir)/*.proto))
#转换成绝对路径，此处将绝对路径名前的$(HOME_PATH)去掉
PROTOSRC:=$(shell for file_name in $(PROTOSRC); do readlink -f $$file_name | sed -n -r "s|^"$(HOME_PATH)/"(.*)|\1|p";done)
#去掉绝对路径的代码根目录前缀,替换成out目录
PROTOSRC:=$(addprefix $(OUT_BIN_PATH)/,$(PROTOSRC))
PROTOSRC:= $(PROTOSRC:%.proto=%.pb.cc)
OBJS_CC:= $(PROTOSRC:%.cc=%.o)

PROTOSRC3:=$(foreach dir, $(DIRS3), $(wildcard $(dir)/*.proto))
PROTOSRC3+=$(foreach dir, $(CLIENT_PATH), $(wildcard $(dir)/*.proto))
#转换成绝对路径
PROTOSRC3:=$(shell for file_name in $(PROTOSRC3); do readlink -f $$file_name | sed -n -r "s|^"$(HOME_PATH)/"(.*)|\1|p";done)
#去掉绝对路径的代码根目录前缀,替换成out目录
PROTOSRC3:=$(addprefix $(OUT_BIN_PATH)/,$(PROTOSRC3))
PROTOSRC3:= $(PROTOSRC3:%.proto=%.pb.cc)
OBJS_CC3:= $(PROTOSRC3:%.cc=%.o)

SOURCES_CPP:=$(foreach dir, $(DIRS), $(wildcard $(dir)/*.cpp))
SOURCES_CPP+=$(foreach dir, $(CLIENT_PATH), $(dir)/$(shell basename $(dir)).cpp)
#转换成绝对路径
SOURCES_CPP:=$(shell for file_name in $(SOURCES_CPP); do readlink -f $$file_name | sed -n -r "s|^"$(HOME_PATH)/"(.*)|\1|p";done)
#去掉绝对路径的代码根目录前缀,替换成out目录
SOURCES_CPP:=$(addprefix $(OUT_BIN_PATH)/,$(SOURCES_CPP))

#替换后缀，格式$(VAR:A=B),将SOURCE_CPP中的.cpp文件全部替换成.o，并赋值给OBJS_CPP,=号前面的%作用到了=号后面
OBJS_CPP:=$(SOURCES_CPP:%.cpp=%.o)
$(info $$OBJS_CPP is [${OBJS_CPP}]) #打印调试

SOURCES_C:=$(foreach dir, $(DIRS), $(wildcard $(dir)/*.c))
#转换成绝对路径
SOURCES_C:=$(shell for file_name in $(SOURCES_C); do readlink -f $$file_name | sed -n -r "s|^"$(HOME_PATH)/"(.*)|\1|p";done)
#去掉绝对路径的代码根目录前缀,替换成out目录
SOURCES_C:=$(addprefix $(OUT_BIN_PATH)/,$(SOURCES_C))
OBJS_C:=$(SOURCES_C:%.c=%.o)

OBJS_S:=$(OUT_BIN_PATH)/platform/comm/colib/coctx_swap.o
##############################
#  change OBJECT to set execute file name
##############################



all : $(TARGET) 
	if [ ! -d $(@D) ]; then mkdir -p $(@D); fi;

$(TARGET): $(OBJS_CC) $(OBJS_CPP) $(OBJS_C) $(OBJS_S)
	ar cru $(TARGET) $(OBJS_CC) $(OBJS_CPP) $(OBJS_C) $(OBJS_S)
	ranlib $(TARGET)

#后面的冒号两边也是表示替换，如同上面的$(VAR:A=B)，
#假如OUT_BIN_PATH=/home/rayjay/bin，HOME_PATH=/home/rayjay
#/home/rayjay/bin/platform/test/protobuf/test_reflection.o就会被替换成/home/rayjay/platform/test/protobuf/test_reflection.cpp
#此时%=platform/test/protobuf/
$(OBJS_CPP):$(OUT_BIN_PATH)/%.o:$(HOME_PATH)/%.cpp
	@echo "  build  CPP  $@"
  #$@表示目标文件集，表示"$@"的目录部分（无斜杠），如果"$@"值是"dir/foo.o"，那么"$(@D)"就是"dir"
	if [ ! -d $(@D) ]; then mkdir -p $(@D); fi;\
	$(CC) $(CFLAGS) -o $@ -c $^ $(INCLUDE) $(LINK_LIBS)

$(OBJS_C):$(OUT_BIN_PATH)/%.o:$(HOME_PATH)/%.c
	@echo "  build  C  $@"
	if [ ! -d $(@D) ]; then mkdir -p $(@D); fi;\
	$(CC) $(CFLAGS) -o $@ -c $^ $(INCLUDE) $(LINK_LIBS)

$(OBJS_CC):%.o:%.cc
	@echo "  build  CC  $@"
	if [ ! -d $(@D) ]; then mkdir -p $(@D); fi;\
	$(CC) $(CFLAGS) -o $@ -c $^ $(INCLUDE) $(LINK_LIBS)

$(OBJS_S):$(OUT_BIN_PATH)/%.o:$(HOME_PATH)/%.S
	@echo "  build  S  $@"
	if [ ! -d $(@D) ]; then mkdir -p $(@D); fi;\
	$(CC) $(CFLAGS) -o $@ -c $^ $(INCLUDE) $(LINK_LIBS)

$(PROTOSRC): $(OUT_BIN_PATH)/%.pb.cc:$(HOME_PATH)/%.proto
	if [ ! -d $(@D) ]; then mkdir -p $(@D); fi;\
	$(PROTO_CC) $(PROTO_FLAGS) $(PROTOINCLUDE) $^ 
	@echo 'generate file' $@

$(PROTOSRC3): $(OUT_BIN_PATH)/%.pb.cc:$(HOME_PATH)/%.proto
	if [ ! -d $(@D) ]; then mkdir -p $(@D); fi;\
	$(PROTO_CC3) $(PROTO_FLAGS) $(PROTOINCLUDE) $^ 
	@echo 'generate file' $@

clean:
	rm -f $(PROTOSRC) $(PROTOSRC:%.pb.cc=%.pb.h)
	rm -f $(PROTOSRC3) $(PROTOSRC3:%.pb.cc=%.pb.h)
	rm -f $(TARGET) $(OBJS_CPP) $(OBJS_C) $(OBJS_CC) $(OBJS_S)
```
