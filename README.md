# Jz-SLAM
## 1. 环境配置：
### (1). 首先编译第三方库
cd Jz-SLAM/third_parties/

对于 absl 库：
```
cd abseil-cpp/

mkdir build

cd build/

cmake -G Ninja -DCMAKE_BUILD_TYPE=Release -DCMAKE_POSITION_INDEPENDENT_CODE=ON -DCMAKE_INSTALL_PREFIX=/usr/local/stow/absl ..

ninja

sudo ninja install

cd /usr/local/stow

sudo stow absl
```

对于protobuf库：
```
cd protobuf-3.6.0/

sudo apt-get install autoconf automake libtool

./autogen.sh

./configure

sudo make install

sudo ldconfig
```
对于geometry库:
```
cd geometry2-melodic-devel/

catkin_make install
```

其余库：
```
cd xxx/

mkdir build

cd build/

cmake ..

sudo make install
```

### (2). 编译catkin_ws:
```
cd Jz-SLAM/catkin_ws/

catkin_make install
```

注意，imreg_fmt需单独配置如下
```
cd src/imreg_fmt

sudo cp -r include/ /usr/

cd build/

sudo cp libimreg_fmt.a /usr/lib

sudo mkdir -p /usr/share/imreg_fmt/cmake/

cd /usr/share/imreg_fmt/cmake/

sudo gedit imreg_fmtConfig.cmake
```

然后把新建文件.txt里的内容复制进去

### (3). 编译ethzasl_ws:
```
cd Jz-SLAM/ethzasl_ws/

catkin_make install
```

### (4). 编译cartographer_ws:
(一定要sudo apt-get remove libprotobuf-dev，否则会在编译cartographer_ros时会引起protobuf版本冲突)

```
sudo apt-get install ros-melodic-gps-common

cd Jz-SLAM/cartographer_ws/

catkin_make_isolated --install --use-ninja
```

### (5). 编译iplus_perception_ws:
需要sudo apt-get install ros-melodic-sophus

```
cd Jz-SLAM/iplus_perception_ws/

catkin_make install
```
### (6). 编译line_detector_ws:
```
cd Jz-SLAM/line_detector_ws/

catkin_make install
```
### (7). 编译vtr_ws:
需要sudo apt-get install ros-melodic-libg2o
```
cd Jz-SLAM/vtr_ws/

catkin_make -DCATKIN_WHITELIST_PACKAGES="vtr;lidar_localiser"
```
### (8). 编译map_manager_ws:
```
cd Jz-SLAM/map_manager_ws/

catkin_make install
```
### (9). 编译emma_tools_ws:
```
cd Jz-SLAM/emma_tools_ws/

catkin_make install
```


## 2. 建图里程
### （1）LeGO-LOAM前端 + cartographer后端建图，LeGO-LOAM里程：
step1: 先建图

激光信息可在cartograpgher_ws/install_isolated/share/cartographer_ros/configuration_files/lego_parms.yaml改

imu信息在 /opt/jz/caliber/wbias.txt abias.txt下设置，没有噪声noise

可在launch文件里配置imu_topic、激光topic，imu_frame、laser_frame和激光imu的外参（用tf_static发布，注意车的朝向一般为base_footprint x正方向）

map_name建图保存名称 map保存在/home/map下

roslaunch map_manager offline_ui.launch  # 使用ui界面进行mapping

ui界面点击mapping，开始放bag，放完后点击save map，等待地图构建结束

跑完后点IDLE 然后ctrl c关闭会快一点

如果图觉得有点大，可以在offline_ui.launch 文件里改display_ratio

step2：后里程

/map_name  /vtr_path要改  /use_scan3D设为false      外参tf设置

roslaunch map_manager offline_ui.launch  # 使用ui界面进行localization

先是ui界面localization，放bag，等call localiser success时点 open client

如果在线跑定位的时候，需要把/use_sim_time改为false


### （2）LIO-SAM前端 + cartographer后端建图，LIO-SAM里程：
step1: 先建图

roslaunch map_manager lio_sam_mapping.launch

里边可设置imu topic

cartographer_ros/cartographer_ros.launch里设置 laser frame和lidar topic

lio_sam/run.launch里设置外参

然后放bag，同时 rosservice call /cartographer_launcher "status: 0" （可tab出来）

当bag放完时，结束mapping：rosservice call /map_manager_launcher "status: 0" （可tab出来）

rosparam set /map_name xxx

roservice call /shutdown_cartographer （后边tab出来一堆东西）

地图保存在/home/主机名/map/xxx/下

step2：后里程

roslaunch map_manager lio_localization.launch

sudo chmod 777 ./call.sh （记得改地图名）

./call.sh

rosrun map_manager map_client_node

### （3）LIO-SAM前端 + cartographer后端建图，LIO-SAM里程，加入RTK：

step1：采集含GPS消息的bag，其中消息类型为gps_common/GPSFix，topic名称为/jzhw/gps/fix

frame_id为gnss_link，status设为2，dip为与正北方向的夹角（顺时针为正），err_dip赋予较小的值，如0.1

step2：估算rtk到base的外参

一般车前后放两个蘑菇头，前边蘑菇头定位，后边蘑菇头定向，gnss_link坐标系为以定位蘑菇头为原点，定位到定向射线为x轴正方向

由此估算gnss_link到base的外参（z轴不用管）

step3：建图

roslaunch map_manager lio_sam_mapping_rtk.launch

里边可设置tf外参

然后放bag，同时 rosservice call /cartographer_launcher "status: 0" （可tab出来）

rosservice call /lasermark_launcher "status: 0" （可tab出来）

当bag放完时，结束mapping：rosservice call /map_manager_launcher "status: 0" （可tab出来）

rosparam set /map_name xxx

roservice call /shutdown_cartographer （后边tab出来一堆东西）

地图保存在/home/主机名/map/xxx/下，可点击gps_data.xml查看优化出的tf外参

step4：里程

```
roslaunch map_manager lio_localization_rtk.launch

sudo chmod 777 ./call.sh （记得改地图名）

./call.sh

rosrun map_manager map_client_node

rosservice call /mark_localization/set_mark_switch "type: 'gps' flag: 1"  // 可tab出来，gps和lidar协同定位，正常只开这个就能啪地一下定上

rosservice call /mark_localization/set_mark_switch "type: 'gps_manual' flag: 1"  // 可tab出来，强制仅使用gps定位
```
