---
layout: post
title: "ROS2でカメラ映像を取得する"
date: 2026-11-09
---

# 目的
物体認識結果(世界座標)をカメラ映像に重畳表示する

---
# 概要
* 1.USBカメラの接続

* 2.USBカメラから映像を取得してROSに流す  
　camera_img_publisher.py 

* 3.カメラのキャリブレーション

* 4.物体認識結果をROSに流す  
　object_detect_publisher.py

* 5.物体認識結果を画面座標に変換し、カメラ映像に重畳表示してROSに流す  
　camera_to_screen.py

* 6.rVizで表示する  

---
# 環境
* Windows11(WSL) - Ubuntu24.04  
* Python 3.12.3  
* カメラ：ロジクール ウェブカメラ HD 720P C270nd

---
# 構成図
<img
  src="{{ '/assets/images/CameraToScreen/soft_block.png' | relative_url }}"
  alt="win_カメラチェック"
  style="max-width: 40%; height: auto;"
    >

---
# 1.USBカメラの接続
PC(Windows)に接続したUSBカメラをWSL2(Ubuntu)で認識できるようにする

1.Windowsの認識状態を確認  
<img
  src="{{ '/assets/images/CameraToScreen/1.1.camera_check.png' | relative_url }}"
  alt="win_カメラチェック"
  style="max-width: 40%; height: auto;"
    >

2.windowsにusbipdのインストールする  
※windowsのカメラをWSL2に共有するためのソフト

