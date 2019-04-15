# Calibration

Here, we explain how to use the calibration tools with [TUM-VI](https://vision.in.tum.de/data/datasets/visual-inertial-dataset) and [EuRoC](https://projects.asl.ethz.ch/datasets/doku.php?id=kmavvisualinertialdatasets) datasets as an example.


## TUM-VI dataset
Download the datasets for camera and camera-IMU calibration:
```
mkdir ~/tumvi_calib_data
cd ~/tumvi_calib_data
wget http://vision.in.tum.de/tumvi/calibrated/512_16/dataset-calib-cam3_512_16.bag
wget http://vision.in.tum.de/tumvi/calibrated/512_16/dataset-calib-imu1_512_16.bag
```

### Camera calibration
Run the camera calibration:
```
basalt_calibrate --dataset-path ~/tumvi_calib_data/dataset-calib-cam3_512_16.bag --dataset-type bag --result-path ~/tumvi_calib_result/ --cam-types ds ds
```
The command line options have the following meaning:
* `--dataset-path` path to the dataset.
* `--dataset-type` type of the datset. Currently only `bag` and `euroc` formats of the datasets are supported.
* `--result-path` path to the folder where the resulting calibration and intermediate results will be stored.
* `--cam-types` camera models for the image streams in the dataset. For more detais see [arXiv:1807.08957](https://arxiv.org/abs/1807.08957).

After that, you should see the calibration GUI:
![tumvi_cam_calib](doc/img/tumvi_cam_calib.png)

The buttons in the GUI are located in the order which you should follow to calibrate the camera. After pressing a button the system will print the output to the command line:
* `load_dataset` loads the dataset.
* `detect_corners` starts corner detection in the backround thread. Since it is the most time consuming part of the calibration process, the detected corners are cached and loaded if you run the executable again pointing to the same result folder path.
* `init_cam_intr` computes an initial guess for camera intrinsics.
* `init_cam_poses` computes an initial guess for camera poses given the current intrinsics.
* `init_cam_extr` computes an initial transformation between the cameras.
* `init_opt` initializes optimization and shows the projected points given the current calibration and camera poses.
* `optimize` runs an iteration of the optimization and visualizes the result. You should press this button until the error printed in the console output stops decreasing and the optimization converges.
* `save_calib` saves the current calibration as `calibration.json` in the result folder.
* `compute_vign` **(Experimental)** computes a radially-symmetric vignetting for the cameras. For the algorithm to work, **the calibration pattern should be static (and camera moving around it) and have a constant lighting throughout the calibration sequence**. If you run `compute_vign` you should press `save_calib` afterwards. The png images with vignetting will also be stored in the result folder.

You can also control the process using the following buttons:
* `show_frame` slider to switch between the frames in the sequence.
* `show_corners` toggles the visibility of the detected corners shown in red.
* `show_corners_rejected` toggles the visibility of rejected corners.
* `show_init_reproj` shows the initial reprojections computed by the `init_cam_poses` step.
* `show_opt` shows reprojected corners with the current estimate of the intrinsics and poses.
* `show_vign` toggles the visibility of the points used for vignetting estimation. The points are distributed across white areas of the pattern.
* `show_ids` toggles the ID visualization for every point.
* `huber_thresh` controls the threshold for the huber norm in pixels for the optimization.
* `opt_intr` controls if the optimization can change the intrinsics. For some datasets it might be helpful to disable this option for several first iterations of the optimization.

### Camera + IMU + Mocap calibration
After calibrating the camera you can run the camera + IMU + Mocap calibration. The result path should point to the **same folder as before**:
```
basalt_calibrate_imu --dataset-path ~/tumvi_calib_data/dataset-calib-imu1_512_16.bag --dataset-type bag --result-path ~/tumvi_calib_result/ --gyro-noise-std 0.000282 --accel-noise-std 0.016 --gyro-bias-std 0.0001 --accel-bias-std 0.001
```
The command line options for the IMU noise are continous-time and defined as in [Kalibr](https://github.com/ethz-asl/kalibr/wiki/IMU-Noise-Model):
* `--gyro-noise-std` gyroscope white noise.
* `--accel-noise-std` accelerometer white noise.
* `--gyro-bias-std` gyroscope random walk.
* `--accel-bias-std` accelerometer random walk.

![tumvi_imu_calib](doc/img/tumvi_imu_calib.png)

The buttons in the GUI are located in the order which you need to follow to calibrate the camera-IMU setup:
* `load_dataset`, `detect_corners`, `init_cam_poses` same as above.
* `init_cam_imu` initializes the rotation between camera and IMU by aligning rotation velocities of the camera to the gyro data.
* `init_opt` initializes the optimization. Shows reprojected corners in magenta and the estimated values from the spline as solid lines below.
* `optimize` runs an iteration of the optimization. You should press it several times until convergence before proceeding to next steps.
* `init_mocap` initializes the transformation from the Aprilgrid calibration pattern to the Mocap coordinate system. **Currently we assume that the orientation of the Mocap marker frame is approximately aligned with IMU axes**.
* `save_calib` save the current calibration as `calibration.json` in the result folder.
* `save_mocap_calib` save the current Mocap to IMU calibration as `mocap_calibration.json` in the result folder.

You can also control the visualization using the following buttons:
* `show_frame` - `show_ids` the same as above.
* `show_spline` toggles the visibility of enabled measurements (accel, gyro, position, velocity) generated from the spline that we optimize.
* `show_data` toggles the visibility of raw data containted in the dataset.
* `show_accel` shows accelerometer data.
* `show_gyro` shows gyroscope data.
* `show_pos` shows the position data.
* `show_mocap` shows the mocap marker position transformed to the IMU frame.
* `show_mocap_rot_error` shows rotation between the spline and Mocap measurements.
* `show_mocap_rot_vel` shows the rotation velocity computed from the Mocap.

The following options control the optimization process:
* `opt_intr` enables optimization of intrinsics. Usually should be disabled for the camera-IMU calibration.
* `opt_poses` enables optimization based camera pose initialization. Sometimes helps to better initialize the spline before running optimization with `opt_corners`.
* `opt_corners` enables optimization based on reprojection corner positions **(should be used by default)**.
* `opt_cam_time_offset` computes the time offset between camera and the IMU. This option should be used only for refinement when the optimization already converged.
* `opt_imu_scale` enables IMU axis scaling, rotation and mislignment calibration. This option should be used only for refinement when the optimization already converged.
* `opt_mocap` enables Mocap optimization. You should run it only after pressing `init_mocap`.
* `huber_thresh` controls the threshold for the huber norm in pixels for the optimization.


**NOTE:** In this case the we use pre-calibrated sequence, so most of refinements or Mocap to IMU calibration will not have any visible effect. If you want to test this functionality use the "raw" sequences, for example `http://vision.in.tum.de/tumvi/raw/dataset-calib-cam3.bag` and `http://vision.in.tum.de/tumvi/raw/dataset-calib-imu1.bag`. 

## EuRoC dataset
Download the datasets for camera and camera-IMU calibration:
```
mkdir ~/euroc_calib_data
cd ~/euroc_calib_data
wget http://robotics.ethz.ch/~asl-datasets/ijrr_euroc_mav_dataset/calibration_datasets/cam_april/cam_april.bag
wget http://robotics.ethz.ch/~asl-datasets/ijrr_euroc_mav_dataset/calibration_datasets/imu_april/imu_april.bag
```

### Camera calibration
Run the camera calibration:
```
basalt_calibrate --dataset-path ~/euroc_calib_data/cam_april.bag --dataset-type bag --result-path ~/euroc_calib_result/ --cam-types ds ds
```
![euroc_cam_calib](doc/img/euroc_cam_calib.png)

### Camera + IMU calibration
After calibrating the camera you can run the camera + IMU calibration. The result-path should point to the same folder as before:
```
basalt_calibrate_imu --dataset-path ~/euroc_calib_data/imu_april.bag --dataset-type bag --result-path ~/euroc_calib_result/ --gyro-noise-std 0.000282 --accel-noise-std 0.016 --gyro-bias-std 0.0001 --accel-bias-std 0.001
```
![euroc_imu_calib](doc/img/euroc_imu_calib.png)