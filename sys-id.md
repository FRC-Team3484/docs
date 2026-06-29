# Sys ID
SysID mode was another test mode we created for running SysID routines during testing.

Similar to how we used [Robot Container](/robot-container.md) and `test_container.py` for splitting up all of the different test modes and operations, we did the same and created a `sysid_container.py`.

First, we needed to create a controller interface for SysID:
```py
# constants.py
class UserInterface:
    class SysidController:
        QUASI_FWD_BUTTON: Input = ControllerMap.A_BUTTON
        QUASI_REV_BUTTON: Input = ControllerMap.B_BUTTON
        DYNAMIC_FWD_BUTTON: Input = ControllerMap.X_BUTTON
        DYNAMIC_REV_BUTTON: Input = ControllerMap.Y_BUTTON
```

```py
# oi.py
class SysIDInterface:
    _sysid_controller: SC_Controller = SC_Controller(
        UserInterface.SysidController.CONTROLLER_PORT,
        UserInterface.SysidController.AXIS_LIMIT,
        UserInterface.SysidController.TRIGGER_LIMIT,
        UserInterface.SysidController.JOYSTICK_DEADBAND
    )
        
    def get_quasistatic_forward(self) -> bool:
        return self._sysid_controller.get_button(UserInterface.SysidController.QUASI_FWD_BUTTON)

    def get_quasistatic_reverse(self) -> bool:
        return self._sysid_controller.get_button(UserInterface.SysidController.QUASI_REV_BUTTON)

    def get_dynamic_forward(self) -> bool:
        return self._sysid_controller.get_button(UserInterface.SysidController.DYNAMIC_FWD_BUTTON)

    def get_dynamic_reverse(self) -> bool:
        return self._sysid_controller.get_button(UserInterface.SysidController.DYNAMIC_REV_BUTTON)
```

Then, each subsystem that we wanted to support SysID routines, needed to add some functions for logging and running the SysID command:
```py
# flywheel_subsystem.py

class FlywheelSubsystem(Subsystem):
    def __init__(self) -> None:
        super().__init__()

        self._sys_id_routine: SysIdRoutine = SysIdRoutine(
            SysIdRoutine.Config(
                # Use default ramp rate (1 V/s) and timeout (10 s)
                # Reduce dynamic voltage to 4 V to prevent brownout
                stepVoltage=4.0
            ),
            SysIdRoutine.Mechanism(
                self._set_voltage,
                self._log_motors,
                self,
                'flywheel'
            )
        )

    def _set_voltage(self, voltage: volts) -> None:
        self._motor.set_raw_voltage(voltage)

    def _log_motors(self, log: SysIdRoutineLog) -> None:
        log.motor(f'flywheel') \
            .voltage(self._motor.get_raw_voltage()) \
            .angularPosition(self._motor.get_raw_position()) \
            .angularVelocity(self._motor.get_raw_velocity())

    def get_sysid_command(self, mode: Literal['quasistatic', 'dynamic'], direction: SysIdRoutine.Direction) -> Command:
        if mode == "quasistatic":
            return self._sys_id_routine.quasistatic(direction)
        elif mode == "dynamic":
            return self._sys_id_routine.dynamic(direction)
```

Finally, our `sysid_container.py` has a function for each routine to run, then uses a `SelectCommand` to associate our SysID buttons to the correct function to run.

