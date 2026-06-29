# Robot Container
This document describes the design reasoning by `robot_container.py` that we implemented in 2026. It also goes over the two special modes for the robot, test mode and demo mode.

> [!IMPORTANT]
> Driver station modes may change in the upcoming 2027 season with SystemCore.

The robot (and driver station itself) provides four modes. `Teleop`, `Auton`, `Test` and `Practice`. We run both test and demo mode in the driver station's test mode, by creating a SmartDashboard input box that allows the user to choose which test mode they want to run.

> [!NOTE]
> See the [full 2026 robot code](https://github.com/FRC-Team3484/X26_RobotCode/tree/main) for more complete examples for topics in this document.

## Test Mode
Test mode refers to a robot state where we can manually jog motors and read sensor values. This mode is used early in the testing and tuning phase of build season, where we need to make sure the physical mechanisms are working correctly, and that everything is wired as it should. This mode provides no extra automation, it simply takes raw user input and applies it to the motors with no other logic, including logic that would cause the robot to behave in unsafe ways. These motors run in power mode, with no velocity or position control. At times, motors can be given power limits.

First, create your test controller:
```py
# constants.py
class TestConstants:
    ...
    FLYWHEEL_INPUT: Input = ControllerMap.RIGHT_TRIGGER
    INDEXER_INPUT: Input = ControllerMap.LEFT_TRIGGER
    TURRET_INPUT: Input = ControllerMap.LEFT_JOY_X
    CLIMBER_INPUT: Input = ControllerMap.RIGHT_JOY_X

    POWER_LIMIT: float = 0.4
```

```py
# oi.py
from frc3484.controls import SC_Controller
from constants import TestConstants

class TestInterface:
    _controller: SC_Controller = SC_Controller(
        TestConstants.CONTROLLER_PORT,
        TestConstants.AXIS_LIMIT,
        TestConstants.TRIGGER_LIMIT,
        TestConstants.JOYSTICK_DEADBAND
    )
    def get_wheel_power(self) -> float:
        return self._controller1.get_axis(TestConstants.FLYWHEEL_INPUT)

    def get_indexer(self) -> float:
        return self._controller1.get_axis(TestConstants.INDEXER_INPUT) * 0.6
    
    def get_turret(self) -> float:
        return self._controller1.get_axis(TestConstants.TURRET_INPUT) * TestConstants.POWER_LIMIT

    def get_climber(self) -> float:
        return self._controller1.get_axis(TestConstants.CLIMBER_INPUT) * TestConstants.POWER_LIMIT
```

Create simple test commands for each mechanism you wish to control in test mode:
```py
# turret_test_command.py
class TurretTestCommand(Command):
    def __init__(self, oi: TestInterface1, turret_subsystem: TurretSubsystem) -> None:
        super().__init__()
        self._turret_subsystem: TurretSubsystem = turret_subsystem
        self._oi: TestInterface1 = oi
        self.addRequirements(turret_subsystem)

    @override
    def initialize(self) -> None:
        pass

    @override
    def execute(self) -> None:
        self._turret_subsystem.set_power(self._oi.get_turret())
        self._turret_subsystem.print_diagnostics()
    
    @override
    def end(self, interrupted: bool) -> None:
        self._turret_subsystem.set_power(0)

    @override
    def isFinished(self) -> bool:
        return False
```

Notice that all this command does is run the motor based on the raw test interface input.

Create your `test_container.py` that runs this command, only when enabled in SmartDashboard:

> [!NOTE]
> See the [full `test_container.py` from 2026](https://github.com/FRC-Team3484/X26_RobotCode/blob/main/src/test_container.py) for more.
```py
# test_container.py
class TestMode(Enum):
    __test__ = False
    DISABLED = 0
    MOTOR = 1
    SYSID = 2
    DEMO = 3
    LAUNCHER_RPM_TEST = 4

class TestContainer:
    __test__ = False
    def __init__(self,
            test_interface_1: TestInterface1,
            test_interface_2: TestInterface2,
            demo_interface: DemoInterface,
            sysid_interface: SysIDInterface, 
            robot_container: RobotContainer, 
        ) -> None:
        ...
        self._mode_chooser: SendableChooser = SendableChooser()

        self._mode_chooser.setDefaultOption("Disabled", TestMode.DISABLED)
        self._mode_chooser.addOption("Motor", TestMode.MOTOR)
        self._mode_chooser.addOption("SysID", TestMode.SYSID)
        self._mode_chooser.addOption("Demo", TestMode.DEMO)
        self._mode_chooser.addOption("Launcher RPM Test", TestMode.LAUNCHER_RPM_TEST)
        SmartDashboard.putData("Test Mode", self._mode_chooser)

        ...
        SmartDashboard.putBoolean("Turret Test Enabled", False)
        ...

    def get_test_command(self) -> Command:
        match self._mode_chooser.getSelected():
            case TestMode.DISABLED:
                print("[Test Container] No test mode selected, so no commands will be run")
                return Command()

            case TestMode.MOTOR:
                commands: list[Command] = []

                ...
                if SmartDashboard.getBoolean("Turret Test Enabled", False) and self._robot_container.turret_subsystem is not None:
                    commands.append(TurretTestCommand(self._test_interface_1, self._robot_container.turret_subsystem))
                ...

                return ParallelCommandGroup(*commands)

            case TestMode.SYSID:
                ...

            case TestMode.DEMO:
                ...

            case TestMode.LAUNCHER_RPM_TEST:
                ...

            case _:
                return Command()
```

Now, you can load the commands from the test container on `testInit` in `robot.py`:

> [!NOTE]
> See the [full `robot.py` from 2026](https://github.com/FRC-Team3484/X26_RobotCode/blob/main/robot.py) for a more complete example.
```py
# robot.py
...
class MyRobot(commands2.TimedCommandRobot):
    def __init__(self):
        super().__init__(RobotConstants.TICK_RATE)
        ...

        self._test_container: TestContainer = TestContainer(self._test_interface_1, self._test_interface_2, self._demo_interface, self._sysid_interface, self._robot_container)

        self._test_commands: commands2.Command = commands2.InstantCommand()
        ...

    def testInit(self):
        self._robot_container.launcher_subsystem.aim_at(TargetType.NONE)

        self._test_commands = self._test_container.get_test_command()
        self._test_commands.schedule()

    def testExit(self):
        self._test_commands.cancel()
```

### SysID Mode
Another test mode we created was SysID mode.

See the [SysID](/sys-id.md) page for more information.

## Demo Mode
Demo mode refers to a more limited teleop state, used for demonstrations, pit tests, or on the practice field. This is useful when testing the robot when some safeguards and light automation is needed, but full teleop automation isn't needed.
For example in 2026, demo mode didn't allow moving the turret past the bounds of it's track, while not enabling the auto aiming code. This is also helpful to test the tuning of parts of the robot without having automated code running.

First, create your controller:
```py
# constants.py
class DemoControllerConstants:
    ...

    TURRET_LEFT: Input = ControllerMap.LEFT_BUMPER
    TURRET_RIGHT: Input = ControllerMap.RIGHT_BUMPER
    ...
```

```py
# oi.py
from constants import DemoController

class DemoInterface:
    _demo_controller: SC_Controller = SC_Controller(
        DemoControllerConstants.CONTROLLER_PORT,
        DemoControllerConstants.AXIS_LIMIT,
        DemoControllerConstants.TRIGGER_LIMIT,
        DemoControllerConstants.JOYSTICK_DEADBAND
    )

    def demo_get_turret(self) -> float:
        return (
            self._demo_controller.get_axis(_DEMO_INPUTS.TURRET_LEFT) - 
            self._demo_controller.get_axis(_DEMO_INPUTS.TURRET_RIGHT)
        ) * 0.1 # Returns the total value to move, limited to 10% power
```

> [!NOTE]
> In 2026, we created a large [`demo_command.py`](https://github.com/FRC-Team3484/X26_RobotCode/blob/main/src/commands/test/demo_command.py) which had smaller commands for each feature to use in demo mode. There could be other ways to do this as well.

Create your `demo_command.py`
```py
# demo_command.py

class DemoCommand(ParallelCommandGroup):
    def __init__(self, oi: DemoInterface, *commands: Command):
        super().__init__(*commands)
        self.oi = oi

    ...
    def add_turret(self, subsystem: TurretSubsystem):
        self.addCommands(DemoTurret(subsystem, self.oi))
    ...

...    
class DemoTurret(Command):
    def __init__(self, turret: TurretSubsystem, oi: DemoInterface):
        self.addRequirements(turret)
        self.turret = turret
        self.oi = oi
    
    def execute(self):
        turret_request: float = self.oi.demo_get_turret()
        if self.turret.get_position() >= TurretSubsystemConstants.MAXIMUM_ANGLE and turret_request > 0:
            turret_request = 0
            self.oi.set_rumble(UserInterface.Operator.RUMBLE_HIGH)
        elif self.turret.get_position() <= TurretSubsystemConstants.MINIMUM_ANGLE and turret_request < 0:
            turret_request = 0
            self.oi.set_rumble(UserInterface.Operator.RUMBLE_HIGH)
        else:
            self.oi.set_rumble(UserInterface.Operator.RUMBLE_OFF)
        
        self.turret.set_power(turret_request)
    
    def end(self, interrupted: bool):
        return super().end(interrupted)
    
    def isFinished(self) -> bool:
        return False
...
```

Then, use the `demo_command.py` in your `test_container.py`
```py
# test_container.py

...
class TestMode(Enum):
    __test__ = False
    DISABLED = 0
    MOTOR = 1
    SYSID = 2
    DEMO = 3
    LAUNCHER_RPM_TEST = 4


class TestContainer:
    __test__ = False
    """
    Handles test commands

    Parameters:
        - robot_container (`RobotContainer`): the robot container
        - oi (`TestInterface`): the oi test interface
    """
    def __init__(self,
            test_interface_1: TestInterface1,
            test_interface_2: TestInterface2,
            demo_interface: DemoInterface,
            sysid_interface: SysIDInterface, 
            robot_container: RobotContainer, 
        ) -> None:
        ...
        self._mode_chooser: SendableChooser = SendableChooser()
        self._sysid_chooser: SendableChooser = SendableChooser()

        self._mode_chooser.setDefaultOption("Disabled", TestMode.DISABLED)
        self._mode_chooser.addOption("Motor", TestMode.MOTOR)
        self._mode_chooser.addOption("SysID", TestMode.SYSID)
        self._mode_chooser.addOption("Demo", TestMode.DEMO)
        self._mode_chooser.addOption("Launcher RPM Test", TestMode.LAUNCHER_RPM_TEST)
        SmartDashboard.putData("Test Mode", self._mode_chooser)
        ...

    def get_test_command(self) -> Command:
        match self._mode_chooser.getSelected():
            case TestMode.DISABLED:
                print("[Test Container] No test mode selected, so no commands will be run")
                return Command()

            case TestMode.MOTOR:
                ...

            case TestMode.SYSID:
                ...

            case TestMode.DEMO:
                return self.get_demo_command()

            case TestMode.LAUNCHER_RPM_TEST:
                ...

            case _:
                return Command()

    def get_demo_command(self) -> Command:
        _demo_command = DemoCommand(self._demo_interface)
        if self._robot_container.intake_subsystem is not None:
            _demo_command.add_intake(self._robot_container.intake_subsystem)
        
        if self._robot_container.climber_subsystem is not None:
            _demo_command.add_climb(self._robot_container.climber_subsystem)

        if self._robot_container.feeder_subsystem is not None:
            _demo_command.add_feeder(self._robot_container.feeder_subsystem)

        if self._robot_container.flywheel_subsystem is not None:
            _demo_command.add_flywheel(self._robot_container.flywheel_subsystem)
        
        if self._robot_container.indexer_subsystem is not None:
            _demo_command.add_indexer(self._robot_container.indexer_subsystem)
        
        if self._robot_container.turret_subsystem is not None:
            _demo_command.add_turret(self._robot_container.turret_subsystem)

        if self._robot_container.drivetrain_subsystem is not None:
            #_demo_command.add_drive(self._robot_container.drivetrain_subsystem)
            _demo_command.addCommands(DriveTestCommand(self._robot_container.drivetrain_subsystem, self._demo_interface))

        return _demo_command
```

## Robot Container
In 2026, we opted for using a `robot_container.py` file to handle most of the subsystem initialization and other logic. This made the logic in `robot.py` much cleaner, and separated each of the main robot states (test, demo, sysid, ect.) into their own files.

> [!NOTE]
> See the [full `robot_container.py` from 2026](https://github.com/FRC-Team3484/X26_RobotCode/blob/main/src/robot_container.py) file for a complete example.

First, we created a `config.py` file. This let us enable and disable major parts of the robot, if they weren't working or ready to test.
```py
# config.py

DRIVETRAIN_ENABLED: bool = True
VISION_ENABLED: bool = True

CLIMBER_ENABLED: bool = False
FLYWHEEL_ENABLED: bool = True
FEEDER_ENABLED: bool = True
INDEXER_ENABLED: bool = True
INTAKE_ENABLED: bool = True
TURRET_ENABLED: bool = True
FEED_TARGET_ENABLED: bool = True
LEDS_ENABLED: bool = True
...
```

Then, `robot_container.py` becomes responsible for initializing each subsystem. If they are enabled, a variable is created in `__init__` for that subsystem, and then `@property` functions are used to access these subsystems later:
```py
# robot_container.py

class RobotContainer:
    def __init__(self, driver_interface: DriverInterface, operator_interface: OperatorInterface) -> None:
        self._driver_interface: DriverInterface = driver_interface
        self._operator_interface: OperatorInterface = operator_interface

        ...
        
        # Subsystems
        ... 
        if config.TURRET_ENABLED:
            self._turret_subsystem: TurretSubsystem = TurretSubsystem()
        ...

    # Subsystem Properties
    ...
    @property
    def turret_subsystem(self) -> TurretSubsystem | None:
        if config.TURRET_ENABLED:
            return self._turret_subsystem
        else:
            print("[RobotContainer] Unable to return TurretSubsystem because it is disabled")
            return None
    ...
```

`robot_container.py` is also responsible for creating the command groups for each significant robot state. We also use `@property` functions to use these command groups from elsewhere:
```py
# robot_container.py

class RobotContainer:
    def __init__(self, driver_interface: DriverInterface, operator_interface: OperatorInterface) -> None:
        self._driver_interface: DriverInterface = driver_interface
        self._operator_interface: OperatorInterface = operator_interface

        # Command Groups
        self._intake_commands: ParallelCommandGroup = ParallelCommandGroup()
        self._feed_commands: ParallelCommandGroup = ParallelCommandGroup()
        self._launch_commands: ParallelCommandGroup = ParallelCommandGroup()
        self._goto_climb_commands: ParallelCommandGroup = ParallelCommandGroup()

        if config.INTAKE_ENABLED:
            self._intake_commands.addCommands(
                TeleopIntakeCommand(self._intake_subsystem, self._operator_interface)
            )
            self._feed_commands.addCommands(
                TeleopIntakeCommand(self._intake_subsystem, self._operator_interface)
            )
            self._launch_commands.addCommands(
                TeleopIntakeCommand(self._intake_subsystem, self._operator_interface)
            )
        if config.TURRET_ENABLED and self.launcher_subsystem and self.feed_target_subsystem:
            self._intake_commands.addCommands(
                TeleopTurretTrackingCommand(self._operator_interface, self.launcher_subsystem, self._feed_target_subsystem, self._drivetrain_subsystem, self.feed_target_subsystem)
            )
        if config.CLIMBER_ENABLED:
            self._intake_commands.addCommands(
                TeleopClimbCommand(self._operator_interface, self._climber_subsystem)
            )
            self._goto_climb_commands.addCommands(
                self.goto_climb()
            )
        if config.DRIVETRAIN_ENABLED:
            self._intake_commands.addCommands(
                TeleopDriveCommand(self._drivetrain_subsystem, self._driver_interface)
            )
            self._feed_commands.addCommands(
                TeleopDriveCommand(self._drivetrain_subsystem, self._driver_interface)
            )
            self._launch_commands.addCommands(
                TeleopDriveSlowCommand(self._drivetrain_subsystem, self._driver_interface)
            )
        if self.launcher_subsystem:
            self._launch_commands.addCommands(
                TeleopLaunchCommand(self.launcher_subsystem, self._operator_interface, self._feed_target_subsystem)
            )
            self._feed_commands.addCommands(
                TeleopLaunchCommand(self.launcher_subsystem, self._operator_interface, self._feed_target_subsystem)
            )

    ...

    # Command Group Properties
    @property
    def teleop_intake_commands(self) -> Command:
        return self._intake_commands

    @property
    def teleop_feed_commands(self) -> Command:
        return self._feed_commands

    @property
    def teleop_launch_commands(self) -> Command:
        return self._launch_commands

    @property
    def goto_climb_commands(self) -> Command:
        return self._goto_climb_commands

    ...
```

### PDP Logging
A PDP logging system is also something we implemented in 2026. This is something that `robot_container.py` handles.

This allowed us to enable PDP logging through SmartDashboard, and see power draw and other information for each channel on the PDP:
```py
# robot_container.py

class RobotContainer:
    def __init__(self, driver_interface: DriverInterface, operator_interface: OperatorInterface) -> None:
        self._driver_interface: DriverInterface = driver_interface
        self._operator_interface: OperatorInterface = operator_interface

        SmartDashboard.putBoolean("Get PDP Data", False)

        self._pdp: PowerDistribution = PowerDistribution(1, PowerDistribution.ModuleType.kRev)

    # Other Properties
    @property
    def get_pdp_data(self) -> bool:
        """
        Returns whether or not to get PDP data
        """
        return SmartDashboard.getBoolean("Get PDP Data", False)

    def write_pdp_data(self) -> None:
        """
        Writes PDP data to SmartDashboard

        Called by robot when get_pdp_data is true
        """
        SmartDashboard.putNumber("PDP Voltage (Volts)", self._pdp.getVoltage())
        SmartDashboard.putNumber("PDP Total Current (Amperes)", self._pdp.getTotalCurrent())
        SmartDashboard.putNumber("PDP Total Power (Watts)", self._pdp.getTotalPower())

        for i in range(self._pdp.getNumChannels()):
            try:
                SmartDashboard.putNumber(f"PDP Channel {i} (Amperes)", self._pdp.getCurrent(i))
            except:
                SmartDashboard.putString(f"PDP Channel {i} (Amperes)", "NaN")
```

These PDP logging functions are later used in `robot.py` to actually log the data:
```py
# robot.py
class MyRobot(commands2.TimedCommandRobot):
    def robotPeriodic(self):
        self.trigger_animations()
        SmartDashboard.putNumber("Battery Voltage", DriverStation.getBatteryVoltage())
        SmartDashboard.putNumber("Match Time", DriverStation.getMatchTime())

        if self._robot_container.get_pdp_data:
            self._robot_container.write_pdp_data()
```

## Robot
All of these different containers come together in `robot.py`. From there we have a state machine for each major robot state, and schedule commands from `robot_container.py`, test commands from `test_container.py`, and autons from `auton_generator.py`.

> [!NOTE]
> See the [full `robot.py`](https://github.com/FRC-Team3484/X26_RobotCode/blob/main/robot.py) for the complete example.

```py
# robot.py

class State(Enum):
    INTAKE = 0
    FEED = 1
    SCORE = 2
    GOTO_CLIMB = 3

class MyRobot(commands2.TimedCommandRobot):
    def __init__(self):
        super().__init__(RobotConstants.TICK_RATE)

        self._driver_interface: DriverInterface = DriverInterface()
        self._operator_interface: OperatorInterface = OperatorInterface()

        self._test_interface_1: TestInterface1 = TestInterface1()
        self._test_interface_2: TestInterface2 = TestInterface2()
        self._demo_interface: DemoInterface = DemoInterface()
        self._sysid_interface: SysIDInterface = SysIDInterface()

        self._robot_container: RobotContainer = RobotContainer(self._driver_interface, self._operator_interface)
        self._auton_generator: AutonGenerator = AutonGenerator(self._robot_container.drivetrain_subsystem, self._robot_container.launcher_subsystem, self._robot_container.intake_subsystem, self._robot_container.feed_target_subsystem, self._robot_container.feeder_subsystem)
        self._test_container: TestContainer = TestContainer(self._test_interface_1, self._test_interface_2, self._demo_interface, self._sysid_interface, self._robot_container)
        self._state: State = State.INTAKE

        self._test_commands: commands2.Command = commands2.InstantCommand()

    def autonomousInit(self):
        if self._robot_container.turret_subsystem:
            self._robot_container.turret_subsystem.reset()
        self._auton_generator.get_auton_command().schedule()

    def teleopInit(self):
        if self._robot_container.turret_subsystem:
            self._robot_container.turret_subsystem.reset()
            
        self.start_intake_state()

    def teleopPeriodic(self):
        match self._state:
            case State.INTAKE:
                if self._operator_interface.get_left_feed_point() or self._operator_interface.get_right_feed_point():
                    self.start_feed_state()
                elif self._operator_interface.get_launcher():
                    self.start_launch_state()
                elif self._driver_interface.get_goto_climb():
                    self.start_goto_climb_state()

            case State.FEED:
                if not (self._operator_interface.get_left_feed_point() or self._operator_interface.get_right_feed_point()):
                    self.start_intake_state()

            case State.SCORE:
                if not self._operator_interface.get_launcher():
                    self.start_intake_state()

            case State.GOTO_CLIMB:
                if not self._driver_interface.get_goto_climb():
                    self.start_intake_state()

    def teleopExit(self):
        self._state = State.INTAKE
        self.stop_teleop_commands()

    def testInit(self):
        self._robot_container.launcher_subsystem.aim_at(TargetType.NONE)

        self._test_commands = self._test_container.get_test_command()
        self._test_commands.schedule()

    def testExit(self):
        self._test_commands.cancel()

    def stop_teleop_commands(self) -> None:
        self._robot_container.teleop_intake_commands.cancel()
        self._robot_container.teleop_feed_commands.cancel()
        self._robot_container.teleop_launch_commands.cancel()
        self._robot_container.goto_climb_commands.cancel()

    def start_intake_state(self) -> None:
        self._state = State.INTAKE
        self.stop_teleop_commands()
        self._robot_container.teleop_intake_commands.schedule()

    def start_feed_state(self) -> None:
        self._state = State.FEED
        self.stop_teleop_commands()
        self._robot_container.teleop_feed_commands.schedule()
    
    def start_launch_state(self) -> None:
        self._state = State.SCORE
        self.stop_teleop_commands()
        self._robot_container.teleop_launch_commands.schedule()
    
    def start_goto_climb_state(self) -> None:
        self._state = State.GOTO_CLIMB
        self.stop_teleop_commands()
        self._robot_container.goto_climb_commands.schedule()
```