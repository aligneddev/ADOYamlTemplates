# Development Process

```mermaid
flowchart TD
    Jira["Create Work Ticket"] -->
    Refine["Define work, Improve requirements"] -->
    Kickoff["Assign to Developer(s) and share information"] --> Refine
    Kickoff -->
    Start["Developer Start: Create Git Branch, Work, Push"] -->
    PR["Create Pull Request(PR)"] --> Trigger
    PR --> Feedback
    Feedback{"Feedback, Validation Pipeline and Approval"}
    Feedback -- No --> Stop["Stop"]
    Feedback -- Yes --> Merge

    Trigger["Pipeline Triggered"] --> 
    BuildTest["BuildAndTest Stage"] --> 
    CommonBuildTest["netBuildAndTest.yml@AdoCommonTools"] --> NugetInstall["Nuget Install"] --> 
    NugetRestore["Nuget Restore"] --> 
    DotNetBuild["Build"] --> 
    RunTests["Run Tests"] -->
    Success{"Success"}
    Success -- No --> Feedback
    Success -- Yes --> Feedback

    Merge["Merge to Main"] -->
    PublishDev["PublishDevelopment Stage"] -->
    PublishProd["PublishProd Stage"] --> 
    Approval["Wait for Approval"] --> 
    Publish["Publish to Environment, based on type"]
    Retain["Retain deployment"] -->
    TagGitRepo["Tag Commit with ADO API"] -->
    Validate["Validate PROD"]
```

Note: You can convert this to draw.io, see https://www.drawio.com/blog/mermaid-diagrams