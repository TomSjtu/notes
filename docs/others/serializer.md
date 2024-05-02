# 数据序列化

数据序列化是将数据结构转换为易于存储和传输的形式的过程，在另一个环境中可以反序列化回原始数据结构。

Protocol Buffers(简称Protobuf)，是谷歌开发的序列化框架。与开发语言无关，和平台无关，具有良好的可扩展性。Protobuf和所有的序列化框架一样，都可以用于数据存储、通讯协议。

使用Protobuf序列化的结果体积远比XML、JSON要小，且速度快很多，因此是数据传输的首选。

## 使用方法

1. 定义.proto文件：

    ```protobuf title="person.proto"
    syntax = "proto3";

    message Person {
        string name = 1;
        int32 id = 2;
        bool has_pets = 3;
    }
    ```

2. 编译.proto文件：

    ```shell
    protoc --cpp_out=. person.proto
    ```

3. 定义序列化类：

    ```CPP
    #include "person.pb.h"

    int main() {
        // 创建一个 Person 对象
        Person person;
        person.set_name("Alice");
        person.set_id(1234);
        person.set_has_pets(true);

        // 序列化到内存中
        std::string serialized_data;
        person.SerializeToString(&serialized_data);

        // 反序列化从内存中
        Person deserialized_person;
        deserialized_person.ParseFromString(serialized_data);

        // 使用反序列化的对象
        std::cout << "Name: " << deserialized_person.name() << std::endl;
        std::cout << "ID: " << deserialized_person.id() << std::endl;
        std::cout << "Has pets: " << deserialized_person.has_pets() << std::endl;

        return 0;
    }
    ```