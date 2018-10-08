# 简介

## GOMQ 是一个由 GO 语言编写的开源消息中间件，它具有以下特性 ：

+ 作为消息服务器运行，在服务器上部署消息队列，由生产者客户端向服务器的队列发送消息或消费者客户端从服务器的队列接收消息。

![MQServer](https://github.com/garychenc/gomq/blob/master/doc/img/MQ-Server.png "MQ服务器示意图")

+ 消息队列使用文件持久化消息，保证已经发送到服务器端的消息的可靠存储，假如服务器重启或者突然崩溃，不会导致消息丢失。

+ 队列持久化文件采用 Append Only 的形式存储消息，消息发送到队列之后会被添加到持久化文件中，即使消息被消费了也不会从文件中删除，保证新的消费者可从头重新读取队列中的所有消息（待开发功能）。可设置消息在队列中最长的保存时间，到达最长保存时间之后，新写入的消息会被写在文件的开头位置，覆盖掉过期的消息。所以，文件会循环写入，不会无止境的增长。

+ 每个消费者都有一个在当前队列最后消费位置的指针，指向最后一个消费的消息，并且独立存储在消费者元数据文件中。每次消费者从队列中消费一个消息，处理完毕之后，调用 Commit 函数，即可将最后消费指针写入文件，保证无论消费者客户端重启，或者服务器重启，再次从队列中消费消息的时候都从下一个消息的位置开始读取新的消息。

+ 消费者可 Reset 最后消费位置的指针，让其重新指向最后一次 Commit 的消息位置。也即是说可以从队列中接收多个消息，进行处理，假如消息处理失败可以重新接收相同的消息再次进行处理。

+ 创建消费者的时候需要指定消费者的名称，当多个名称**相同**的消费者消费同个队列的时候，它们共享相同的消息指针。因此，接收消息的时候，队列中的消息采用分发的形式分发给多个消费者，每个消费者接收到一个不同的消息。当多个名称**不同**的消费者消费同个队列的时候，由于它们有各自的消息指针，所以，每个消费者都可以接收到队列中的所有消息。

![Consumer-Offset](https://github.com/garychenc/gomq/blob/master/doc/img/consumer-offset.png "消费者指针示意图")

+ 生产者、消息队列服务器、消费者可分别处于不同服务器上，它们之间通过 TCP 长连接进行通讯。客户端与服务器端之间由独立的心跳来保持长连接，并且进行对端的健康检查。该长连接具有可靠通讯的功能，保证消息能够可靠的发给对端，消除由于网络闪断等带来的各种不可靠传输问题。

+ 该消息中间件通过可靠的网络传输、保存到文件的消息存储、Receive - Process - Commit的消息消费模式等机制，保证消息从发生到存储到消费各个环节的可靠性。

## GOMQ 目前还有以下特性待开发 ：

+ 目前队列还不支持多个 Partition，有多个生产者或消费者的情况下只能往同个 Partition 发送消息或从同个 Partition 中竞争接收消息。这样会影响性能和消息接收的可靠性。

+ 目前消息服务器还不支持集群，只能运行单个消息服务器。接下来会采用 Master Server - Data Server 的模式设计并开发消息服务器的集群功能。

+ 目前还没有完善的监控和管理系统可以监控服务器的各种运行指标，并且对服务器进行管理。

+ 目前只有 GO 语言开发的生产者和消费者客户端，还不能支持其它语言。

# 快速入门

接下来将会从服务器构建、配置服务器、部署消息队列、启动服务器、创建消息生产者、创建消息消费者等步骤介绍如何快速使用 GOMQ，本入门假设你熟悉 GO 语言开发。

+ 由于 GOMQ 是采用 GO 语言编写，并且没有提供已编译版本，所以，在使用之前需要先 checkout 源代码，并且在目标操作系统中使用 GO 编译器编译源代码。

+ 安装 GO 开发环境，[Windows & Linux 安装说明](https://www.jianshu.com/p/b6f34ae55c90)，[MAC 安装说明](https://www.jianshu.com/p/ae872a26b658)（安装过程可能需要翻墙）。

+ 由于 GOMQ 使用 DEP 进行依赖管理，所以需要安装 DEP，用于下载项目中用到的第三方组件。[DEP 安装说明](https://studygolang.com/articles/10589) 。

+ 使用 GIT Checkout 源代码，代码路径：https://github.com/garychenc/gomq.git 。假设 Checkout 之后源代码的存放根目录（目录下有 README.md 文件的那个目录）的路径为 CODE_ROOT 。

+ 将 CODE_ROOT/mqd 路径添加到 GOPATH 环境变量中。

+ 在命令行控制台中进入 CODE_ROOT/mqd/src/client 目录，运行 dep ensure 命令。稍微等待之后，可看到 CODE_ROOT/mqd/src/client 目录多了一个 vendor 目录，即表示 dep ensure 命令运行成功，拉取到 client 所需的第三方组件（beego 日志组件）。

+ 在命令行控制台中进入 CODE_ROOT/mqd/src/common 目录，运行 dep ensure 命令。稍微等待之后，可看到 CODE_ROOT/mqd/src/common 目录多了一个 vendor 目录，即表示 dep ensure 命令运行成功。

+ 在命令行控制台中进入 CODE_ROOT/mqd/src/server 目录，运行 dep ensure 命令。稍微等待之后，可看到 CODE_ROOT/mqd/src/server 目录多了一个 vendor 目录，即表示 dep ensure 命令运行成功，拉取到 server 所需的第三方组件（beego 日志组件）。

+ 在命令行控制台中进入 CODE_ROOT/mqd 目录，运行 go build -o ./bin/gomqd server/main 命令，编译可执行文件到 bin 目录，文件名为 gomqd 。

+ 将 CODE_ROOT/mqd/src/server/main 中的 config，logs，store 目录拷贝到 CODE_ROOT/mqd/bin 目录。

+ 打开 CODE_ROOT/mqd/bin/config/server.yml 配置文件，将 ListeningAddress 配置项的服务器监听 IP 改为本机 IP，或者不改动亦可。从配置文件中可以看到服务器默认部署了三条测试队列，部署新队列的方法请参考配置文件的注释。

+ 在命令行控制台中进入 CODE_ROOT/mqd/bin 目录，运行 ./gomqd 命令，看到 “-- MQ SERVER STARTED !! --” 日志则 GOMQ 服务器启动正常。

+ 查看 CODE_ROOT/mqd/bin/logs 目录，可以看到服务器运行时日志存放在该目录。包含：mq.log 和 main.log 两个日志文件，mq.log 记录消息服务器运行时的各种日志，main.log 记录 main 函数的启动日志。

+ 查看 CODE_ROOT/mqd/bin/store 目录，可以看到所有队列的消息持久化文件存放在该目录。消费者的元数据文件也会存放在该目录，目前没有消费者，所以没有任何消费者元数据文件。

+ 至此 GOMQ 服务器已成功编译并启动，可将 CODE_ROOT/mqd/bin 目录下的所有文件和子目录移到其它任何不包含中文路径的目录，也可正常启动。接下来将介绍客户端示例，通过这个示例可了解如何使用 GOMQ 的客户端与 GOMQ 服务器进行交互。

+ 暂时将 CODE_ROOT/mqd 路径从 GOPATH 环境变量中移除，然后将 CODE_ROOT/client-example 路径添加到 GOPATH 环境变量中。

+ 在命令行控制台中进入 CODE_ROOT/client-example/src/pe 目录，运行 dep ensure 命令。

+ 在命令行控制台中进入 CODE_ROOT/client-example/src/ce 目录，运行 dep ensure 命令。

+ 在命令行控制台中进入 CODE_ROOT/client-example 目录，运行 go test pe 命令。然后在 CODE_ROOT/client-example/src/pe 目录下可看到 producer.log 文件，记录着生产者示例函数的消息发送日志。

+ 在命令行控制台中进入 CODE_ROOT/client-example 目录，运行 go test ce 命令。然后在 CODE_ROOT/client-example/src/ce 目录下可看到 consumer.log 文件，记录着消费者示例函数的消息接收日志。

+ 可通过查看 CODE_ROOT/client-example/src 目录下的源代码，了解 GOMQ 客户端的使用方法和 GOMQ 客户端各接口的详细 API 说明。

# 联系方式

Gary CHEN : email : gary.chen.c@qq.com

# 系统架构简介

**系统整体架构图如下 ：**

![WholeArchitecture](https://github.com/garychenc/gomq/blob/master/doc/img/whole-architecture.png "系统整体架构图")

系统从整体上分成服务器端和客户端，服务器端从下至上包含以下组件：

+ Server Side Long Connection Network Layer，该组件用于对客户端 TCP 长连接进行管理，接收所有进入服务器的网络请求，结合 Network Transfer Protocol Process 组件对请求进行解码，然后将请求分发到对应的处理器进行处理。该组件同时还用于维持客户端与服务器端之间的长连接心跳。

+ MQ Command & Message Protocol Process，在下层组件对网络传输包进行解码之后，就将解码之后的请求发送到该组件对应的处理器进行请求处理。在处理过程中，需要对请求中包含的命令类型、消息内容等进行解码，然后执行相应的命令，例如：将消息添加到队列的末尾或者从队列中取出消息并且发送到客户端。

+ Queue Container，服务器端创建的队列、生产者、消费者对象的容器，每个客户端创建的生产者或消费者都对应着一个服务器端的生产者或消费者，并且通过远程调用的形式调用到服务器端对象的相应方法。这些服务器端的对象由 Queue Container 管理其生命周期。从队列的部署，生产者、消费者的创建到队列、生产者、消费者的关闭。

+ Producer，服务器端对应的生产者对象，用于执行将消息发送到队列的逻辑。

+ Consumer，服务器端对应的消费者对象，用于执行从队列中取出消息，管理最后消费消息指针等逻辑。

+ Queue，基于文件实现的循环数组列表（Circle Array List）。把一个大文件看成整块的大内存，在此基础上基于文件操作 API 实现：空间分配管理、空间循环使用、空间写入、空间数据读取、列表元素持久化、反持久化等逻辑。

+ Message & Consumer MetaData Storage，该组件属于 Queue 组件和 Consumer 组件的一部分，封装对文件操作的逻辑。

+ MQ Server，服务器端所有对象的容器，管理着整个服务器的启动、停止，以及所需的各种资源的申请和销毁。

客户端包含以下组件：

+ Client Side Long Connection Network Layer、Network Transfer Protocol Process、MQ Command & Message Protocol Process 这些组件与服务器端的 Server Side Long Connection Network Layer、Network Transfer Protocol Process、MQ Command & Message Protocol Process 组件一一对应，完成客户端的长连接管理、网络请求协议编解码、网络请求发送、网络请求处理等逻辑。

+ MQ Client，实现 MQ 客户端业务逻辑的封装。

+ MQ Client API，对外提供操作 MQ 服务器的 API 接口。

**基于文件的队列、消费者名称、消息指针示意图 ：**

![Consumer-Offset](https://github.com/garychenc/gomq/blob/master/doc/img/consumer-offset.png "消费者指针示意图")

+ 每个客户端创建的消费者对象都对应着一个服务器端的消费者对象，根据消费者名称形成对应关系。假如多个客户端的消费者对象采用相同的消费者名称，则这些客户端的消费者对象都对应着同个服务器端的消费者对象，它们共享着相同的消息指针，所以当一个客户端消费者接收一个消息之后，服务器端消费者的消息指针向前，另外一个客户端消费者只能接收下一个消息。而不同名称的客户端消费者对应着不同的服务器端消费者，使用不同的消息指针，所以各自都可以接收队列中所有的消息。

+ 每个服务器端消费者包含两个消息指针，1. 当前已接收消息指针，指向最后一个已成功获取的消息的位置，2. 最后 Commit 消息指针，指向最后一次调用 Commit 时的当前已接收消息指针。Reset 消费者的时候将当前已接收消息指针重新指向最后 Commit 消息指针。

+ 队列只是一个按顺序存储消息的文件，对外的表现形式就像一个循环数组列表（Circle Array List）一样，不删除消息，不更新消息。消费者自己保存当前消费指针，通过指针指向的位置查询队列中的消息内容。
