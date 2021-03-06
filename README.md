# CarND-Controls-MPC
Self-Driving Car Engineer Nanodegree Program

---
## Project Introduction
In this project Model Predictive Control is implemented to drive the car around the track.
However, this time the cross track error was not provided and have to be calculed by programm. Additionally, there's a 100 millisecond latency between actuations commands on top of the connection latency which should be correctly handled by model.

## Solution
The modle predictive control algorithm was used in order to address the problem.
Model predictive controllers rely on dynamic models of the process, most often linear empirical models obtained by system identification. The main advantage of MPC is the fact that it allows the current timeslot to be optimized, while keeping future timeslots in account. This is achieved by optimizing a finite time-horizon, but only implementing the current timeslot and then optimizing again.

The prediction horizon is the duration over which future predictions are made. Prediction horizon is refered as T.
T is the product of two other variables, N and dt.

N, dt, and T are hyperparameters that usually tuned for each model predictive controller. However, there are some general guidelines. T should be as large as possible, while dt should be as small as possible.

### The model
I used global kinematic model described by lesson 18: Vechicle Models.
`[x,y,ψ,v]` is the state of the vehicle, `Lf` is a physical characteristic of the vehicle, and `[δ,a]` are the actuators, or control inputs, to our system.
```
    // The equations for the model:
    // x[t+1] = x[t] + v[t] * cos(psi[t]) * dt
    // y[t+1] = y[t] + v[t] * sin(psi[t]) * dt
    // psi[t+1] = psi[t] + v[t] / Lf * delta[t] * dt
    // v[t+1] = v[t] + a[t] * dt
    // cte[t+1] = f(x[t]) - y[t] + v[t] * sin(epsi[t]) * dt
    // epsi[t+1] = psi[t] - psides[t] + v[t] * delta[t] / Lf * dt
```
Where,
* `x[t]` x coordinate of the vehicle at the moment t
* `y[t]` y coordinate of the vehicle at the moment t
* `psi[t]` vehicle heading at the moment t
* `cte` cross-track error
* `epse` orientation error
* `Lf` measures the distance between the front of the vehicle and its center of gravity. The larger the vehicle, the slower the turn rate.
* `psides` is desired orientation of vehicle which can be found using first order derivatives of fitted polynomial in the point.
* `f(x)` is the ground thruth position of vehicle which is calculated by fitted polynomial.

The goal is to find values for `[δ,a]` which minimize the objective function which is calculated by the following code:
```
    // Any additions to the cost should be added to `fg[0]`.
    fg[0] = 0;

    // Reference State Cost
    for (int i = 0; i < N; ++i) {
      fg[0] += 750 * CppAD::pow(vars[cte_start + i], 2);
      fg[0] += 250 * CppAD::pow(vars[epsi_start + i], 2);
      fg[0] += 0.1 * CppAD::pow(vars[v_start + i] - REF_V, 2);
    }

    for (int i = 0; i < N - 1; ++i) {
      fg[0] += 500 * CppAD::pow(vars[delta_start + i], 2);
      fg[0] += 1 * CppAD::pow(vars[a_start + i], 2);
    }

    for (int i = 0; i < N - 2; ++i) {
      fg[0] += 250000 * CppAD::pow(vars[delta_start + i + 1] - vars[delta_start + i], 2);
      fg[0] += 1 * CppAD::pow(vars[a_start + i + 1] - vars[a_start + i], 2);
    }
```

The model objective function was turned manually based on model performance.
Thus,
* if model diverge from reference trajectory a lot, I tried to increase `cte` and `epse` contribution
* if model used a lot for controls, I tried to increase contribution of `delta`
* if model did a lot of sharp turns, I tried to increase contribution of `difference between consequitive delta`

The mistake which I did at the begining, was trying to optimize objective function first without latency in the model.
Eventually, it results that I needed to turn objective parameters twice, first time for the model without latency used and then one more time for the model with control latency.

Also, it seems that usage of different weights for different speed might improve robustness of algorithm and make it possible drive the vehicle with even higher speed.

### Possible improvements
Try to implement twiddle algorithm in order to optimize weights.

### ψ updates
In order to handle difference in ψ rotation by model (positive rotation is counter-clockwise) and in the simulator (a positive value implies a right turn and a negative value implies a left turn) I multiplied the steering value by -1 before sending it back to the server and when receiving it from the server.

