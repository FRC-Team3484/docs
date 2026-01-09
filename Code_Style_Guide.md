# Code Style Guide
This page documents good practices for how to format and style our Python code

## Cases
- File names are in lower snake case (`drivetrain_subsytem`)
- Function names are in Pascal Case (`PascalCase`)
- Classes are in Pascal Case (`PascalCase`)
- Constants should be in capitalized snake case (`CAPITALIZED_SNAKE_CASE`)
- Variables are in lower snake case (`lower_snake_case`)
- Namespaces are Camel Case and one word (`Namespace`)

## Formatting
- **Always type-hint variables, function paramaters, and function return types**
    ```python
    variable: int = 10

    def function(number: int) -> str:
        return f"This is now a string: {number}"
    ```
- Indentation is 4 spaces
- Private variables begin with an underscore (`_lower_snake_case`)
- Comments should have a space between the hash (`# This is a comment`)
- Functions should never be condensed to one line
- All functions should have a comment describing what it does, what paramaters it takes, and what it returns (if applicable)
- Classes should also have a comment which describes what that class does and it's paramaters
- Multiple functions in a class should be seperated by a single new line
- Sort imports first by standard Python libraries, then by generic WPIlib libraries, and then finally by

## File Names
- Subsystem files are named `<subsystem name>_subsystem_.py`
- Commands are separated into `/auton` and `/teleop` folders
  - The names should be either `auton_<command name>.py` or `teleop_<command name>.py`