```py
# sysid_container.py

from typing import Literal

from commands2 import Command, SelectCommand
from commands2.sysid import SysIdRoutine

from src.oi import SysIDInterface
from src.subsystems.drivetrain_subsystem import DrivetrainSubsystem
from src.subsystems.feeder_subsystem import FeederSubsystem
from src.subsystems.flywheel_subsystem import FlywheelSubsystem
from src.subsystems.turret_subsystem import TurretSubsystem


class SysIDContainer():
    """
    Handles SysID commands
    """
    def __init__(self, 
            oi: SysIDInterface, 
            flywheel_subsystem: FlywheelSubsystem | None = None
        ) -> None:
        self._oi: SysIDInterface = oi
        self._flywheel_subsystem: FlywheelSubsystem | None = flywheel_subsystem

    def get_flywheel_sysid(self) -> Command:
        if self._flywheel_subsystem is not None:
            quasistatic_forward: Command = self._flywheel_subsystem.get_sysid_command('quasistatic', SysIdRoutine.Direction.kForward)
            quasistatic_reverse: Command = self._flywheel_subsystem.get_sysid_command('quasistatic', SysIdRoutine.Direction.kReverse)
            dynamic_forward: Command = self._flywheel_subsystem.get_sysid_command('dynamic', SysIdRoutine.Direction.kForward)
            dynamic_reverse: Command = self._flywheel_subsystem.get_sysid_command('dynamic', SysIdRoutine.Direction.kReverse)

            return SelectCommand(
                {
                    'quasistatic_forward': quasistatic_forward,
                    'quasistatic_reverse': quasistatic_reverse,
                    'dynamic_forward': dynamic_forward,
                    'dynamic_reverse': dynamic_reverse
                },
                lambda: 'quasistatic_forward' if self._oi.get_quasistatic_forward() else (
                    'quasistatic_reverse' if self._oi.get_quasistatic_reverse() else (
                        'dynamic_forward' if self._oi.get_dynamic_forward() else (
                            'dynamic_reverse' if self._oi.get_dynamic_reverse() else None
                        )
                    )
                )
            ).onlyWhile(
                lambda: self._oi.get_quasistatic_forward()
                    or self._oi.get_quasistatic_reverse()
                    or self._oi.get_dynamic_forward()
                    or self._oi.get_dynamic_reverse()) \
            .repeatedly()
        else:
            print("[SysID Container] Unable to return flywheel SysID commands because FlywheelSubsystem is None")
            return Command()
```