### Timestep Length and Elapsed Duration (N & dt)
In order to chose N and dt I followed the guidelines from lesson:
* in the case of driving a car, T should be a few seconds, at most
* T should be as large as possible, while dt should be as small as possible
So, I tried the following values for `N={7, 8, 10, 12, 15, 20, 40}` for `DT={0.025, 0.05, 0.1, 0.15, 0.2, 0.25}`
The values which worked reasonably well for me are the following:
```
const size_t N = 10;
const double DT = 0.1;
```
I noticed the following:
* if N * dt is large then the model behave bad on sharp turns when speed is large.
* for smaller dt the model gives better accuracy but requires higher N for given horizon. Thus, requires more computations and can increase latency.
* if N * dt is small then the model does not predict correct trajectory

So, trying all different combinations listed above, `N=10, DT=0.1` showed the best tradef-off between accuracy and computational time.

### Polynomial Fitting and MPC Preprocessing
Provided way points were first transformed into the vehicle coordinage system.
```
    Eigen::VectorXd points_x(ptsx.size());
    Eigen::VectorXd points_y(ptsy.size());

    //Display the waypoints/reference line
    vector<double> next_x_vals;
    vector<double> next_y_vals;

    // convert pts to car coordinate
    for (int i = 0; i < ptsx.size(); ++i) {
        auto dx = ptsx[i] - px;
        auto dy = ptsy[i] - py;
        points_x[i] = dx * cos(psi) - dy * sin(psi);
        points_y[i] = dx * sin(psi) + dy * cos(psi);
        next_x_vals.push_back(points_x[i]);
        next_y_vals.push_back(points_y[i]);

    }
```
Then, third order polynomial is fitted using transformed way points.
```
    // fit polyline for pts
    auto coeffs = polyfit(points_x, points_y, 3);
```
Obtained polinomial coefficients then used in order to calculate `cte` and `epsi` as well as used by solver in order find control sequence which best fit to reference vehicle trajectory.
```
    // calculate errors
    double cte = polyeval(coeffs, 0);
    double epsi = -atan(coeffs[1]);
```
and
```
    auto solution = mpc.Solve(state, coeffs);
```
### Model Predictive Control with Latency
#### Handling Latency
As proposed by lesson latency can easily be modeled by a simple dynamic system and incorporated into the vehicle model.
I used approach which runs a simulation using the vehicle model starting from the current state for the duration of the latency. The resulting state from the simulation is the new initial state for MPC.
```
    // take into account delay in control
    px = v * DT;   // v * cos(psi) * dt
    py = 0;        // v * sin(psi) * dt
    psi = v * delta * DT / Lf;
    cte = cte + v * sin(epsi) * DT;
    epsi = epsi + v * delta * DT / Lf;
    v = v + throttle * DT;  // update v last, since it is used for cte and epsi
    // initialize initial state
    Eigen::VectorXd state(6);
    state << px, py, psi, v, cte, epsi;

    auto solution = mpc.Solve(state, coeffs);
```

Thus, MPC can deal with latency much more effectively, by explicitly taking it into account, than a PID controller.

### The vehicle must successfully drive a lap around the track.
Here is [link on the final video with vehicle driving a lap](/video/mpc.avi)

---
## Dependencies

