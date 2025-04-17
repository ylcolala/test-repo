Subject: SonarQube Integration Issue – Investigation & Suggested Fixes

Hi [Team/Name],

I’ve investigated the issue where no archive is shown in the EE portal due to a lack of SonarQube integration. Below are the possible causes and corresponding resolutions:

Problem 1: No Integration with SonarQube
The EE portal does not display archived data, indicating a potential SonarQube integration issue. Possible reasons and suggested actions include:

SonarQube is Disabled in Bitbucket Server

Fix: Ensure that the SonarQube plugin is installed and enabled in Bitbucket Server.​

Navigate to Bitbucket Server Admin > Manage Add-ons.

Search for the SonarQube for Bitbucket Server plugin and enable it if it's disabled.

No SonarQube Project Created for the Component

Fix: Create a corresponding SonarQube project for the component.​

In SonarQube, go to Projects > Create Project.

Provide the necessary details such as project key and name.

Incorrect Project Key Configured in SonarQube

Fix: Verify and update the project key in SonarQube to match the Bitbucket repository.​

In SonarQube, navigate to Project Settings > General Settings > DevOps Platform Integration.

Ensure that the Project Key and Repository Slug match those of your Bitbucket repository.

No SonarQube Report Due to Pipeline Failure

Fix: Investigate the component's pipeline build logs in Bitbucket to identify and resolve any errors that may prevent SonarQube analysis from running successfully.​

Problem 2: No Component Report Data in SonarQube Portal
Even if integration is set up, no report data is shown for the component in the SonarQube portal.

Possible Cause: The component’s pipeline may not be triggering a SonarQube scan or report generation.​

Fix: Raise a Change Management Process (CMP) request to enable or fix the component's pipeline and ensure it publishes the SonarQube report properly.​