[usbipd-win](https://github.com/dorssel/usbipd-win?utm_source=chatgpt.com)

インストール結果  
<img
  src="{{ '/assets/images/CameraToScreen/1.2.usbpid_version.png' | relative_url }}"
  alt="win_usbpid_version"
  style="max-width: 60%; height: auto;"
    >

usbpid list でカメラ認識を確認  
<img
  src="{{ '/assets/images/CameraToScreen/1.2.usbpid_list.png' | relative_url }}"
  alt="win_usbpid_version"
  style="max-width: 40%; height: auto;"
    >

3.USBデバイスを共有可能にする

PowerShellを管理者権限で起動し、「usbipd bind --busid 6-1」を実行  
ステータスが「Not Shared」から「Shared」に変わる  
<img
  src="{{ '/assets/images/CameraToScreen/1.3.usbipd_list.png' | relative_url }}"
  alt="win_usbpid_version"
  style="max-width: 40%; height: auto;"
    >

4.WSL2にアタッチ

「usbipd attach --wsl --busid 6-1」を実行  
<img
  src="{{ '/assets/images/CameraToScreen/1.4.usbipd_attach.png' | relative_url }}"
  alt="win_usbpid_version"
  style="max-width: 60%; height: auto;"
    >

ステータスが「Shared」から「Attached」に変わる  
<img
  src="{{ '/assets/images/CameraToScreen/1.4.usbipd_list.png' | relative_url }}"
  alt="win_usbpid_version"
  style="max-width: 40%; height: auto;"
    >

5.Ubuntu側で確認

<img
  src="{{ '/assets/images/CameraToScreen/1.5.lsusb.png' | relative_url }}"
  alt="win_usbpid_version"
  style="max-width: 40%; height: auto;"
    >

---
# 2.USBカメラから映像を取得してROSに流す  

cv2を使ってカメライメージを取り出すpythonコード

```python
  camera = cv2.VideoCapture("/dev/video0", cv2.CAP_V4L2)
  camera.set(cv2.CAP_PROP_FOURCC, cv2.VideoWriter_fourcc(*"MJPG"))
  camera.set(cv2.CAP_PROP_FRAME_WIDTH, 640)
  camera.set(cv2.CAP_PROP_FRAME_HEIGHT, 480)
  camera.set(cv2.CAP_PROP_FPS, 30)
  ret, frame = camera.read()
  cv2.imshow("Logitech C270 - MJPG", frame)
```
※デバイス(/dev/video0, /dev/video1)はいずれか繋がる方を選択。

ROS送信には「sensor_msgs/Image」を使用、適当なトピック名で送信します。

```python
  message = Image()
  message.header.stamp          = self.get_clock().now().to_msg()
  message.header.frame_id       = "usb_camera"
  message.height, message.width = frame.shape[:2]
  message.encoding              = "bgr8"
  message.is_bigendian          = 0
  message.step                  = message.width * frame.shape[2]
  message.data                  = frame.tobytes()
  self.publisher.publish(message)
```

カメラ画像を取り込んでROS送信するソフトを作成  
[camera_img_publisher.py を参照](https://github.com/watanabe-shoji/ROS2/blob/main/camera_to_screen_ws/src/camera_to_screen_pkg/camera_to_screen_pkg/camera_img_publisher.py)


---
# 3.カメラのキャリブレーション
物体認識結果(カメラから物体までの距離)を画面座標に変換するため、カメラのキャリブレーションを行う。  
ROSからカメライメージをSubscribeする必要があるため、手順2でROS送信が出来ている必要がある。

[参考]
[Camera Calibration](https://docs.nav2.org/jazzy/tutorials/general_tutorials/camera_calibration/camera_calibration/)


1.チェッカーボードを用意

rows:10, col:8, Checker Width:20[mm]の設定で、でチェッカーボードを印刷しておく

2.キャリブレーションツールのインストール
> sudo apt install ros-jazzy-camera-calibration

3.キャリブレーション実行

[ターミナル1] 手順2のアプリを起動(カメラ画像をROS送信する)  
[ターミナル2] camera_calibrationツールを実行  
> ros2 run camera_calibration cameracalibrator--no-service-check --size 7x9 --square 0.02   --ros-args -r image:=/CameraImages -p camera:=c270  

```
  --size 7x9              :チェッカーボードがcol:8, rows:10に設定したので、交点の数は7x9
  --square 0.02           :checker width[mm]に合わせる
  -r image:=/CameraImages :取得するトピック名
```
以下のような画面が表示されるので、[参考]ページに従ってキャリブレーションをおkの合う。

<img
  src="{{ '/assets/images/CameraToScreen/3.3.camera_calibration.png' | relative_url }}"
  alt="win_usbpid_version"
  style="max-width: 40%; height: auto;"
    >

「/tmp/calibrationdata.tar.gz」ができるので、解凍する。

<img
  src="{{ '/assets/images/CameraToScreen/3.3.calib_ost_txt.png' | relative_url }}"
  alt="win_usbpid_version"
  style="max-width: 20%; height: auto;"
    >
<img
  src="{{ '/assets/images/CameraToScreen/3.3.calib_ost_yaml.png' | relative_url }}"
  alt="win_usbpid_version"
  style="max-width: 20%; height: auto;"
    >

---
# 4.物体認識結果をROSに流す
[前提]

LiDAR等を用いた物体測距を想定するが、それは用意できないので、
ここではカメラから物体までの距離をざっくり測定した結果を使用する。  
また、カメラやLiDARを使う場合、起点となる座標を決めて、そこからの距離となるため、
後段の画面座標を算出する場合は、「起点からの距離」を「カメラからの距離」に座標変換する必要があるが、ここでは「カメラからの距離」をROSに流すこととする。  
また、性格な距離を測ることが困難なため、後段で調整ができるよう、csvから読み込んだ物体情報をROSに流す方式とする。  

1.読み込むcsvファイルの作成(物体認識結果)

* 物体サイズ(width/height)は15[cm] × 9[cm]、  
* 物体までの距離はおおよそ、奥行き(z):140[cm], 縦(y):5[cm], 横(x):13[cm]  
※カメラの光軸からの距離を正確に測ることは困難なため、  
　後からROIを表示しながら調整した結果。

<img
  src="{{ '/assets/images/CameraToScreen/4.1.object_detect_csv.png' |
   relative_url }}"
  alt="win_usbpid_version"
  style="max-width: 20%; height: auto;"
    >

2.ROS送信するカスタムメッセージを作成

<img
  src="{{ '/assets/images/CameraToScreen/4.2.custom_msg.png' |
   relative_url }}"
  alt="win_usbpid_version"
  style="max-width: 20%; height: auto;"
    >


