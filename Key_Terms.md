# Key Terms
This document lists some important key terms to know in regards to robot programming.

## Subsystems
Low-level code for the individual systems that make up the robot such as: launchers, intakes, elevators, etc.
This adds substance to these parts of the system so they can be controlled in sync later

## Commands
Declares what states the robot may encounter and how it must function accordingly. Commands combine multiple subsystems into the necessary action for the robot. For example, if you wanted to intake from the feeder station you would need to first utilize the elevator subsystem to raise the manipulator and then the pivot subsystem to line it up and the intake subsystem to actually take in the piece. 

## PID
Helps motors to get from point A to B in a stable manner, each part of the Proportional Integral Derivative works through part of this process to make movement smooth, and values inserted will be adjusted.

## RobotPy
The toolkit the team uses to code the robot.

## State Machines
Goes through a process of checking each given variable in the machine through "if", "elif", and "else" statements. The code checks each variable in descending order. If the variable given in one of these cases matches, the code then works through the process given within it.

## Feed Forward
An equation that can figure out how much power is need for a motor/system to move/accomplish its task. For example, the elevator needs to move to the second level, therefore the feed forward can determine what power is needed to get to the second level.

## Python Virtual Environment
A container for your code so that the specific libraries and their versions don't conflict with the system-wide libraries, or libraries needed for other projects.

## Library
Stored Code that can be reused for future use and be imported just as you would.

* **Phoenix**
The library for the most commonly used motor for our team.

* **Path Planner**
Works with auton planning generating optimal paths for use in the robotics competitions.

* **Photon Vision**
Library used to communicate with the cameras on the robot, used for path planning, playing a key role in how that is constructed

* **FRC3484_Lib_Python**
Our team library containing all the basic code we reuse on a yearly basis. This includes code for handling controller input, motion classes for the low level motor interactions, and pathfinding code.

* **Smart Dashboard**
Works with Driver Station to help set up robot operations in terms of visualizing sensor data and feedback from robot.