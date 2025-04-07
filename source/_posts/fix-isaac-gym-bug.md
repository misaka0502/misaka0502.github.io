---
title: 记录一个在运行isaac gym时遇到的bug和修bug过程:cudaExternamMemoryGetMappedBuffer failed on…
date: 2025-04-07 12:53:44
tags:
    - Python
    - IsaacGym
banner_img: /images/bg/mari1.jpg
---
<script src="https://fastly.jsdelivr.net/gh/misaka0502/live2d-widget@V0.2/autoload.js"></script>
<!-- <script src="/live2d-widget/autoload.js"></script> -->

# Bug的出现

今天在跑项目时，发现会报错：

```bash
Traceback (most recent call last):
	...
	color_obs = torch.stack(color_obs)[..., :-1] #RGBA->RGB
TypeError: expected Tensor as elment 0 in argument 0, but got NoneType
```

大概情况是，在环境reset时，读取不到相机的image，全是None。但是在以前从来没出现过这个报错，而且距离上一次运行也没有对环境代码做什么修改。

然后尝试在初始化环境时输出相关信息，发现了一些很奇怪的ERROR：

```bash
[Error] [carb.gym.plugin] cudaExternamMemoryGetMappedBuffer failed on rgbImage buffer with error 101
[Error] [carb.gym.plugin] cudaExternamMemoryGetMappedBuffer failed on depthImage buffer with error 101
[Error] [carb.gym.plugin] cudaExternamMemoryGetMappedBuffer failed on segmentationImage buffer with error 101
[Error] [carb.gym.plugin] cudaExternamMemoryGetMappedBuffer failed on optical flow buffer with error 101
*** Can't create empty tensor
```

大致意思是**cudaExternalMemoryGetMappedBuffer**在尝试映射 **RGB 图像、深度图、分割图、光流图**的 CUDA 内存时失败，错误码为 **101**，填充刚体张量也失败。

此时我用的是卡9（RTX3090），当我换到没有进程在跑的卡3时，又能正常运行了，暂时不知道是何原因，多次尝试过后，发现此时在某几张卡上会报这个错误，而其他卡上又能正常运行代码。

一段时间后，我发现服务器上所有的卡都会报这个错误，自己的代码直接无法运行了🙃。

目前还是没有找到具体原因和根本的解决方法，在网上检索了很久，发现很多人都遇到过这个bug，但是没有找到一个通用的解释和解决办法，只能推测大概是Isaac Gym和CUDA底层的一些问题，但是找到了一个避开此问题的方法。

# Bug的解决过程

最后定位到两处问题。

一是发现是在创建相机传感器时，相机属性设置如下：

```python
camera_cfg = gymapi.CameraProperties()
camera_cfg.enable_tensors = True
camera_cfg.width = self.img_size[0]
camera_cfg.height = self.img_size[1]
camera_cfg.near_plane = 0.001
camera_cfg.far_plane = 2.0
camera_cfg.horizontal_fov = 40.0 if self.resize_img else 69.4
self.camera_cfg = camera_cfg
```

问题就出现在了`camera_cfg.enable_tensors = True` 这一句上，这句代码指示 Isaac Gym **将该摄像头的图像数据直接输出为 GPU 张量 (tensors)**，而不是默认的 CPU NumPy 数组格式。只要设置该值为True，就一定会报错，而将其设置为False则不会出现；且当enable_tensor=True时，运行到以下语句

```python
self.isaac_gym.prepare_sim(self.sim)
```

时，还会报**Failed to fill rigid body state tensor**（我不清楚填充刚体张量为何会与相机设置有关），而设置enable_tensor=False后，也不会报这个错误了。

二是在原来在获取图像时，用的是以下方式

```python
tensor = gymtorch.wrap_tensor(
    self.isaac_gym.get_camera_image_gpu_tensor(
        self.sim, env, handle, render_type
    )
)
```

原本的代码是通过这样的方式直接从相机传感器获取张量，这本来是提高效率的办法，但是由于前面创建相机时就有问题，这里自然也获取不到图像。

既然如此，那我就不用原来的那一套相机传感器，而是换了一种写法，直接重写环境获取相机图像的方式。

首先在创建相机时更改`camera_cfg.enable_tensors = False` ，然后在`step`获取`observation` 时，先加上：

```python
self.isaac_gym.fetch_results(self.sim, True)
self.isaac_gym.step_graphics(self.sim)
self.isaac_gym.render_all_camera_sensors(self.sim)
```

这三行很重要，确保获取相机图像。然后用`gym.get_camera_image()` 方法获取图像。举个例子：

```python
class env(gym.Env):
		# 创建环境
		self.isaac_gym = gymapi.acquire_gym()
		sim = self.isaac_gym.create_sim(
			compute_device_id,
		    graphics_device_id,
		    gymapi.SimType.SIM_PHYSX,
		    sim_config["sim_params"],
		)
		self.env = self.isaac_gym.create_env(self.sim, ...)
		# 设置相机
		camera_cfg = gymapi.CameraProperties()
		camera_cfg.enable_tensors = False
		self.camera = self.isaac_gym.create_camera_sensor(env, camera_cfg)
		... # 其他设置

		def step(self, action):
			...
			obs = get_observation()

		def get_observation(self):
			self.isaac_gym.fetch_results(self.sim, True)
		    self.isaac_gym.step_graphics(self.sim)
		    self.isaac_gym.render_all_camera_sensors(self.sim)
			# 获取color image
			image = isaac_gym.get_camera_image(self.sim, self.env, self.camera, gymapi.IMAGE_COLOR)
			image = np.reshape(image, (self.img_size[1], self.img_size[0], -1))[..., :-1] # RGBA -> RGB
			image = torch.from_numpy(image).to(self.device)
			# 获取其他观测
			...
```

这样处理之后暂时就没什么问题了，代码也终于能跑起来了。🤧