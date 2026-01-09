## Subsystems
Code for the indiviual systems that make up the robot such as: launchers, intakes, elevators, etc.
This adds substance to these parts of the system so they can be controlled in sync later

## Commands
Declares what states the robot  may encounter and how it must function accordingly
Commands combine multiple subsytems into the nessisary action for the robot
Example, if you wanted to intake from the feeder station you would need to first utilize the elevator subsystem to raise the manipulator and then the pivot subsystem to line it up and the intake subsystem to actually take in the piece. 

## PID
Helps motors to get from point A to B in a stable manner, each part of the Proportional Intergrale Dirivitive works through part of this process to make movement smooth, and values inserted will be adjusted.

## Robot Py
The toolkit the team uses to code the robot.

## State Machines
Goes through a process of checking each given variable in the machine through "if", "elif", and "else" statements. The code checks each variable in decending order. If the variable given in one of these cases matches, the code then works through the process given within it.

## Feed Forward
An equation that can firgure out how much power is nessacary for a motor/system to move/accomplish its task. Example, the elavator needs to move to the second level, therefore the feet foward can determine what power is needed to get to the econd level.

## Python Virtual Enviorment
Its a container for your code so that the code in the container doesn't go with the other code.

## Library
Stored Code that can be reused for future use and be imported just as you would.

* **Phoenix**
The library for the most commonly used motor for our team.

* **Path Planner**
Works with auton planning generating optimal paths for use in the robotics competitions.

* **Photon Vision**
Library used to comunicate with the cameras on the robot, used for path planning, playing a key role in how that is constructed

* **FRC3484_Lib_Python**
Our team library containing all of the basic code we reuse on a yearly basis

* **Smart Dashboard**
Works with Driver Station to help set up robot operations in terms of visualizing sensor data and feedback from robot