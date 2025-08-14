---
title: 在humanoid-gym训练以及演示过程中记录的视频无法在Vscode中播放
date: 2025-08-14 12:05:06
tags:
    - 强化学习
banner_img: ../images/kisaki4.jpg
---
<script src="https://fastly.jsdelivr.net/gh/misaka0502/live2d-widget@V0.2/autoload.js"></script>
<!-- <script src="/live2d-widget/autoload.js"></script> -->
用Humanoid-gym这套框架训练之后，运行其提供的play.py记录下的视频无法在Vscode中播放。由于我在远程服务器上训练，要想查看演示效果就得下载到本地再播放，十分不方便。

humanoid-gym原有的记录视频的代码如下：

```python
#------------相机初始化-----------#
camera_properties = gymapi.CameraProperties()
camera_properties.width = 1280
camera_properties.height = 720
h1 = env.gym.create_camera_sensor(env.envs[0], camera_properties)
camera_offset = gymapi.Vec3(1, -1, 0.5)
camera_rotation = gymapi.Quat.from_axis_angle(gymapi.Vec3(-0.3, 0.2, 1),
                                            np.deg2rad(135))
actor_handle = env.gym.get_actor_handle(env.envs[0], 0)
body_handle = env.gym.get_actor_rigid_body_handle(env.envs[0], actor_handle, 0)
env.gym.attach_camera_to_body(
    h1, env.envs[0], body_handle,
    gymapi.Transform(camera_offset, camera_rotation),
    gymapi.FOLLOW_POSITION)
#------------video writer初始化---------#
fourcc = cv2.VideoWriter_fourcc(*"mp4v")
video_dir = os.path.join(LEGGED_GYM_ROOT_DIR, 'videos')
experiment_dir = os.path.join(LEGGED_GYM_ROOT_DIR, 'videos', train_cfg.runner.experiment_name)
dir = os.path.join(experiment_dir, datetime.now().strftime('%b%d_%H-%M-%S')+ args.run_name + '.mp4')
if not os.path.exists(video_dir):
    os.mkdir(video_dir)
if not os.path.exists(experiment_dir):
    os.mkdir(experiment_dir)
video = cv2.VideoWriter(dir, fourcc, 50.0, (1280, 720))

···
#-------------添加视频帧---------------#
env.gym.fetch_results(env.sim, True)
env.gym.step_graphics(env.sim)
env.gym.render_all_camera_sensors(env.sim)
img = env.gym.get_camera_image(env.sim, env.envs[0], h1, gymapi.IMAGE_COLOR)
img = np.reshape(img, (720, 1280, 4))
img = cv2.cvtColor(img, cv2.COLOR_RGB2BGR)
```

其记录的视频无法在Vscode中播放，导致我需要将其下载到本地才能播放，非常麻烦。最后发现其根本原因在于**编码格式（Codec）**的问题。代码中使用了`fourcc=cv2.VideoWriter_fourcc(*"mp4v")`，这会使用 MPEG-4 Part 2 视频编解码器。然而，VSCode 内置的视频播放器对视频编码格式的支持非常有限。可以尝试将`fourcc`参数修改为 `avc1` 或 `X264`。avc1 是 H.264 编码的 FourCC 之一，通常有更好的兼容性。

然而，环境中安装的OpenCV版本在编译时**没有包含或链接到支持 H.264 编码的后端库（如 FFmpeg）（尤其是在通过 pip 安装标准 opencv-python 包时，因为 H.264 编码器涉及软件专利许可问题。）**

测试代码：

```python
import cv2
import numpy as np
import os

def test_codec(fourcc_str):
    """测试指定的 FourCC 编码器是否可用"""
    fourcc = cv2.VideoWriter_fourcc(*fourcc_str)
    filename = f"test_{fourcc_str}.mp4"
    # 尝试创建一个 VideoWriter 对象
    writer = cv2.VideoWriter(filename, fourcc, 30.0, (10, 10))
    
    # 关键检查点: isOpened()
    if writer.isOpened():
        print(f"✅ 编码器 '{fourcc_str}' 可用。")
        # 写入一帧以确保文件可以创建
        writer.write(np.zeros((10, 10, 3), dtype=np.uint8))
        writer.release()
        if os.path.exists(filename):
            os.remove(filename) # 清理测试文件
        return True
    else:
        # isOpened() 返回 False 是解码器不可用的明确信号
        print(f"❌ 编码器 '{fourcc_str}' 不可用或初始化失败。")
        return False

print("--- 开始测试您环境中可用的视频编码器 ---")
print("OpenCV 版本:", cv2.__version__)

# 测试常用的编码器
test_codec("mp4v") # MPEG-4
test_codec("avc1") # H.264/AVC
test_codec("X264") # H.264/AVC 的另一种形式
print("----------------- 测试结束 -----------------")
```

输出：

```bash
OpenCV 版本: 4.12.0
✅ 编码器 'mp4v' 可用。
[ERROR:0@0.030] global cap_ffmpeg_impl.hpp:3207 open Could not find encoder for codec_id=27, error: Encoder not found
[ERROR:0@0.030] global cap_ffmpeg_impl.hpp:3285 open VIDEOIO/FFMPEG: Failed to initialize VideoWriter
❌ 编码器 'avc1' 不可用或初始化失败。
OpenCV: FFMPEG: tag 0x34363258/'X264' is not supported with codec id 27 and format 'mp4 / MP4 (MPEG-4 Part 14)'
OpenCV: FFMPEG: fallback to use tag 0x31637661/'avc1'
[ERROR:0@0.031] global cap_ffmpeg_impl.hpp:3207 open Could not find encoder for codec_id=27, error: Encoder not found
[ERROR:0@0.031] global cap_ffmpeg_impl.hpp:3285 open VIDEOIO/FFMPEG: Failed to initialize VideoWriter
❌ 编码器 'X264' 不可用或初始化失败。
----------------- 测试结束 -----------------
```

一种方法是想办法通过Conda安装一个更完整的OpenCV，另一种方法是用imageio库，仅需修改几行：

```python
video = cv2.VideoWriter(dir, fourcc, 50.0, (1280, 720))
修改为：
video = imageio.get_writer(dir, fps=50, codec='libx264', quality=8)

# 记录视频帧时：
img = np.frombuffer(img, dtype=np.uint8).reshape(camera_properties.height, camera_properties.width, 4) # img是从gym中获取的图像
img = img[..., :3] # 丢弃 Alpha 通道
video.append_data(img)
```

这样就能在Vscode中播放了。