* cmake >= 3.5
 * All OSes: [click here for installation instructions](https://cmake.org/install/)
* make >= 4.1(mac, linux), 3.81(Windows)
  * Linux: make is installed by default on most Linux distros
  * Mac: [install Xcode command line tools to get make](https://developer.apple.com/xcode/features/)
  * Windows: [Click here for installation instructions](http://gnuwin32.sourceforge.net/packages/make.htm)
* gcc/g++ >= 5.4
  * Linux: gcc / g++ is installed by default on most Linux distros
  * Mac: same deal as make - [install Xcode command line tools]((https://developer.apple.com/xcode/features/)
  * Windows: recommend using [MinGW](http://www.mingw.org/)
* [uWebSockets](https://github.com/uWebSockets/uWebSockets)
  * Run either `install-mac.sh` or `install-ubuntu.sh`.
  * If you install from source, checkout to commit `e94b6e1`, i.e.
    ```
    git clone https://github.com/uWebSockets/uWebSockets
    cd uWebSockets
    git checkout e94b6e1
    ```
    Some function signatures have changed in v0.14.x. See [this PR](https://github.com/udacity/CarND-MPC-Project/pull/3) for more details.

* **Ipopt and CppAD:** Please refer to [this document](https://github.com/udacity/CarND-MPC-Project/blob/master/install_Ipopt_CppAD.md) for installation instructions.
* [Eigen](http://eigen.tuxfamily.org/index.php?title=Main_Page). This is already part of the repo so you shouldn't have to worry about it.
* Simulator. You can download these from the [releases tab](https://github.com/udacity/self-driving-car-sim/releases).
* Not a dependency but read the [DATA.md](./DATA.md) for a description of the data sent back from the simulator.


## Basic Build Instructions

1. Clone this repo.
2. Make a build directory: `mkdir build && cd build`
3. Compile: `cmake .. && make`
4. Run it: `./mpc`.

## Tips

1. It's recommended to test the MPC on basic examples to see if your implementation behaves as desired. One possible example
is the vehicle starting offset of a straight line (reference). If the MPC implementation is correct, after some number of timesteps
(not too many) it should find and track the reference line.
2. The `lake_track_waypoints.csv` file has the waypoints of the lake track. You could use this to fit polynomials and points and see of how well your model tracks curve. NOTE: This file might be not completely in sync with the simulator so your solution should NOT depend on it.
3. For visualization this C++ [matplotlib wrapper](https://github.com/lava/matplotlib-cpp) could be helpful.)
4.  Tips for setting up your environment are available [here](https://classroom.udacity.com/nanodegrees/nd013/parts/40f38239-66b6-46ec-ae68-03afd8a601c8/modules/0949fca6-b379-42af-a919-ee50aa304e6a/lessons/f758c44c-5e40-4e01-93b5-1a82aa4e044f/concepts/23d376c7-0195-4276-bdf0-e02f1f3c665d)
5. **VM Latency:** Some students have reported differences in behavior using VM's ostensibly a result of latency.  Please let us know if issues arise as a result of a VM environment.

## Editor Settings

We've purposefully kept editor configuration files out of this repo in order to
keep it as simple and environment agnostic as possible. However, we recommend
using the following settings:

* indent using spaces
* set tab width to 2 spaces (keeps the matrices in source code aligned)

## Code Style

Please (do your best to) stick to [Google's C++ style guide](https://google.github.io/styleguide/cppguide.html).

## Project Instructions and Rubric

Note: regardless of the changes you make, your project must be buildable using
cmake and make!

More information is only accessible by people who are already enrolled in Term 2
of CarND. If you are enrolled, see [the project page](https://classroom.udacity.com/nanodegrees/nd013/parts/40f38239-66b6-46ec-ae68-03afd8a601c8/modules/f1820894-8322-4bb3-81aa-b26b3c6dcbaf/lessons/b1ff3be0-c904-438e-aad3-2b5379f0e0c3/concepts/1a2255a0-e23c-44cf-8d41-39b8a3c8264a)
for instructions and the project rubric.

## Hints!

* You don't have to follow this directory structure, but if you do, your work
  will span all of the .cpp files here. Keep an eye out for TODOs.

## Call for IDE Profiles Pull Requests

Help your fellow students!

We decided to create Makefiles with cmake to keep this project as platform
agnostic as possible. Similarly, we omitted IDE profiles in order to we ensure
that students don't feel pressured to use one IDE or another.

However! I'd love to help people get up and running with their IDEs of choice.
If you've created a profile for an IDE that you think other students would
appreciate, we'd love to have you add the requisite profile files and
instructions to ide_profiles/. For example if you wanted to add a VS Code
profile, you'd add:

* /ide_profiles/vscode/.vscode
* /ide_profiles/vscode/README.md

The README should explain what the profile does, how to take advantage of it,
and how to install it.

Frankly, I've never been involved in a project with multiple IDE profiles
before. I believe the best way to handle this would be to keep them out of the
repo root to avoid clutter. My expectation is that most profiles will include
instructions to copy files to a new location to get picked up by the IDE, but
that's just a guess.

One last note here: regardless of the IDE used, every submitted project must
still be compilable with cmake and make./

## How to write a README
A well written README file can enhance your project and portfolio.  Develop your abilities to create professional README files by completing [this free course](https://www.udacity.com/course/writing-readmes--ud777).