Using these functions corresponds to the SysID test mode in `test_container.py`:
```py
# test_container.py

from enum import Enum
from commands2 import Command, ParallelCommandGroup
from wpilib import SendableChooser, SmartDashboard

from src.sysid_container import SysIDContainer
from src.commands.test.test_drive_command import DriveTestCommand
from src.oi import SysIDInterface, TestInterface1, TestInterface2, DemoInterface
from src.robot_container import RobotContainer
from src.commands.test.demo_command import DemoCommand
from src.commands.test.climber_test_command import ClimberTestCommand
from src.commands.test.flywheel_test_command import FlywheelTestCommand
from src.commands.test.feeder_test_command import FeederTestCommand
from src.commands.test.indexer_test_command import IndexerTestCommand
from src.commands.test.intake_test_command import IntakeTestCommand
from src.commands.test.turret_test_command import TurretTestCommand
from src.commands.test.launcher_rpm_test_command import LauncherRpmTestCommand


class TestMode(Enum):
    __test__ = False
    DISABLED = 0
    MOTOR = 1
    SYSID = 2
    DEMO = 3
    LAUNCHER_RPM_TEST = 4

class SysIDMode(Enum):
    DISABLED = 0
    DRIVETRAIN_DRIVE = 1
    DRIVETRAIN_STEER = 2
    FLYWHEEL = 3
    FEEDER_BOTTOM = 4
    FEEDER_TOP = 5
    TURRET = 6

class TestContainer:
    def __init__(self,
            test_interface_1: TestInterface1,
            test_interface_2: TestInterface2,
            demo_interface: DemoInterface,
            sysid_interface: SysIDInterface, 
            robot_container: RobotContainer, 
        ) -> None:
        self._test_interface_1: TestInterface1 = test_interface_1
        self._test_interface_2: TestInterface2 = test_interface_2
        self._demo_interface: DemoInterface = demo_interface
        self._sysid_interface: SysIDInterface = sysid_interface
        self._robot_container: RobotContainer = robot_container

        self._sysid_container: SysIDContainer = SysIDContainer(self._sysid_interface, self._robot_container.drivetrain_subsystem, self._robot_container.flywheel_subsystem, self._robot_container.feeder_subsystem, self._robot_container.turret_subsystem)

        self._mode_chooser: SendableChooser = SendableChooser()
        self._sysid_chooser: SendableChooser = SendableChooser()

        self._mode_chooser.setDefaultOption("Disabled", TestMode.DISABLED)
        self._mode_chooser.addOption("Motor", TestMode.MOTOR)
        self._mode_chooser.addOption("SysID", TestMode.SYSID)
        self._mode_chooser.addOption("Demo", TestMode.DEMO)
        self._mode_chooser.addOption("Launcher RPM Test", TestMode.LAUNCHER_RPM_TEST)
        SmartDashboard.putData("Test Mode", self._mode_chooser)

        self._sysid_chooser.setDefaultOption("Disabled", SysIDMode.DISABLED)
        if self._robot_container.drivetrain_subsystem is not None:
            self._sysid_chooser.addOption("Drivetrain Drive", object=SysIDMode.DRIVETRAIN_DRIVE)
            self._sysid_chooser.addOption("Drivetrain Steer", SysIDMode.DRIVETRAIN_STEER)
        if self._robot_container.flywheel_subsystem is not None:
            self._sysid_chooser.addOption("Flywheel", SysIDMode.FLYWHEEL)
        if self._robot_container.feeder_subsystem is not None:
            self._sysid_chooser.addOption("Feeder Bottom", SysIDMode.FEEDER_BOTTOM)
            self._sysid_chooser.addOption("Feeder Top", SysIDMode.FEEDER_TOP)
        if self._robot_container.turret_subsystem is not None:
            self._sysid_chooser.addOption("Turret", SysIDMode.TURRET)
        SmartDashboard.putData("SysID Mode", self._sysid_chooser)

    def get_test_command(self) -> Command:
        match self._mode_chooser.getSelected():
            case TestMode.DISABLED:
                print("[Test Container] No test mode selected, so no commands will be run")
                return Command()

            case TestMode.MOTOR:
                ...

            case TestMode.SYSID:
                match self._sysid_chooser.getSelected():
                    case SysIDMode.DISABLED:
                        print("[Test Container] No sysid mode selected, so no commands will be run")
                        return Command()

                    case SysIDMode.DRIVETRAIN_DRIVE:
                        return self._sysid_container.get_drivetrain_sysid("drive")

                    case SysIDMode.DRIVETRAIN_STEER:
                        return self._sysid_container.get_drivetrain_sysid("steer")

                    case SysIDMode.FLYWHEEL:
                        return self._sysid_container.get_flywheel_sysid()

                    case SysIDMode.FEEDER_BOTTOM:
                        return self._sysid_container.get_feeder_sysid("bottom")

                    case SysIDMode.FEEDER_TOP:
                        return self._sysid_container.get_feeder_sysid("top")

                    case SysIDMode.TURRET:
                        return self._sysid_container.get_turret_sysid()

                    case _:
                        return Command()

            case TestMode.DEMO:
                return self.get_demo_command()

            case TestMode.LAUNCHER_RPM_TEST:
                ...

            case _:
                return Command()
```

## Using SysID Data
Once these SysID routines have been run, data is stored on the RoboRio (or attached USB storage device). Use the "DataLog tool" app to download the data file from the RoboRio, then open that file in the "SysId" app.

Once obtaining the needed PID values, plug them into your `constants.py`.

See the [WPIlib docs](https://docs.wpilib.org/en/stable/docs/software/advanced-controls/system-identification/index.html) for more information.

## Manual Tuning
For some mechanisms, SysID is not the best way to tune it. Instead, you should use Phoenix Tuner X to do so, through trial and error.

See [Tuner X documentation](https://v6.docs.ctr-electronics.com/en/stable/docs/tuner/index.html) for more information.