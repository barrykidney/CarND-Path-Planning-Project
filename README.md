## Project: Path Planning
[![Udacity - Self-Driving Car NanoDegree](https://s3.amazonaws.com/udacity-sdc/github/shield-carnd.svg)](http://www.udacity.com/drive)

## 1. Introduction
The aim of this write up is to describe the process of developing a path planning system for highway driving that will complete at least one lap of the track provided. I will begin by setting the goals of the project, then I will give a walkthrough of the process and the decisions I made. I will then describe the results of the process, followed by a discussion about the limitations and possible improvements. Please note my approach is based on the solution described in the Project Q&A session as I found the lesson material for this assignment particularly difficult to follow and difficult to relate to the assignment problem.


## 2. Goals
The goals / steps of this project are to build a path planner that that satisfies the following requirements:
* Complete at least one lap of the track provided.
* Maintain a reasonable road position and distance to other vehicles.
* Maintain a reasonable speed (close to the speed limit).
* Avoiding excessive jerk on accelerating and turning.
* Must not break the speed limit.
* Must deal with traffic in a safe and effective manner.


## 3. Walkthrough

### 3.1 Move the vehicle in a straight line
The first step is to make the vehicle move in a straight line. This is achieved by generating a vector of waypoints (in cartesian coordinates) at uniform distance intervals directly in front of the vehicle. We use the vehicles yaw value to ensure the points are generated along the vehicle's heading. The code used was provided in the 'Getting started' section of the Highway driving assignment material.

### 3.2 Set the vehicles speed at close to the speed limit
Control of the vehicles speed can now be tuned by adjusting the distance between the generated waypoints. We know the function is called every 0.02 seconds therefore if the waypoints are generated at 0.44 meter intervals the vehicle will travel at 80kph (~50mph). By adding a reference velocity parameter and using it when determing the distance between waypoints we have a more accessible method for adjusting the vehicles velocity.

### 3.3 Limit jerk
The next step is to reduce the jerk when accelerating and braking to below the threshold. We can do this by initially setting the reference velocity to 0, then every 0.02 seconds we check if the reference velocity is below the desired velocity (close to the speed limit). If the reference velocity is found to be below the desired velocity then we increment the velocity by a set amount. The adjustment can be tuned until it is close to the maximum allowable jerk without exceeding it. I set this value to 0.224 as discussed in the Project Q&A section of the assignment learning material.

### 3.4 Generating the waypoints
The most difficult part of this assignment is to get the vehicle to stay in its lane and to follow a smooth trajectory. I utilised the spline library as it does much of the tedious maths efficiently and there would be little point in reproducing it. The general approach I used is to create a spline that originates at the vehicles current location and trace the desired path to the medium distance (90 meters in 30 meter increments). Next we create a new empty path (vector) and copy the remaining waypoints in previous path to the new path. Finally we use the spline created earlier to generate new waypoints to fill out the new path to the desired number of waypoints.

#### 3.4.1 Generate points
To generate a new point in fernet coordinates we can use the vehicles current S coordinate and simply add the desired increment, add 30 if you wish the new point to be 30 meters ahead of the vehicles current position. The D coordinate can be a copy of the vehicles current D coordinate if you wish the new point to be in the same position and in the same lane as the vehicles current position. We can add or subtract to the D value to have the point to the left or right in the lane relative to the vehicles current position. The getXY method can be used to convert the fernet coordinates into cartesian coordinates.

#### 3.4.2 Smooth the trajectory
The spline object exposes a method that will return the Y coordinate of a point on the spline given its X coordinate.
Once we have generated the medium distance spline using the last 2 waypoints in in the previous path (or the vehicles current position if no previous path exists) and 3 waypoints set at 30 meter increments as discussed above, we can obtain X & Y values for any position on that spline. Given that the spline is inherently smooth we now obtain points at intervals corresponding with the current velocity of the vehicle. This will produce a vector of waypoints tracking a smooth path at the required velocity.

#### 3.4.3 Final Path
Generation of the ultimate path that the vehicle will follow can be optimised. If parameters such as the number of point in the path and maximum velocity have been tuned adequately it is be unlikely that the vehicle will complete the entire path in a single timestep. Therefore instead of generating a full path at each timestep we can reuse any points that have not been driven from the previous path. If the number of points in the path is 50 and the vehicle progresses past 3 in a single timestep, in the next timestep we can copy the 47 unused waypoints and generate 3 new waypoints to add to the end of the path, ensuring the path remains a consistent length at each timestep but reducing the amount of processing required.

### 3.5 Slow for vehicle ahead
When the vehicle is approaching a slower vehicle ahead it must be able to adjust its speed accordingly to avoid running into the back of the other vehicle. The sensor fusion module will provide a continuously updated list of all other vehicles in range with their S&D coordinates, velocity, heading and current lane. Each timestep we loop through this list and identify all the vehicle that occupy the same lane as the ego vehicles. Next we use the S value to identify if the vehicle is ahead of the ego vehicle, if so we find the vehicles velocity and its S coordinate. Using these 2 values we can predict with reasonable accuracy where the vehicle will be in a short time period (for example 1 second). We then check if the vehicle will be less than a defined distance (30 meters) from the ego vehicle, if so we decrement the relative velocity of the ego vehicle.

### 3.6 Lane change
To generate a point in a specific lane we can set the D coordinate to the centre of the desired lane. We know that the lanes are 4 meters wide and by assigning a number to each lane (0, 1, 2) we can determine the D value (distance from the origin) of the centre of each lane with the equation: 2+(4*lane), 2 being the offset to the centre of the current lane and 4*lane being the number of full lane widths to offset from the origin.
The distance increment between the S value of the points in the medium distance spline is an important parameter, the distance must be large enough to allow a smooth lane change to take place. If a point in the spline is in lane 1 and the next point is in lane 2, the spline must have space to track a smooth curve from lane 1 to lane 2. The next point in the spline will be an additional 30 meters ahead and should also be located in lane 2, this forces the spline to straighten its trajectory within lane 2. When we now obtain the velocity adjusted waypoints for the path vector they will trace this smooth trajectory between lane 1 and lane 2.

### 3.7 When to overtake
Overtaking presents a surprisingly complicated logical problem. The approach described in the learning material is use of a finite state machine with states including: stay in lane, follow vehicle, prepare for overtake, overtake left and overtake right, conditions determining when the system should transition between these states and cost functions governing these conditions. In a real world application this approach would be more suitable but for the this assignment I found a more simplistic approach adequate. My approach is, when the ego vehicle is following another vehicle and is travelling below the optimal velocity (close to the speed limit) I check if the adjacent lane to the left is clear (I prioritise overtaking using the left lane). If the left lane is clear, I create the medium distance spline with the 30, 60 and 90 meter points located in the desired lane thereby executing a lane change. If the left lane is not clear of other vehicles I check the lane to the right and repeat the process. This approach had a significant flaw, because if multiple vehicles were travelling at a similar velocity side by side the ego vehicle would continuously change lane back and forth. I solved this issue by only executing a lane change if the desired lane is clear beyond the current lane (33 meters vs 30 meters for the current lane), therefore the ego vehicle will only execute a lane change if there is a benefit to doing so.

## 4. Results/Discussion
Overall the implementation works quite effectively, although there is room for significant improvement. The vehicle sometimes cuts in front of other vehicles too aggressively when changing lane, code could be added to assess if a vehicle is approaching quickly from behind when changing lane. It would also be beneficial if the vehicle were to always return to the centre lane when possible as it can get boxed in when occupying an outside lane. Implementation of a finite state machine strategy would be beneficial when determining when to overtake a slower moving vehicle and to actively search for gaps in the traffic to move into.

## 6.0 References:
https://classroom.udacity.com/nanodegrees/nd013/parts/30260907-68c1-4f24-b793-89c0c2a0ad32/modules/b74b8e43-47d1-47d6-a4cf-4d64ea3e0b80/lessons/407a2efa-3383-480f-9266-5981440b09b3/concepts/3bdfeb8c-8dd6-49a7-9d08-beff6703792d






# Original README file from this point







# CarND-Path-Planning-Project
Self-Driving Car Engineer Nanodegree Program
   
### Simulator.
You can download the Term3 Simulator which contains the Path Planning Project from the [releases tab (https://github.com/udacity/self-driving-car-sim/releases/tag/T3_v1.2).  

To run the simulator on Mac/Linux, first make the binary file executable with the following command:
```shell
sudo chmod u+x {simulator_file_name}
```

### Goals
In this project your goal is to safely navigate around a virtual highway with other traffic that is driving +-10 MPH of the 50 MPH speed limit. You will be provided the car's localization and sensor fusion data, there is also a sparse map list of waypoints around the highway. The car should try to go as close as possible to the 50 MPH speed limit, which means passing slower traffic when possible, note that other cars will try to change lanes too. The car should avoid hitting other cars at all cost as well as driving inside of the marked road lanes at all times, unless going from one lane to another. The car should be able to make one complete loop around the 6946m highway. Since the car is trying to go 50 MPH, it should take a little over 5 minutes to complete 1 loop. Also the car should not experience total acceleration over 10 m/s^2 and jerk that is greater than 10 m/s^3.

#### The map of the highway is in data/highway_map.txt
Each waypoint in the list contains  [x,y,s,dx,dy] values. x and y are the waypoint's map coordinate position, the s value is the distance along the road to get to that waypoint in meters, the dx and dy values define the unit normal vector pointing outward of the highway loop.

The highway's waypoints loop around so the frenet s value, distance along the road, goes from 0 to 6945.554.

## Basic Build Instructions

1. Clone this repo.
2. Make a build directory: `mkdir build && cd build`
3. Compile: `cmake .. && make`
4. Run it: `./path_planning`.

Here is the data provided from the Simulator to the C++ Program

#### Main car's localization Data (No Noise)

["x"] The car's x position in map coordinates

["y"] The car's y position in map coordinates

["s"] The car's s position in frenet coordinates

["d"] The car's d position in frenet coordinates

["yaw"] The car's yaw angle in the map

["speed"] The car's speed in MPH

#### Previous path data given to the Planner

//Note: Return the previous list but with processed points removed, can be a nice tool to show how far along
the path has processed since last time. 

["previous_path_x"] The previous list of x points previously given to the simulator

["previous_path_y"] The previous list of y points previously given to the simulator

#### Previous path's end s and d values 

["end_path_s"] The previous list's last point's frenet s value

["end_path_d"] The previous list's last point's frenet d value

#### Sensor Fusion Data, a list of all other car's attributes on the same side of the road. (No Noise)

["sensor_fusion"] A 2d vector of cars and then that car's [car's unique ID, car's x position in map coordinates, car's y position in map coordinates, car's x velocity in m/s, car's y velocity in m/s, car's s position in frenet coordinates, car's d position in frenet coordinates. 

## Details

1. The car uses a perfect controller and will visit every (x,y) point it recieves in the list every .02 seconds. The units for the (x,y) points are in meters and the spacing of the points determines the speed of the car. The vector going from a point to the next point in the list dictates the angle of the car. Acceleration both in the tangential and normal directions is measured along with the jerk, the rate of change of total Acceleration. The (x,y) point paths that the planner recieves should not have a total acceleration that goes over 10 m/s^2, also the jerk should not go over 50 m/s^3. (NOTE: As this is BETA, these requirements might change. Also currently jerk is over a .02 second interval, it would probably be better to average total acceleration over 1 second and measure jerk from that.

2. There will be some latency between the simulator running and the path planner returning a path, with optimized code usually its not very long maybe just 1-3 time steps. During this delay the simulator will continue using points that it was last given, because of this its a good idea to store the last points you have used so you can have a smooth transition. previous_path_x, and previous_path_y can be helpful for this transition since they show the last points given to the simulator controller with the processed points already removed. You would either return a path that extends this previous path or make sure to create a new path that has a smooth transition with this last path.

## Tips

A really helpful resource for doing this project and creating smooth trajectories was using http://kluge.in-chemnitz.de/opensource/spline/, the spline function is in a single hearder file is really easy to use.

---

## Dependencies

* cmake >= 3.5
  * All OSes: [click here for installation instructions](https://cmake.org/install/)
* make >= 4.1
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


## Call for IDE Profiles Pull Requests

Help your fellow students!

We decided to create Makefiles with cmake to keep this project as platform
agnostic as possible. Similarly, we omitted IDE profiles in order to ensure
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

