# Commenting Guide

The goal of comments are to make it easy for someone else to use your code and for someone who hasn't seen your code but knows how to program a robot to figure out how your code works

## Classes
Comment the top of classes to describe the class' purpose, and what paramaters it takes
```python
class SC_Pathfinding:
    """
    A library for handling the pathfinding of the robot

    Can do april tag math, and return commands for pathfinding

    Parameters:
        - drivetrain_subsystem (Subsystem): The drivetrain subsystem for adding command requirements
        - pose_supplier (Callable[[], Pose2d]): Function to get the robot's current pose
        - output (Callable[[ChassisSpeeds], None]): Function to output chassis speeds to the drivetrain
        - alignment_controller (PathFollowingController): The controller to use for final alignment
    """
    def __init__(self, drivetrain_subsystem: commands2.Subsystem, pose_supplier: Callable[[], Pose2d], output: Callable[[ChassisSpeeds], None], alignment_controller: PathFollowingController) -> None:
      --snip--
```

## Functions
> [!NOTE]
> we use a modified version of [this](https://google.github.io/styleguide/pyguide.html#38-comments-and-docstrings) (google's docstring style) for our docstrings.
> more on the differences [here](/Docstring_Guide.md)
Comment the top of functions to describe the purpose of the function, what paramaters it takes (with the types of the arguments*), and what it returns:
```python
    def get_final_alignment_command(self, target: Pose2d) -> commands2.Command:
        """
        Returns a command to align the robot to a target pose

        Parameters:
            - target (`Pose2d`): The target pose to align to
            - defer (`bool`): Whether to defer the command

        Returns:
            - Command: The command to align to the target
        """
        --snip--
```
_*we also put them in a little inline code block_

## Complex Code
Comments should also be used to explain unique or complex parts of code. Especially when the snippet is complex, potentially hard to understand, or to explain why the snippet is needed.
```python
# If the motor_type is minion, it needs a talon FXS controller to be able to set the correct commutation
# There is no communtation for the falcon, so use a talon FX controller instead
# The portion for the external encoder is here, but the rest of the configuration is in the PowerMotor class
if type(self._motor_config) is TalonFXSConfiguration:
    if self._encoder is not None:
        self._motor_config.external_feedback = ExternalFeedbackConfigs() \
            .with_rotor_to_sensor_ratio(gear_ratio) \
            .with_feedback_remote_sensor_id(self._encoder.device_id) \
            .with_external_feedback_sensor_source(ExternalFeedbackSensorSourceValue.REMOTE_CANCODER)

elif type(self._motor_config) is TalonFXConfiguration:
    if self._encoder is not None:
        self._motor_config.feedback = FeedbackConfigs() \
            .with_rotor_to_sensor_ratio(gear_ratio) \
            .with_feedback_remote_sensor_id(self._encoder.device_id) \
            .with_feedback_sensor_source(FeedbackSensorSourceValue.REMOTE_CANCODER)
else:
    raise ValueError(f"Invalid motor type: {motor_config.motor_type}")
```
