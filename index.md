# <code>WorkItemTitleDefinition</code>

When creating or updating Work Schedule via Karbon APIs, you will come across `WorkItemTitleDefinition` property. This property accepts a JSON string that defines the naming convention of the Title of a Work Item. Once saved, the definition will be used to name any new Work Item created by the Work Schedule.

The JSON String used as the definition is an array of multiple objects. Each object represents a part of the Work Item Title text and its order dictates where, on the Work Item Title, the text will appear. 

## Properties of the objects

The objects used in the JSON String have 4 properties. These properties can be in any order.

* **`Variable`**
* **`Format`**
* **`Text`**
* **`Offset`**

### `Variable` <sup>[string]</sup>

!!! warning

    Set the value of `Variable` property to `null`, if you want to show static text instead of a calculated value.

Karbon lets you use one of the following variables with `Variable` property in `WorkItemTitleDefinition`. Wherever these variables are mentioned in the definition, they will be replaced by calculated values when appearing as a the Title of the Work Item.

* ***`RepeatPeriod`*** represents the period for which this Work Item is created. 

* ***`StartDate`*** represents the date at which this Work Item should start.

* ***`EndDate`*** represents the date at which this Work Item should end.

*Example*:

- "Variable": "StartDate"
- "Variable": "EndDate"
- "Variable": "RepeatPeriod"

### `Format` <sup>[string]</sup>

!!! warning

    If the value of `Variable` property is `null`, the value of `Format` property must also be `null`.

Once you have selected the variable you want to use, you will need to define the format of the variable. Since Karbon currently offers datetime variables only, you can choose between one of the following datetime formats.


| For `StartDate` and `EndDate` | For `RepeatPeriod` |
| ----------------------------- | ------------------ |
| DD MMM, YYYY                  | DD MMM, YYYY - DD MMM, YYYY |
| DD MMM                        | DD MMM - DD MMM             |
| MMM YYYY                      | MMM YYYY - MMM YYYY |
| MMM                           | MMM - MMM |
| YYYY                          | YYYY - YYYY |

*Example*:

- "Format": "DD MMM"
- "Format": "MMM YYYY - MMM YYYY"

### `Text` <sup>[string]</sup>

!!! warning

    If you want to set the value of `Text` property, then the value of `Variable` and `Format` properties should be `null`. Similarly, the value of `Offset` property should be `0`.

    Conversely, if you want to set the value of `Variable` property, then the value `Text` property should be `null`. 

If you don't want to use calculated value in an object, then you can use static text instead using the `Text` property.

*Example*:

- "Text": "Payroll for the current month"
- "Text": "Payroll for the month of "

### `Offset` <sup>[integer]</sup>

!!! warning

    - `Offset` property only work in conjunction with `Variable` property. 
    - If the value of the `Variable` property is `null`, the value of the `Offset` property must be `0`.

The value specified in the `Offset` property lets you increment (or decrement) the dates specified in the `Variable` property. The behaviour of `Offset` property for each variable is given below.

=== "For `StartDate` and `EndDate`"

    !!! note

        - If you provide a positive value to the `Offset` property, it will move the dates forward by the number of days specified as the value.
        - If you provide a negative value to the `Offset` property, it will move the dates backward by the number of days specified as the value.
        - If the value of the `Offset` property is `0`, the dates will be unchanged.

    The value of the the `Offset` property affects the `day` function of the *`StartDate`* and the *`EndDate`*. 

    When you use the `Offset` property with these variables, you will be able to move the dates forward (or decrement) by the number of days specified in the value of the `Offset` property.

    For example, say the *`StartDate`* of the Work Item is 12 July, 2022. But you want the Work Item Title to read *10 JUL, 2022*. You will set the `Variable` property to *`StartDate`* and `Offset` property to `-2`. 

    *Example*:

    - "Offset": 5
    - "Offset": 0
    - "Offset": -2

=== "For `RepeatPeriod`"

    !!! note

        - `Offset` property for *`RepeatPeriod`* variable can only accept `-1`, `0`, or `1` as its value.
        - If you the value of `Offset` property is `-1`, the Work Item Title will show Previous period.
        - If you the value of `Offset` property is `1`, the Work Item Title will show Next period.
        - If you the value of `Offset` property is `0`, the Work Item Title will show Current period.

    The value of the the `Offset` property lets you move the value of *`RepeatPeriod`*, a period forward (or backward). 

    For example, say the *`RepeatPeriod`* of the Work Item is 12 July, 2022 - 11 NOV, 2022. But you want the Work Item Title to read *12 APR, 2022 - 11 JUL, 2022*. You will set the `Variable` property to *`RepeatPeriod`* and `Offset` property to `-1`.

    *Example*:

    - "Offset": 1   
    - "Offset": 0
    - "Offset": -1     

## Bringing it together with an example

**Usecase**

Say you want a Work Schedule to create Payroll Work Items for each month. You might want the name of the Work Item Title to explicitly mention that it is for *Payroll*. You might also want to mention the *Start Date* and the *End Date* of the Work Item on its Title. 

To define this naming convention for the Work Item Title, you will write the JSON string for the `WorkItemTitleDefinition` value as below:

    "[{"Text": "Payroll Starting at ", "Variable": null, "Format": null, "Offset": 0},
    {"Text": null, "Variable": "StartDate", "Format": "DD MMM, YYYY", "Offset": 0},
    {"Text": " and Ending at ", "Variable": null, "Format": null, "Offset": 0},
    {"Text": null, "Variable": "EndDate", "Format": "DD MMM, YYYY", "Offset": 0}]"

If the Start Date of the Work Item is 05 July, 2022 and the End Date of the Work Item is 12 July, 2022, above definition will render the Title of the Work Item as follows:

    Payroll Starting at 05 JUL, 2022 and Ending at 12 JUL, 2022