# OBS开发者文档

## OBS Studio后端设计

OBS Studio后端基于libobs库。Libobs提供主要的管线，音视频子系统，及通用插件框架。

### Libobs插件对象

Libobs采用模块化设计，通过添加模块来增加自定义功能。这里有4个libobs对象，你能为之创建插件：

- 源 - 用于在流上渲染音视频。例如捕获显示器/游戏/音频，播放视频，显示图片，或播放音频。源也能够用于产生音频和视频滤镜。

- 输出 - 用于输出当前渲染的音视频。串流和录制是两个输出示例，但不止有这两种方式。输出能够接受原始数据或编码数据。

- 编码器 - OBS特有的音视频编码器实现，用于输出时被使用。x264，NVENC，Quicksync是编码器实现的示例。 

- 服务 - 串流服务的自定义实现，用于输出音视频流。举例，你能有一个自定义的实现时：串流到Twitch，和登录YouTube，并使用它的API去做一些获取RTMP服务或控制频道的事情。

### Libobs工作线程

这里有三个主要的线程，其被创建于libobs初始化的时候：

-  `obs_graphics_thread`函数专门用于渲染，其位于`libobs/obs-video.c`

- `video-thread`函数专门用于视频编码/输出，其位于`libobs/media-io/video-io.c`

- `audio_thread`函数用于所有的音频处理/编码/输出，其位于`libobs/media-io/audio-io.c`

## 输出通道



## 插件

## 前端

### 初始化停止

去初始化`libobs`，你必须调用`obs_startup()`，`obs_reset_video()`，然后`obs_reset_audio()`。在这之后，通常模块会被加载。

你能手动地加载个别的模块，通过调用`obs_open_module()`。在加载之后，你必须调用`obs_init_module()`去初始化这个模块。

你能自动地加载模块，经由两个函数：`obs_add_module_path()`和`obs_load_all_modules()`。

在所有插件模块已经被加载之后，调用`obs_post_load_modules()`。

某些模块会使用一个配置存储目录，其会被作为参数传递到`obs_startup()`。

在关闭前端的时候，确保去释放所有引用的对象，数据，然后在调用`obs_shutdown()`。如果因为一些原因任何libobs对象还没被释放掉，它们会被自动的销毁，同时警告会被写入日志。

去检测任何的还没有被释放掉的内存，调用`bnum_allocs()`去获取这个分配存留数目。如果这个数目存留大于0，那么就存在内存泄漏。

```mermaid
sequenceDiagram
title: 
participant ui
participant libobs
ui-->libobs: obs_startup()
ui-->libobs: obs_reset_video()
ui-->libobs: obs_reset_audio()
ui-->libobs: obs_add_module_path()
ui-->libobs: obs_load_all_modules()
ui-->libobs: obs_post_load_modules()
ui-->libobs: 做些什么······
ui-->libobs: obs_shutdown()
```

### 重新设置视频参数

在初始化之后，只要没有活动的输出，视频参数能够被重新设置，通过调用`obs_reset_video()`。音频本来也想有这个功能，但当前不支持，一旦初始化便无法重新设置；libobs必须被完全地停止才能合理去设置音频设置。

### 显示

顾名思义，显示用于显示/预览窗格。你必须要有一个原生窗体句柄或标识才能去画在上面。

首先，你必须调用`obs_display_create()`去初始化显示，然后你必须调用`obs_display_add_draw_callback()`赋予一个回调函数。如果你需要去移除绘制回调函数，调用`obs_display_remove_draw_callback()`。

在回调函数中绘制的时候，去绘制这主要的预览窗体，调用`obs_render_main_texture()`。如果你需要去绘制一个指定的源到一个次要的显示上，调用`obs_source_inc_showing()`你能够递增它的“showing”状态 ，并调用`obs_source_video_render()`去绘制它，然后在它不用再显示在次要的显示上时，调用`obs_source_dec_showing()`。

如果这个显示需要去调整大小，调用`obs_display_resize()`。

如果这个显示需要去自定义背景颜色，调用`obs_display_set_background_color()`。

如果这个显示需要去临时禁用，调用`obs_display_set_ennabled()`去禁用，和`obs_display_enabled()`获取它的启用/禁用状态。

然后调用`obs_display_destroy()`去销毁这个显示在它不再被需要的时候。

### 显示源

源会被显示，在输出通道中串流/录制时。调用`obs_set_output_source()`，这里有64个通道可以被设置源，通道会以升序的形式画在彼此的上面。通常，一个普通的源不应该被直接的设置。你应该要使用一个场景。

去绘制一个或更多的源并用一个指定的变换应用到它，场景是被用到的。去创建一个场景，通过调用`obs_scene_create()`。子源被当作场景项引用，然后指定的变换被应用到那些场景项。场景项不是源，而是源的容器；这相同的源能够被多个场景项应用在相同的场景中，或被引用到多个场景中。去创建一个场景项并引用一个源，调用`obs_scene_add()`，它返回一个新的场景项引用。
