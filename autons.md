# Autons
This document goes over how to use Pathplanner to create autons, and the Python code for loading PathPlanner paths

> [!NOTE]
> PathPlanner often has breaking changes between seasons. Update this document accordingly if something changes.

See [PathPlanner's documentation](https://pathplanner.dev/pathplanner-gui.html) for how to download the PathPlanner app and use it to create paths.

## Auton Generator
Once you've created paths, create an `auton_generator.py` which returns them. Ensure that you've [Configured AutoBuilder](https://pathplanner.dev/pplib-getting-started.html#holonomic-swerve) in your drivetrain.

`NamedCommands` are commands that can be run either in a command group, or somewhere along a path, through the PathPlanner GUI.

`AutonGenerator` is fully responsible for creating and managing the auton input in SmartDashboard.

```py
# auton_generator.py

class AutonMode(Enum):
    NONE = 0
    SCORE = 1
    LEFT_DEPOT = 2
    CENTER_DEPOT = 3
    DRIVE_SCALING_TEST = 4

class AutonGenerator:
    def __init__(self, drivetrain_subsystem: DrivetrainSubsystem | None, launcher_subsystem: LauncherSubsystem | TurretlessLauncherSubsystem | None, intake_subsystem: IntakeSubsystem | None, feed_target_subsystem: FeedTargetSubsystem | None, feeder_subsystem: FeederSubsystem | None) -> None:
        self._drivetrain_subsystem: DrivetrainSubsystem | None = drivetrain_subsystem
        self._launcher_subsystem: LauncherSubsystem | TurretlessLauncherSubsystem | None = launcher_subsystem
        self._intake_subsystem: IntakeSubsystem | None = intake_subsystem
        self._feed_target_subsystem: FeedTargetSubsystem | None = feed_target_subsystem
        self._feeder_subsystem: FeederSubsystem | None = feeder_subsystem

        # Register NamedCommands
        if self._intake_subsystem:
            NamedCommands.registerCommand("Intake", AutonIntakeCommand(self._intake_subsystem))
        else:
            print("[Auton Generator] Unable to register named intake command")
            NamedCommands.registerCommand("Intake", InstantCommand())

        if self._launcher_subsystem and self._feed_target_subsystem:
            NamedCommands.registerCommand("Feed Left", AutonScoreCommand(self._launcher_subsystem, TargetType.TARGET_1))
            NamedCommands.registerCommand("Feed Right", AutonScoreCommand(self._launcher_subsystem, TargetType.TARGET_2))
            NamedCommands.registerCommand("Score", AutonScoreCommand(self._launcher_subsystem, TargetType.HUB))
        else:
            print("[Auton Generator] Unable to register named feed commands")
            NamedCommands.registerCommand("Feed Left", InstantCommand())
            NamedCommands.registerCommand("Feed Right", InstantCommand())
            NamedCommands.registerCommand("Score", InstantCommand())

        if self._feeder_subsystem:
            NamedCommands.registerCommand("Done Launching Command", DoneLaunchingCommand(self._feeder_subsystem))
        else:
            print("[Auton Generator] Unable to register named done launching command")
            NamedCommands.registerCommand("Done Launching Command", InstantCommand())

        # Set up auton chooser
        self._auton_chooser: SendableChooser = SendableChooser()

        self._auton_chooser.setDefaultOption("None", AutonMode.NONE)
        self._auton_chooser.addOption("Score", AutonMode.SCORE)
        self._auton_chooser.addOption("Left Depot", AutonMode.LEFT_DEPOT)
        self._auton_chooser.addOption("Center Depot", AutonMode.CENTER_DEPOT)
        self._auton_chooser.addOption("Drive Scaling Test", AutonMode.DRIVE_SCALING_TEST)
        SmartDashboard.putData("Auton Mode", self._auton_chooser)

    def _load_auto(self, path_name: str) -> Command:
        return PathPlannerAuto(path_name)

    def get_auton_command(self) -> Command:
        if self._drivetrain_subsystem is None:
            print("[Auton Generator] No drivetrain subsystem, so no commands will be run")
            return Command()

        match self._auton_chooser.getSelected():
            case AutonMode.NONE:
                return InstantCommand()
            
            case AutonMode.SCORE:
                return self._load_auto(path_name="Score")
            
            case AutonMode.LEFT_DEPOT:
                return self._load_auto(path_name="Left Depot")
            
            case AutonMode.CENTER_DEPOT:
                return self._load_auto(path_name="Center Depot")
            
            case AutonMode.DRIVE_SCALING_TEST:
                return self._load_auto(path_name="Drive Scaling Test")

            case _:
                print("[Auton Generator] No auton mode selected, so no commands will be run")
                return Command()
```

Then, the auton commands are scheduled in `robot.py`:
```py
# robot.py

class MyRobot(commands2.TimedCommandRobot):
    def __init__(self):
        super().__init__(RobotConstants.TICK_RATE)

        self._robot_container: RobotContainer = RobotContainer(self._driver_interface, self._operator_interface)
        self._auton_generator: AutonGenerator = AutonGenerator(self._robot_container.drivetrain_subsystem, self._robot_container.launcher_subsystem, self._robot_container.intake_subsystem, self._robot_container.feed_target_subsystem, self._robot_container.feeder_subsystem)
        ...

    def autonomousInit(self):
        self._auton_generator.get_auton_command().schedule()
    ...
```