3.csvを読み込んで、ROSPublishするソフトを作成

[object_detect_publisher.py を参照](https://github.com/watanabe-shoji/ROS2/blob/main/camera_to_screen_ws/src/camera_to_screen_pkg/camera_to_screen_pkg/object_detect_publisher.py)

---
# 5.物体認識結果を画面座標に変換し、カメラ映像に重畳表示してROSに流す
キャリブレーション結果を使い、物体のカメラ座標[m]を、
カメラ画像内のピクセル座標に変換する。
元のカメラ画像に合わせるため、ROIの投影時にレンズ歪みを加える。

- image_width             ：校正時の画像の横幅 [pixel]
- image_height            ：校正時の画像の縦幅 [pixel]
- camera_matrix           ：焦点距離・主点位置を持つカメラ内部行列
- distortion_coefficients ：レンズ歪み係数

[流れ]

1. 起動時にカメラキャリブレーションファイルを読み込む。
2. ROSで物体情報を受信し、最新の一覧を保持する。
3. ROSでカメラ画像を受信するたび、保持している物体の位置と実寸から
   ROIの輪郭を作り、cv2.projectPoints()でレンズ歪みを含む画像座標へ投影する。
4. 元のカメラ画像にROIとIDを描画し、/CameraImages/roiへROS配信する。

[camera_to_screen.py を参照](https://github.com/watanabe-shoji/ROS2/blob/main/camera_to_screen_ws/src/camera_to_screen_pkg/camera_to_screen_pkg/camera_to_screen.py)

---
# 6.rVizで表示する
1.rviz2を実行
>rviz2

2.[Displays]の[Add]から[images]を選択

<img
  src="{{ '/assets/images/CameraToScreen/6.2.rviz2_setteing.png' |
   relative_url }}"
  alt="win_usbpid_version"
  style="max-width: 40%; height: auto;"
    >

3.[Displays]に[Images]が追加されるので、トピック名を設定  
  [Reliability Policy]を[Best Effort]に変更

カメラ映像が表示されることを確認

<img
  src="{{ '/assets/images/CameraToScreen/6.3.Image_viewer.png' |
   relative_url }}"
  alt="win_usbpid_version"
  style="max-width: 10%; height: auto;"
    >

後は余計なエリアを削除してconfigを保存しておく。[File]-[Save Config As]


---
# 起動スクリプト

[ターミナル1]
start_camera_img_pub.sh
```
source /opt/ros/jazzy/setup.bash 
source ../install/setup.bash
ros2 run camera_to_screen_pkg camera_img_publisher
```

[ターミナル2]
start_object_detect.sh
```
source /opt/ros/jazzy/setup.bash 
source ../install/setup.bash
ros2 run camera_to_screen_pkg object_detect_publisher
```

[ターミナル3]
start_camera_to_screen.sh
```
source /opt/ros/jazzy/setup.bash 
source ../install/setup.bash
ros2 run camera_to_screen_pkg camera_to_screen
```
[ターミナル4]
start_rviz.sh
```
source /opt/ros/jazzy/setup.bash 
source ../install/setup.bash
rviz2 -d ../src/camera_to_screen_pkg/config/rViz_cfg.rviz
```

実行結果

<img
  src="{{ '/assets/images/CameraToScreen/rviz_result.png' |
   relative_url }}"
  alt="win_usbpid_version"
  style="max-width: 40%; height: auto;"
    >

※前述の通り、手動で物体計測をしているためぴったりROIが重なることはない。  
　本件はカメラ座標->画面座標への変換を確認する目的なので、  
　ここでROIが重なるように、config/object_detect.csvを編集しながら測距値を調整している。
　
　