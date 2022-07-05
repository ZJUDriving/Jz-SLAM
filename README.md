# Jz-SLAM

## 建图里程
### （1）LeGO-LOAM前端 + cartographer后端建图，LeGO-LOAM里程：
step1: 先建图

激光信息可在cartograpgher_ws/install_isolated/share/cartographer_ros/configuration_files/lego_parms.yaml改

imu信息在 /opt/jz/caliber/wbias.txt abias.txt下设置，没有噪声noise

可在launch文件里配置imu_topic、激光topic，imu_frame、laser_frame和激光imu的外参（用tf_static发布，注意车的朝向一般为base_footprint x正方向）

map_name建图保存名称 map保存在/home/map下
```
roslaunch map_manager offline_ui.launch  # 使用ui界面进行mapping
```
ui界面点击mapping，开始放bag，放完后点击save map，等待地图构建结束

跑完后点IDLE 然后ctrl c关闭会快一点

如果图觉得有点大，可以在offline_ui.launch 文件里改display_ratio

step2：后里程

/map_name  /vtr_path要改  /use_scan3D设为false      外参tf设置
```
roslaunch map_manager offline_ui.launch  # 使用ui界面进行localization
```
先是ui界面localization，放bag，等call localiser success时点 open client

如果在线跑定位的时候，需要把/use_sim_time改为false

