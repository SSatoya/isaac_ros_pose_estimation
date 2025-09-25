# Isaac ROS FoundationPose with Isaac Sim

## 参考
https://nvidia-isaac-ros.github.io/repositories_and_packages/isaac_ros_pose_estimation/isaac_ros_foundationpose/index.html


## 環境構築

isaac commonは用意されている前提


## モデルのダウンロード

1. （コンテナ内に入る（ローカルでも可））
    
    ```bash
    cd ${ISAAC_ROS_WS}/src/isaac_ros_common && \
    ./scripts/run_dev.sh
    ```
    
2. 必要なライブラリがインストールされていることを確認
    
    ```bash
    sudo apt-get install -y curl jq tar
    ```
    
3. NGC からアセットをダウンロード
    
    ```bash
    NGC_ORG="nvidia"
    NGC_TEAM="isaac"
    PACKAGE_NAME="isaac_ros_foundationpose"
    NGC_RESOURCE="isaac_ros_foundationpose_assets"
    NGC_FILENAME="quickstart.tar.gz"
    MAJOR_VERSION=3
    MINOR_VERSION=2
    VERSION_REQ_URL="https://catalog.ngc.nvidia.com/api/resources/versions?orgName=$NGC_ORG&teamName=$NGC_TEAM&name=$NGC_RESOURCE&isPublic=true&pageNumber=0&pageSize=100&sortOrder=CREATED_DATE_DESC"
    AVAILABLE_VERSIONS=$(curl -s \
        -H "Accept: application/json" "$VERSION_REQ_URL")
    LATEST_VERSION_ID=$(echo $AVAILABLE_VERSIONS | jq -r "
        .recipeVersions[]
        | .versionId as \$v
        | \$v | select(test(\"^\\\\d+\\\\.\\\\d+\\\\.\\\\d+$\"))
        | split(\".\") | {major: .[0]|tonumber, minor: .[1]|tonumber, patch: .[2]|tonumber}
        | select(.major == $MAJOR_VERSION and .minor <= $MINOR_VERSION)
        | \$v
        " | sort -V | tail -n 1
    )
    if [ -z "$LATEST_VERSION_ID" ]; then
        echo "No corresponding version found for Isaac ROS $MAJOR_VERSION.$MINOR_VERSION"
        echo "Found versions:"
        echo $AVAILABLE_VERSIONS | jq -r '.recipeVersions[].versionId'
    else
        mkdir -p ${ISAAC_ROS_WS}/isaac_ros_assets && \
        FILE_REQ_URL="https://api.ngc.nvidia.com/v2/resources/$NGC_ORG/$NGC_TEAM/$NGC_RESOURCE/\
    versions/$LATEST_VERSION_ID/files/$NGC_FILENAME" && \
        curl -LO --request GET "${FILE_REQ_URL}" && \
        tar -xf ${NGC_FILENAME} -C ${ISAAC_ROS_WS}/isaac_ros_assets && \
        rm ${NGC_FILENAME}
    fi
    ```
    
4. NGC から事前トレーニング済みの[FoundationPose](https://catalog.ngc.nvidia.com/orgs/nvidia/teams/isaac/models/foundationpose)モデルを必要なディレクトリにダウンロード
    
    ```bash
    mkdir -p ${ISAAC_ROS_WS}/isaac_ros_assets/models/foundationpose && \
       cd ${ISAAC_ROS_WS}/isaac_ros_assets/models/foundationpose && \
       wget 'https://api.ngc.nvidia.com/v2/models/nvidia/isaac/foundationpose/versions/1.0.0_onnx/files/refine_model.onnx' -O refine_model.onnx && \
       wget 'https://api.ngc.nvidia.com/v2/models/nvidia/isaac/foundationpose/versions/1.0.0_onnx/files/score_model.onnx' -O score_model.onnx
    ```
    
5. Dokcerないの場合は、一度抜ける
（exitではなく`ctrl+P`, `ctrl+Q`でコンテナを止めることなく抜けることができる）

---

## ビルド

1. リポジトリをクローン
    
    ```bash
    cd ${ISAAC_ROS_WS}/src && \
       git clone -b release-3.2 https://github.com/NVIDIA-ISAAC-ROS/isaac_ros_pose_estimation.git isaac_ros_pose_estimation
    ```
    
2. Docker コンテナを起動
    
    ```bash
    cd ${ISAAC_ROS_WS}/src/isaac_ros_common && \
       ./scripts/run_dev.sh
    ```
    
3. 依存関係をインストール
    
    ```bash
    sudo apt-get update
    ```
    
    ```bash
    rosdep update && rosdep install --from-paths ${ISAAC_ROS_WS}/src/isaac_ros_pose_estimation/isaac_ros_foundationpose --ignore-src -y
    ```
    
4. ソースからパッケージをビルド
    
    ```bash
    cd ${ISAAC_ROS_WS}/ && \
       colcon build --symlink-install --packages-up-to isaac_ros_foundationpose --base-paths ${ISAAC_ROS_WS}/src/isaac_ros_pose_estimation/isaac_ros_foundationpose
    ```
    
5. ROS ワークスペースをソース
    
    ```bash
    source install/setup.bash
    ```

## 起動ファイルの実行

1. コンテナ内で、モデルを`.onnx`TensorRT エンジン プランに変換
    
    ```bash
    /usr/src/tensorrt/bin/trtexec --onnx=${ISAAC_ROS_WS}/isaac_ros_assets/models/foundationpose/refine_model.onnx --saveEngine=${ISAAC_ROS_WS}/isaac_ros_assets/models/foundationpose/refine_trt_engine.plan --minShapes=input1:1x160x160x6,input2:1x160x160x6 --optShapes=input1:1x160x160x6,input2:1x160x160x6 --maxShapes=input1:42x160x160x6,input2:42x160x160x6
    ```
    
    ```bash
    /usr/src/tensorrt/bin/trtexec --onnx=${ISAAC_ROS_WS}/isaac_ros_assets/models/foundationpose/score_model.onnx --saveEngine=${ISAAC_ROS_WS}/isaac_ros_assets/models/foundationpose/score_trt_engine.plan --minShapes=input1:1x160x160x6,input2:1x160x160x6 --optShapes=input1:1x160x160x6,input2:1x160x160x6 --maxShapes=input1:252x160x160x6,input2:252x160x160x6
    ```
    
## 実行コマンド

```bash
ros2 launch isaac_ros_foundationpose isaac_ros_foundationpose_melon_sim.launch.py  \
	refine_engine_file_path:=${ISAAC_ROS_WS}/isaac_ros_assets/models/foundationpose/refine_trt_engine.plan \
	score_engine_file_path:=${ISAAC_ROS_WS}/isaac_ros_assets/models/foundationpose/score_trt_engine.plan \
	mesh_file_path:=${ISAAC_ROS_WS}/isaac_ros_assets/isaac_ros_foundationpose/Mustard/textured_simple.obj \
	texture_path:=${ISAAC_ROS_WS}/isaac_ros_assets/isaac_ros_foundationpose/Mustard/texture_map.png
```

新しくターミナルを開いて

```bash
rviz2
```
- Fixed Frame:sim_camera
- Detection3DArray
  - topic:/rgb/output
- camera
  - topic:/rgb/image_rect_color