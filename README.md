# Project: Extended Kalman Filter 
## Overview  

This project is about utilizing a [Kalman Filter](https://en.wikipedia.org/wiki/Kalman_filter) to estimate the state (position & velocity) of a moving object of interest with noisy LiDAR and Radar measurements. 

## How Does It Work?

This project involves the use of a simulator which can be downloaded from [here](https://github.com/udacity/self-driving-car-sim/releases). The simulator provides LiDAR and Radar measurements that are utilized by the Extended Kalman Filter(EKF) to provide estimated position & velocity of the object of interest.
The EKF works in two steps; *predict* and *update*. In the *predict* step, based on the time difference alone (between previous & current timestamps), a prediction is made, whereas in the *update* step, the belief in object's location is updated based on the received sensor measurements.

**INPUT**: values provided by the simulator to the c++ program

`sensor_measurement` => the measurement that the simulator observed (either LiDAR or Radar)


**OUTPUT**: values provided by the c++ program to the simulator

`estimate_x` <= kalman filter estimated position x

`estimate_y` <= kalman filter estimated position y

`rmse_x` <= root mean square error of position x

`rmse_y` <= root mean square error of position y

`rmse_vx` <= root mean square error of velocity x

`rmse_vy` <= root mean square error of velocity y


## Rubric Points

### Compilation
#### Your code should compile.

The code compiles using `make` & `cmake` as indicated by the successful creation of the `./ExtendedKF` directory.

### Accuracy
#### px, py, vx, vy output coordinates must have an RMSE <= [.11, .11, 0.52, 0.52] when using the file: "obj_pose-laser-radar-synthetic-input.txt which is the same data file the simulator uses for Dataset 1".

The RMSE of the algorithm is `[.0964, .0853, 0.4154, 0.4316]` as shown in the figure below:

![RMSE](/images/simulator-1.jpg)

### Follows the Correct Algorithm
#### Your Sensor Fusion algorithm follows the general processing flow as taught in the preceding lessons.

The algorithm follows the general processing flow:

1. Initialize state & covariance matrices on first sensor measurement.
2. On subsequent measurements, run the same *predict* function.
3. After *predict* function, call separate functions for *update* depending upon the type of the sensor.

#### Your Kalman Filter algorithm handles the first measurements appropriately. 

The first measurement is handled correctly as follows:

```c++
if (!is_initialized_) {
    /**
      * Initialize the state ekf_.x_ with the first measurement.
      * Create the covariance matrix.
      * Remember: you'll need to convert radar from polar to cartesian coordinates.
    */
    // first measurement
    cout << "EKF: " << endl;
    ekf_.x_ = VectorXd(4);
    ekf_.x_ << 1, 1, 1, 1;
    
    if (measurement_pack.sensor_type_ == MeasurementPackage::RADAR) {
      /**
      Convert radar from polar to cartesian coordinates and initialize state.
      */
      float rho = measurement_pack.raw_measurements_(0);
      float theta = measurement_pack.raw_measurements_(1);
      float rho_dot = measurement_pack.raw_measurements_(2);
      ekf_.x_(0) = rho * cos(theta);
      if (ekf_.x_(0) < 0.0001) ekf_.x_(0) = 0;
      ekf_.x_(1) = rho * sin(theta);
      if (ekf_.x_(1) < 0.0001) ekf_.x_(1) = 0;
      ekf_.x_(2) = rho_dot * cos(theta);
      ekf_.x_(3) = rho_dot * sin(theta);
    }
    else if (measurement_pack.sensor_type_ == MeasurementPackage::LASER) {
      ekf_.x_(0) = measurement_pack.raw_measurements_(0);
      ekf_.x_(1) = measurement_pack.raw_measurements_(1);
    }
    
    previous_timestamp_ = measurement_pack.timestamp_;
    // done initializing, no need to predict or update
    is_initialized_ = true;
    return;
  }
```

#### Your Kalman Filter algorithm first predicts then updates.

The Kalman Filter algorithm first predicts  then updates as shown below:

```c++
/*****************************************************************************
   *  Prediction
   ****************************************************************************/

  /**
     * Update the state transition matrix F according to the new elapsed time.
      - Time is measured in seconds.
     * Update the process noise covariance matrix.
     * Use noise_ax = 9 and noise_ay = 9 for your Q matrix.
   */
  //compute the time elapsed between the current and previous measurements
  float dt = (measurement_pack.timestamp_ - previous_timestamp_) / 1000000.0;	//dt - expressed in seconds
  previous_timestamp_ = measurement_pack.timestamp_;
  
  // Update the state transition matrix F according to the new elapsed time.
  ekf_.F_(0,2) = dt;
	ekf_.F_(1,3) = dt;

  // Update the process noise covariance matrix
  ekf_.Q_ = MatrixXd(4, 4);
	/*ekf_.Q_ << ((pow(dt,4)/4) * noise_ax_), 0, ((pow(dt,3)/2) * noise_ax_), 0,
              0, ((pow(dt,4)/4) * noise_ay_), 0, ((pow(dt,3)/2) * noise_ay_),
              ((pow(dt,3)/2) * noise_ax_), 0, (pow(dt,2) * noise_ax_), 0,
              0, ((pow(dt,3)/2) * noise_ay_), 0, (pow(dt,2) * noise_ay_);*/
  float dt_2 = pow(dt,2);
  float dt_3 = (pow(dt,3)/2);
  float dt_3_x = dt_3 * noise_ax_;
  float dt_3_y = dt_3 * noise_ay_;
  float dt_4 = pow(dt,4)/4;
  
  ekf_.Q_ << (dt_4 * noise_ax_), 0, dt_3_x, 0,
              0, (dt_4 * noise_ay_), 0, dt_3_y,
              dt_3_x, 0, (dt_2 * noise_ax_), 0,
              0, dt_3_y, 0, (dt_2 * noise_ay_);
              
  ekf_.Predict();

  /*****************************************************************************
   *  Update
   ****************************************************************************/

  /**
     * Use the sensor type to perform the update step.
     * Update the state and covariance matrices.
   */

  if (measurement_pack.sensor_type_ == MeasurementPackage::RADAR) {
    // Radar updates
    Tools tools;
    Hj_ = tools.CalculateJacobian(ekf_.x_);
    ekf_.H_ = Hj_;
    ekf_.R_ = R_radar_;    
    ekf_.UpdateEKF(measurement_pack.raw_measurements_);
  } else {
    // laser updates
    ekf_.H_ = H_laser_;
    ekf_.R_ = R_laser_;
    ekf_.Update(measurement_pack.raw_measurements_);
  }
```

#### Your Kalman Filter can handle Radar and LiDAR measurements.

The algorithm handles these types separately by executing different code based on the type of the sensor e.g.:

```c++
 if (measurement_pack.sensor_type_ == MeasurementPackage::RADAR)
``` 

```c++
 else if (measurement_pack.sensor_type_ == MeasurementPackage::LASER)
```

### Code Efficiency
#### Your algorithm should avoid unnecessary calculations.

The algorithm follows the following guidelines:

* Running the exact same calculation repeatedly when you can run it once, store the value and then reuse the value later.
* Loops that run too many times.
* Creating unnecessarily complex data structures when simpler structures work equivalently.
* Unnecessary control flow checks.

For example, consider the following code fragment:

```c++
  float dt_2 = pow(dt,2);
  float dt_3 = (pow(dt,3)/2);
  float dt_3_x = dt_3 * noise_ax_;
  float dt_3_y = dt_3 * noise_ay_;
  float dt_4 = pow(dt,4)/4;
  
  ekf_.Q_ << (dt_4 * noise_ax_), 0, dt_3_x, 0,
              0, (dt_4 * noise_ay_), 0, dt_3_y,
              dt_3_x, 0, (dt_2 * noise_ax_), 0,
              0, dt_3_y, 0, (dt_2 * noise_ay_);
```

Here, the *process covariance matrix* `Q_` is being updated by pre-computing certain values that are used more than once within the matrix.

## Directory Structure

* **data:** Directory containing sample sensor measurements
* **images:** Some screenshots
* **src:** Directory containing c++ source files along with Eigen library
* **CMakeLists.txt:** File containing compilation instructions
* **README.md:** Project readme file
* **Visualization.ipynb:** Python notebook for output data visualization 
* **estimates.txt:** Tab-delimited file containing output data
* **install-mac.sh:** Script for installing uWebSockets on Macintosh
* **install-ubuntu.sh:** Script for installing uWebSockets on Ubuntu


## Requirements

* cmake >= 3.5
  * All OSes: [click here for installation instructions](https://cmake.org/install/)
* make >= 4.1 (Linux, Mac), 3.81 (Windows)
  * Linux: make is installed by default on most Linux distros
  * Mac: [install Xcode command line tools to get make](https://developer.apple.com/xcode/features/)
  * Windows: [Click here for installation instructions](http://gnuwin32.sourceforge.net/packages/make.htm)
* gcc/g++ >= 5.4
  * Linux: gcc / g++ is installed by default on most Linux distros
  * Mac: same deal as make - [install Xcode command line tools](https://developer.apple.com/xcode/features/)
  * Windows: recommend using [MinGW](http://www.mingw.org/)

## Basic Build Instructions

This repository includes two files that can be used to set up and install [uWebSocketIO](https://github.com/uWebSockets/uWebSockets) for either Linux or Mac systems. 

1. Clone this repo.
2. Make a build directory: `mkdir build && cd build`
3. Compile: `cmake .. && make` 
   * On windows, you may need to run: `cmake .. -G "Unix Makefiles" && make`
4. Run it: `./ExtendedKF `


## Usage

### With Simulator

Follow the build instructions above. Once the program is running, start the simulator. You should see a *connected!!!* message upon successful connection between the simulator and the c++ program. Hit the *Start button*. The output should be same as shown [here](https://github.com/wkhattak/Extended-Kalman-Filters#accuracy).

### Without Simulator

Instead of installing the [uWebSocketIO](https://github.com/uWebSockets/uWebSockets) library and running the simulator, following approach can be adopted:

1. Remove the [main.cpp](https://github.com/wkhattak/Extended-Kalman-Filters/blob/master/src/main.cpp) file.
2. Rename the [main_without_uwebsocketio.cpp](https://github.com/wkhattak/Extended-Kalman-Filters/blob/master/src/main_without_uwebsocketio.cpp) to `main.cpp`.
3. Copy the file `data\obj_pose-laser-radar-synthetic-input.txt` to the parent directory where the code is running.
4. Compile/run the project.
5. Upon successful execution, you should see a file `estimates.txt` created in the directory where the program is running. The file contains the data in this format:

`px_est py_est vx_est vy_est px_meas py_meas px_gt 
py_gt vx_gt vy_gt px_rmse py_rmse vx_rmse vy_rmse`


## Output Data Visualizations

Estimated position of the object is shown as green triangles:

![Simulator Zoomed In](/images/simulator-2.jpg)


Visualization generated using the `estimates.txt` file:

![Output Data Visualization](/images/data-visualization.png)


## License

The content of this project is licensed under the [Creative Commons Attribution 3.0 license](https://creativecommons.org/licenses/by/3.0/us/deed.en_US).