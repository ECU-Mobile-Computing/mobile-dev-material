# Assignment: Single-Page MVVM App in .NET MAUI

## Overview

Build a single-page .NET 11 MAUI mobile app that demonstrates the Model-View-ViewModel (MVVM) pattern using CommunityToolkit.Mvvm. The app will be limited to one screen only, but it will include real MVVM architecture, robust data binding, and a button tied to a command.

This assignment is meant to prove that you understand how the View, ViewModel, and Model work together in a MAUI app without placing UI logic directly in the page.

## Required technology stack

- .NET 11 MAUI app
- CommunityToolkit.Mvvm package
- One mobile page only (single screen; no second page or navigation flow)
- Use dependency injection for the ViewModel
- Use a separate Model class
- Use XAML bindings from the View to the ViewModel
- Include at least one button bound to a generated command

## Scenario

Create a mobile app page called a "Daily Study Planner" that lets a user enter information and see a calculated result update immediately on the screen.

Required Elements:
- Student name
- Course name
- Study goal in hours
- Focus area selection
- Daily progress checkbox
- Summary message
- Save or update button

## Core requirements

Your app must include all of the following:

1. .NET 11 MAUI project
   - Target a .NET 11 MAUI app.
   - Keep the entire assignment to one page only.

2. CommunityToolkit.Mvvm
   - Add and use the CommunityToolkit.Mvvm package.
   - Inherit from `ObservableObject`.
   - Use `[ObservableProperty]` for bound properties.
   - Use `[RelayCommand]` for button actions.

3. Model
   - Create a separate model class that represents the app data.
   - The model should hold relevant data, not UI logic.
   - Example: `StudentProfile`
     - Properties
       - StudentName
       - Course
       - StudyHoursGoal
       - FocusArea
       - IsDailyGoalAchieved
       - StatusMessage

4. ViewModel
   - Create a dedicated ViewModel class.
   - You must use the StudentProfile model for property persistence instead of private backer fields.
     - https://learn.microsoft.com/en-us/dotnet/communitytoolkit/mvvm/observableobject
   - Expose properties for bound UI values.
   - Leverage the CommunityToolkit.Mvvm package
   - Include command logic for button actions.
     - SavePlan - updates Status Message
     - Reset    - resets all fields to original dependency injected values
   - Keep the ViewModel responsible for updating values and responding to user actions.

5. Dependency injection
   - The page must receive the ViewModel through dependency injection instead of creating it directly in code-behind.
   - The ViewModel must be prepopulated with data from the model before it is dependency injected. e.g. StudentName, Course, etc...

   - Example pattern:

   ```csharp
   public partial class MainPage : ContentPage
   {
       public MainPage(MainViewModel viewModel)
       {
           InitializeComponent();
           BindingContext = viewModel;
       }
   }
   ```

   - The ViewModel should be registered in DI from the app startup code.

6. Data binding
   - Bind the following elements to properties in the ViewModel.
   - Examples include:
     - text box bound to `StudentName`
     - picker bound to `Course`
       - picker must be bound to at least 3 different courses
     - checkbox bound to `IsDailyGoalAchieved`
     - label bound to `StatusMessage`
     - entry bound to `StudyHours`
     - entry bound to `FocusArea`
     - save button bound to a command
     - reset button bound to a command
   - Every bound element must be logically connected to the ViewModel.

7. Save Button command
   - Must update the StatusMessage label using the following string interpolated example.
   - $"{Name} studied for {StudyHours} hours in {SelectedCourse} focusing on {}."

8. Single-page constraint
   - No second page is allowed.
   - No navigation between pages.
   - No code-behind click handlers used for business logic.

## Suggested app behavior

Your page could look and behave like this:

- User enters a name and course
- User selects a focus area or study goal
- User enters a number such as study hours or task count
- The Save button updates the summary text 
- The StatusMessage label shows live feedback such as `Brian Dietrick studied for 5 hours in Mobile Development focusing on MVVM.`
- A reset button clears the form

