---
layout: post
title: GamingAnyWhere - một open cloud gaming system
subtitle: ... updated on Feb. 18, 2018
image: /img/logo-ga.jpeg
tags: [System]
---


Cloud gaming là một xu hướng phát triển của cloud computing. Nếu như với online game truyền thống, user phải render toàn bộ màn chơi trên máy của mình (client), thì cloud gaming cho phép render màn chơi trên máy chủ (server). Khi đó, user thực chất chỉ đang xem đoạn video được tải về client. 
Bằng việc chuyển phần tác vụ nặng nhất (rendering) từ phía client sang server, user có thể chơi những game nặng trên một thiết bị có cấu hình thấp (thin client). Mặc dù hướng tiếp cận này đem lại cơ hội cho người dùng di động, nó cũng đối mặt với một số thách thức, trong đó đáng kể nhất là (i) giới hạn băng thông - bandwidth và (ii) thời gian đáp ứng của hệ thống - response interval. 

Trong bài viết này, tôi sẽ không đề cập tới những ưu nhược điểm của cloud gaming cũng như không thảo luận về hướng nghiên cứu liên qua. Thay vào đó tôi tập trung mô tả một hệ thống cloud gaming trong thực tế. Hệ thống được tôi lựa chọn là [GamingAnywhere](http://gaminganywhere.org), một open cloud gaming được phát triển từ 2013. 

1. Qua bài viết, chúng ta sẽ nắm được kiến trúc và hoạt động của một hệ thống **cloud gaming hoàn chỉnh trong thực tế**. Lưu ý rằng, hệ thống live streaming, trên thực tế, chỉ là một phần của hệ thống cloud gaming mà thôi. 
2. Hơn nữa, phần mô tả cho server sẽ đi kèm với phần **chú thích đoạn mã nguồn (source code)** tương ứng trong [mã nguồn của GamingAnyWhere](https://github.com/chunying/gaminganywhere). Tôi tin tưởng việc này sẽ tiết kiệm rất nhiều thời gian cho các nghiên cứu sinh và nhà phát triển, những người muốn tìm hiểu một hệt thống streaming trong thực tế.

## Kiến trúc và hoạt động của GamingAnyWhere

Một hệ thống GamingAnyWhere cơ bản gồm có một client, một server, và network. Ở đây, chúng ta chỉ xem xét context đơn giản nhất: (1) client connect tới server, gửi những signal liên quan tới bàn phím và chuột tới server; (2) server xử lý và chụp lại ảnh màn hình để gửi lại cho client. Thực tế trường hợp này khá giống TeamViewer. Điều khác biệt ở đây là cloud gaming cho phép độ tin cậy thấp hơn trong khi yêu cầu khả năng đáp ứng nhanh hơn.

<div class="imgcap" align = "center">
 <img src ="/img/ga-arch.png" width = "600">
 <div class = "thecap"> Hình 1: Kiến trúc của hệ thống GamingAnyWhere [1] </div>
</div>

Hình 1 biểu diễn kiến trúc của server và client. Hai luồng dữ liệu của GamingAnyWhere là data flow và control flow. Trong khi data flow được dùng để truyền multimedia data (A/V frames) từ server tới client thì control flow đi chiều ngược lại, được sử dụng để truyền thao tác của user từ client tới server. 

Tương ứng với hai luồng dữ liệu kể trên, server có hai nhóm tác vụ (task) chính. Task đầu tiên sẽ thu nhận các A/V frame của game, nén, và truyền chúng tới client thông qua data flow. Trong khi đó, task thứ hai trao đổi trực tiếp với game. Khi thu nhận được từ client các thao tác (action) của user, task này hành xử như user và chơi game bằng cách thực hiện lại các thao tác nhận được (VD: keyboard, mouse, joystick, ...). Chú ý, do hiện tại chưa có protocol nào được chuẩn hóa để truyền các thao tác của user, nên một transport protocol được thiết kế và triển khai.

Client của GamingAnyWhere thực tế là một game console được triển khai bằng các kết hợp RTSP/RTP player và keyboard/mouse logger. Hệ thống GamingAnyWhere cho phép độ đáp ứng thấp do sử dụng RTSP và RTP protocol. Theo đó, user có thể dùng media player (VD: VLC media player) xem/chơi game bằng cách đơn giản truy cập URL. Cũng cần chú ý là DASH (Dynamic Adaptive Streaming over HTTP - một chuẩn sử dụng HTTP truyền video qua mạng Internet) hiện tại không đạt được yêu cầu này. 

## Triển khai hệ thống GamingAnyWhere

GamingAnywhere bao gồm nhiều ***module*** và ***buffer*** (cả ở phía server lẫn client). Mỗi module, hoạt động như một thread độc lập, lấy đầu công việc (kết quả của module khác) trong buffer đầu vào, và trả lại kết quả vào buffer đầu ra. Buffer có thể coi là các entity của server/client. Trong khi đó module được coi là các worker; Cách thức hoạt động của các module sẽ được trình bày trong section này. 

Hệ thống hiện tại được triển khai dựa vào các thư viện [libavcodec/libavformat](http://ffmpeg.org), [live555](http://live555.com/liveMedia), và [SDL](http://www.libsdl.org). Thư viện libavcodec/libavformat là một phần của dự án [ffmpeg](http://ffmpeg.org), chịu trách nhiệm encode và decode A/V frames cả ở server và client. Thêm vào đó, nó cũng được sử dụng để điều khiển (handle) RTP ở phía server. Thư viện live555 được sử dụng để điều khiển (handle) RTP/RTSP ở phía client. Thư viện SDL được sử dụng để thu nhận tín hiệu audio, keyboard, mouse, joystick, 3D hardware thông qua OpenGL và 2D video frame buffer. GamingAnywhere cũng được sử dụng để render audio và video ở client. Tât cả thư viện được nêu trên đều có thể triển khai trên nhiều nền tảng bao gồm Windows, Linux, OS X, iOS, và Android.

### GamingAnyWhere server

Mối liên hệ giữa các module và buffer được biểu diễn trên Hình 2. Khởi đầu, bốn module RTSP server, audio source, video source, và input replayer sẽ được khởi tạo (init). Trong đó, module RTSP server sẽ thực thi luôn (start) và chờ client kết nối tới (bắt đầu từ luồng 1n và 1i trong Hình 2). Các module audio source và video source được đặt ở trạng thái chờ đợi. 

<div class="imgcap" align = "center">
 <img src ="/img/ga-server.png" width = "600">
 <div class = "thecap"> Hình 2: Liên hệ giữa các module, buffer, và network connections ở phía server [1] </div>
</div>

Trong mã nguồn GamingAnyWhere, file `ga-server-periodic.cpp` là file chính. Hàm `run_modules()` trong file này được kèm theo dưới đây. Cần phải chú ý rằng, trong mã nguồn GamingAnyWhere, giữa hai module video-source và video-encoder còn có module filter (mặc dù nó không được thể hiện trong Hình 2). 

```cpp
// Trích từ file ga-server-periodic.cpp - file main
// encoder_register_vencoder() chỉ cho đăng ký module video encoder chứ chưa chạy
// Điều tương tự xảy ra với encoder_register_vencoder()
int run_modules() {
    // replay
    struct RTSPConf *conf = rtspconf_global();
    if(conf->ctrlenable) {
        ga_run_single_module_or_quit("control server", ctrl_server_thread, conf);
    }
    // video
    if(m_vsource->start(prect) < 0)    exit(-1);
    if(m_filter->start(filter_param) < 0)    exit(-1);
    encoder_register_vencoder(m_vencoder, video_encoder_param);
    // audio
    if(ga_conf_readbool("enable-audio", 1) != 0) {
        encoder_register_vencoder(m_aencoder, audio_encoder_param);
    }
    // server
    if(m_server->start(NULL) < 0)    exit(-1);
    ...
}
```

```cpp
// Trích từ file server-live555.cpp 
// Câu lệnh m_server->start() phía trên mặc định tới live555.
// Một thread được tạo để handle liveserver_main()
static int live_server_start(void *arg) {
    if(pthread_create(&server_tid, NULL, liveserver_main, NULL) != 0) {
        ga_error("start live-server failed.\n");
        return -1;
    }
    return 0;
}
```

```cpp
// Trích từ file ga-liveserver.cpp
// Trong hàm chính của server, chúng ta khởi tạo các object phục vụ cho việc
// handle toàn bộ streaming session: scheduler, rtsp server, servermedia
// session và QoS server
void *liveserver_main(void *arg) {    
    ...
    TaskScheduler* scheduler = BasicTaskScheduler::createNew();
    env = BasicUsageEnvironment::createNew(*scheduler);
    ...
    RTSPServer* rtspServer 
        = RTSPServer::createNew(*env, rtspconf->serverport > 0 
            ? rtspconf->serverport : 8554, authDB);    
    ...
    ServerMediaSession* sms 
        = ServerMediaSession::createNew(*env, "desktop", "desktop", 
            "GamingAnywhere Server");
    ...
    qos_server_start();
    ...
    env->taskScheduler().doEventLoop();
    ...
}

```

Sau khi một client kết nối tới RTSP server, module video-encoder được khởi động (init và start). Video-encoder sẽ thông báo cho module video-source rằng nó đã sẵn sàng nén các raw frame nhận được. Khi đó module video-source mới chuyển sang trạng thái hoạt động và bắt đầu capture màn hình. Quá trình tương tự xảy ra với module audio-source. Cả audio và video frame được tạo ra đồng thời ở source theo thời gian thực. Data flow của audio và video frame lần lượt được biểu diễn như là một luồng từ 1a tới 5a và từ 1v tới 5v.

```cpp
// Trích từ file server-live555.cpp
// Sau khi client connect tới, module rtsp sẽ đăng kí client với server 
int live_server_register_client(void *ccontext) {
    if(encoder_register_client(ccontext) < 0)
        return -1;
    return 0;
}
```

```cpp
// Trích từ file encoder-common.cpp
// Cần thiết phải khởi tạo lại video và audio encoder trước khi chạy chúng
int encoder_register_client(void /*RTSPContext*/ *rtsp) {
    // Khởi tạo lại video và audio encoder 
    ...
    if(vencoder != NULL && vencoder->init != NULL) {
        if(vencoder->init(vencoder_param) < 0) { ... }
    }
    if(aencoder != NULL && aencoder->init != NULL) {
        if(aencoder->init(aencoder_param) < 0) { ... }
    }
    ...
    // chạy video và audio encoder
    if(vencoder != NULL && vencoder->start != NULL) {
        if(vencoder->start(vencoder_param) < 0) { ... }
    }
    if(aencoder != NULL && aencoder->start != NULL) {
        if(aencoder->start(aencoder_param) < 0) { ... }
    }
    ...
}
```

Các thông tin cụ thể về nguyên tắc hoạt động của các module trong server được trình bày trong [1]. Tuy nhiên, tài liệu này không trình bày cụ thể cách triển khai trong thực tế. Do đó, ***ở bài viết tiếp theo, tôi sẽ tập trung vào việc thiết kế và triển khai module trong hệ thống GamingAnyWhere.*** Bài viết này sẽ hướng tới các nhà phát triển hơn là các nghiên cứu sinh vì thiên về mô tả programing technique hơn.

###  GamingAnywhere Client

Như đã đề cập phía trên, client được coi là một điều khiển từ xa. Chức năng của client là hiển thị màn hình người chơi (được gửi từ server theo thời gian thực) và chuyển các hành xử của user đi. Sự liên hệ giữa các module và buffer trong client được thể hiện trên Hình 3. 

<div class="imgcap">
 <img src ="/img/ga-client.png" align = "center" width = "600">
 <div class = "thecap" align = "center"> Hình 3: Liên hệ giữa các module, buffer, và network connections ở phía client [1] </div>
</div>

Về cơ bản, client bao gồm hai thread chính: một được sử dụng để handle các hành xử của user (luồng dữ liệu bắt đầu từ 1i trong Hình 3) và một được dùng để hiển thị video/audio frame (luồng dữ liệu bắt đầu từ 1r). Kiến trúc và hoạt động của client, nhìn chung, là đơn giản hơn của server. Cách tiếp cận, do đó, mà dễ hơn so với server. Một lần nữa, các module của client đã được trình bày cụ thể trong [1] sẽ không được trình bày trong bài viết này.

## References

[1] Chun-Ying Huang, Cheng-Hsin Hsu, Yu-Chun Chang, and Kuan-Ta Chen, "GamingAnywhere: An Open Cloud Gaming System," *Proc. ACM Multimedia Systems,* Feb. 2013.