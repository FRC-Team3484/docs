# Season Steps
This document goes over most of the steps and tasks done in a season, from a programming perspective.

- [Season Steps](#season-steps)
  - [Season Start](#season-start)
  - [Develop Robot Code](#develop-robot-code)
  - [Robot Setup](#robot-setup)
  - [Testing and Tuning](#testing-and-tuning)
  - [Driver Practice and Autons](#driver-practice-and-autons)

## Season Start
- Work with the team to plan robot strategy, robot design, and prototyping.
- Once a design is mostly formed, begin planning robot code structure.
    - Subsystems - What major mechanisms are there on the robot? 
        - A major mechanism is something like the drivetrain, an intake, or a game piece manipulator.
        - Consider how major mechanisms interact with each other, and how that could translate in code.
        - Eg. Should the intake and indexer work together as one subsystem, or would it be easier to have two subsystems controlled in a command? If a series of mechanisms has more than one state or action it will perform, it should probably be split into several subsystems.
        - Subsystems should only control the hardware level components, any higher level logic should be the job of a command.
        - You could consider making a subsystem that "wraps" other subsystems. This was necessary in 2026 where our `LauncherSubsystem` coordinated the logic for the feeder, flywheel, turret and indexer to launch pieces. Note that this extra logic was okay, as this logic was necessary for all operations of the robot. If this was logic that we were only going to need during some robot operations, this logic should have been made into a command instead.
    - Commands - What major actions will the robot perform?
        - A command should be created for things like scoring, driving, or intaking.
        - Commands should use the low level hardware access of one or more subsystems, and use other logic like button presses or sensors to perform .actions
        - Consider the different states of your robot, and which things you want it to do. Commands should be organized in a way to perform one single action, and then chained together if needed.
        - Plan commands for test mode, teleop, and auton. See [Robot Container](/robot-container.md) for more information.
- Robot states
    - Robot state - Which commands should run when?
        - These states will later be parts of the main robot code state machine.
        - Eg. The robot should only launch pieces when not intaking. When launching, the robot should drive slowly. Now you have two distinct states, with 4 different commands.
    - Automation - Given this robot design, what parts of operating the robot can we automate?
        - Once ideas start being developed for robot design and strategy, you can start to think about ways some of these mechanisms can be automated to improve strategy.
        - The goal is to make it easier to use the robot, automations that make the robot less precise or slower to use is not a good automation.
        - Eg. We need to launch pieces into a very small goal, so we should create code so the robot can automatically aim itself.
    - Don't make autons the main priority, but pay attention to other team's auton ideas, and how our own robot strategy could play into the autonomous period.
    - Once a robot code design has been created, write it out somewhere central, with clearly separated subsystems, commands, and robot states.

## Develop Robot Code
> [!IMPORTANT]
> Parts of this section are likely to change with the introduction of SystemCore for the 2027 season. `robotpy` will likely have some significant changes to how creating and managing projects will work. Other third party libraries may take extra long to be available for the season. Update this section later accordingly.

- Create a new robot project.
    - Create a new GitHub repository for the project. Set up `robotpy`, the `pyproject.toml`, and your `requirements.txt`.
    - Ensure you've added dependencies that you will need, such as AprilTags, `phoenix6`, and `photonlibpy`. Also add [`FRC3484_Lib`](https://github.com/FRC-Team3484/FRC3484_Lib_Python)
      - There is a possibility that some libraries will not yet be updated or working for the new season.
    - Add these initial files to the `main` branch in the repo, and commit them so everyone can access them. See [Repository Good Practices](/repository-good-practices.md) for more information.
    - Ensure that setting up the project is as simple as running `git clone` and then installing the Python dependencies. This will make it easier to update the dependencies for the project as it grows.
- Begin working on subsystems.
    - During development, each subsystem should be worked on into its own branch. Once it is completed, then it can be merged into `main`.
    - Subsystems should only control motors and read from sensors. No additional logic should be added, unless that logic will be needed for every operation on the robot. Subsystems should be kept clean and free from clutter if possible.
- Migrate in common code
    - Things that we have done in the past, like drivetrain code, teleop drive command, the robot container, and vision odometry are not likely to change too much between seasons.
    - Migrate the code from past seasons into the new robot project.
    - These features do not need to be fully implemented into the robot's state logic until commands have been finished
- Begin developing OI.
    - Based on the planned robot states, you will likely have pretty simply defined controller buttons.
    - One controller handles all driving inputs, and the other handles all operator inputs.
    - Work with the drive team to decide how the operator wants to interact with the robot to move into each state.
    - Then, create the operator inputs in OI accordingly.
- Next, move into developing commands.
    - Commands represent a single action that the robot will perform.
    - You will need commands for test mode, teleop, and auton. See [Robot Container](/robot-container.md) for more information.
    - Commands can react to subsystem level sensors, and OI level buttons, to control motors using subsystem functions.
    - Common commands control the drivetrain or intake, index or launch game pieces.
    - Commands can have extra logic, or move between several states. Ideally however, commands should end up simple and easy to read. Many state machines beyond the one controlling the main robot state can cause parts of the robot to be active when not intended. Making more commands for each robot state is sometimes better.
      - Consider if there is a single `IntakeCommand`. This command is active any time the intake is needed, which includes during demo mode, test mode, teleop, and auton. 
      - This command will need at least 4 different states, and will have to do a lot of logic to make sure the subsystems are switching between open and closed loop control based on the robot state. 
      - Instead, this could turn into a `TeleopIntakeCommand`, `TestIntakeCommand` and `AutonIntakeCommand`, moving the command's state machine to just run the needed commands based on the robot state
- Now you can start putting together the robot state machine, the [Robot Container](/robot-container.md), and the various other test and SysID containers.
    - See [Robot Container](/robot-container.md) and [SysID](/sys-id.md) for more information.
- While robot power on, testing, and tuning are always the first priority, consider starting to develop autons if you have time.
    - When deciding what autons to make, use your robot strategy and the auton ideas of other teams.
    - See [Autons](/autons.md) for more information.

## Robot Setup
> [!NOTE]
> Some steps in this section may change due to the introduction of SystemCore in 2027. Update this section accordingly if this is the case.

- Once most of the main mechanisms, wiring, and other physical robot features are finished, it is time for S/C to do the initial power on steps of the robot.
    - Turn the robot on. You're already winning if there's no smoke.
    - Hopefully the radio you have on the robot is a lab radio, and it works. If it doesn't you may need to skip ahead to updating or configuring the radio.
    - Check the CAN bus. Use Phoenix Tuner to verify that all devices are visible on CAN.
    - Check other devices not on CAN. These may be sensors, encoders, or PhotonVision co processors.
- If needed, update the RoboRIO. See [WPIlib docs](https://docs.wpilib.org/en/stable/docs/zero-to-robot/step-3/index.html).
- If needed, update or reconfigure the radio. See the [radio docs](https://frc-radio.vivid-hosting.net/)
- Once all the CAN devices are visible, and not at risk of suddenly being disconnected, update all the CAN devices. This is done through Phoenix Tuner.

## Testing and Tuning
- Now that the robot is electrically sound, it is time to begin testing each robot mechanism. This will help identify any potential mechanical issues.
- Using the test mode feature, jog each motor, and verify that sensors are reporting the correct value.
- Once each mechanism seems to be operating as was intended, it is time to tune each motor. See [SysID](/sys-id.md) for information on SysID and tuning.
- Then, demo mode can be used to see how the motors and mechanisms behave with PID and feed forward control, as well as further testing each mechanism. Additional iterations of tuning may be necessary.
- Now, configure vision.
    - Ensure all the PhotonVision co processors are visible on the robot's network, and access their configuration web pages. 
    - Calibrate each camera, and ensure that you enable 3D mode for each one. 
    - Get the camera offsets relative to the center of the robot, and apply them in code.
    - Ensure that vision pose estimation is working correctly, by holding up an AprilTag to the robot's camera.
    - See [PhotonVision docs](https://photonvision.org/) for more information.
    - Also see [FRC3484_Lib's vision documentation](https://github.com/FRC-Team3484/FRC3484_Lib_Python/tree/main/src/frc3484/__lib/vision).
- Once all the mechanisms have been tuned, enable the robot in teleop. Test all features of the robot, and ensure all robot automation is working as expected.

## Driver Practice and Autons
- Now that the robot is mostly working as expected, move into driver practice. As the driver and operator use the robot, make adjustments to the controls, fix smaller bugs, and do other changes as needed.
- If possible, try to test autons. This is often hard to test until at a proper competition with a full test field available, however at least try to verify that the robot attempts to follow paths and runs the correct commands.
    - The most common issue with autons is the robot not driving far enough or driving too far. This can be fixed with better drivetrain tuning, or drive scaling.