## Recommended project structure

```text
MyApp/
├── Models/
│   └── StudentProfile.cs
├── ViewModels/
│   └── MainViewModel.cs
├── App.xaml.cs
├── MauiProgram.cs
├── Views/
│   └── MainPage.xaml
│   └── MainPage.xaml.cs
└── Resources/
```

## Minimum UI requirements

Add some styling to the XAML page.  The completed screen should feel like a polished mobile form, not a bare demo.

## Assignment deliverable

Submit the final MAUI project showing:

- the single page working correctly
- the ViewModel receiving data via DI
- CommunityToolkit.Mvvm in use
- the command reacting to user input
- the UI updating from bound properties

## Mockup of the final page UI

Use this mockup as a visual reference for the layout.

```text
+--------------------------------------------------+
| Daily Study Planner                              |
|--------------------------------------------------|
| Student Name:                                    |
| [ Jane Student                     ]             |
|                                                  |
| Course:                                          |
| [ Mobile App Development           ]             |
|                                                  |
| Study Hours Goal:                                |
| [ 3                                ] hours       |
|                                                  |
| Focus Area:                                      |
| [ UI Design                      v ]             |
|                                                  |
| [ ] I have completed today's setup               |
|                                                  |
| [ Save Plan ]      [ Reset ]                     |
|                                                  |
| Status Message:                                  |
| Ready for a productive study session.            |
+--------------------------------------------------+
```

Your exact style and colors may vary, but the screen should clearly include the same types of components and layout patterns.

## Acceptance checklist

Before submitting, check the following:

- [ ] .NET 11 MAUI project
- [ ] Single page only
- [ ] CommunityToolkit.Mvvm installed and used
- [ ] Model class exists and is separate from the ViewModel
- [ ] ViewModel is dependency injected into the page
- [ ] At least five UI elements are bound to ViewModel properties
- [ ] Save button is bound to a command
- [ ] Reset button is bound to a command
- [ ] App updates based on user input
- [ ] No business logic is placed directly in the page code-behind
- [ ] UI is clean and mobile-friendly

## Grading rubric (100 points)

### 1. MVVM architecture and separation of concerns (25 points)
- 25: Model, View, and ViewModel are clearly separated and used correctly
- 15: Mostly correct structure but some logic is mixed into the UI
- 5: Weak separation of concerns
- 0: Not using MVVM correctly

### 2. CommunityToolkit.Mvvm implementation (20 points)
- 20: `ObservableObject`, `[ObservableProperty]`, and `[RelayCommand]` are used correctly
- 10: Some MVVM toolkit features are used, but with errors or incomplete setup
- 0: CommunityToolkit.Mvvm is missing or not used meaningfully

### 3. Dependency injection and code quality (15 points)
- 15: ViewModel is registered and injected into the page
- 8: ViewModel is created in a mostly acceptable way, but DI is incomplete or inconsistent
- 0: ViewModel is created directly in the page with no DI

### 4. Data binding quality and required elements (25 points)
- 25: At least five bound elements are present and function correctly
- 15: Some bindings are present, but missing required elements or not fully working
- 5: Minimal binding is attempted
- 0: No meaningful data binding

### 5. Command behavior and UI behavior (15 points)
- 15: Button command updates the page, validates input, and clearly responds to user actions
- 8: Command works but is limited or not fully integrated
- 0: Button does not use a meaningful command

### 6. Presentation and polish (10 points)
- 10: Clean layout, intuitive form design, readable labels, mobile-friendly spacing
- 5: Basic but rough presentation
- 0: Poor usability or broken layout

## Suggested rubric summary

| Category | Points |
| --- | ---: |
| MVVM architecture | 25 |
| CommunityToolkit.Mvvm | 20 |
| Dependency injection | 15 |
| Data binding | 25 |
| Command behavior | 15 |
| Presentation and polish | 10 |
| Total | 100 |
