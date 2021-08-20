---
title: "Running Hackathon by Yourself with GitHub Actions, Microsoft 365 and Power Platform"
slug: running-hackathon-by-yourself-with-gha-m365-and-pp
description: "In this post, I'm going to discuss how to automate all the event management processes to run a hackathon, using GitHub Actions, Microsoft 365 and Power Platform."
date: "2021-08-20"
author: Justin-Yoo
tags:
- azure
- github-actions
- power-platform
- azure-functions
cover: https://sa0blogs.blob.core.windows.net/devkimchi/2021/08/running-hackathon-by-yourself-with-gha-m365-and-pp-00-en.png
fullscreen: true
---

Generally speaking, an off-line hackathon event takes place with people getting together at the same time and place for about two to three nights, intensively. On the other hand, all events have turned into online-only nowadays, and there's no exception for the hackathon events either. To keep the same event experiences, hackathon organisers use many online collaboration tools. In this case, almost the same number of event staff members are necessary. What if you have limited resources and budget and are required to run the online hackathon event?

For two weeks, I recently ran an online-only hackathon event called [HackaLearn][hackalearn] from August 2, 2021. This post is the retrospective of the event from the event organiser's perspective. If anyone is planning a hackathon with a similar concept, I hope this post could be helpful.


## The Background ##

In May 2021 at [//Build][build2021] Conference, [Azure Static Web Apps (ASWA)][az swa] became generally available. It's relatively newer than the other competitors' ones meaning it is less popular than the others. This HackaLearn event is one of the practices to promote ASWA. The idea was simple. We're not only running a hackathon event but also offering the participants learning experiences with [Microsoft Learn][az swa learn] to participants so that they can feel how convenient ASWA is to use. Therefore, all participants can learn ASWA and build their app with ASWA &ndash; this was the direction.

![HackaLearn Banner][image-01]

In fact, the first HackaLearn event was held in Israel, and other countries in the EMEA region have been running this event. I also borrowed the concept and localised the format for Korean uni students. With support from [Microsoft Learn Student Ambassadors (MLSA)][mlsa] and [GitHub Campus Experts (GCE)][gce], they review the participants pull requests and external field experts were invited as mentors and ran online mentoring sessions.


## The Problems ##

As mentioned above, running a hackathon event requires intensive, dedicated and exclusive resources, including time, people and money. However, none of them was adequate. I've got very limited resources, and even I couldn't dedicate myself to this event either. I was the only one who could operate the event. Both [MLSAs][mlsa] and [GCEs][gce] were dedicated for PR reviews and mentors for mentoring sessions. Automating all the event operation processes was the only answer for me.

> How can I automate all the things?

For me, finding out the solution is the key focus area throughout this event.


## The Constraints ##

* **No Website for Hackathon**

  🚨 There was no website for HackaLearn. Usually, the event website is built on a one-off basis, which seems less economical.
  <br>
  👉 Therefore, I decided to use the GitHub repository for the event because it offers many built-in features such as Project, Discussions, Issues, Wiki, etc.

* **No Place for Participant Registration**

  🚨 There was no registration form.
  <br>
  👉 Therefore, I decided to use [Microsoft Forms][spo forms].

* **No Database for Participant Management**

  🚨 There was no database for the participant management to record their challenge progress.
  <br>
  👉 Therefore, instead of provisioning a database instance, I decided to use [Microsoft Lists][spo lists].

* **No Dashboard for Teams and Individuals Progress Tracking**

  🚨 There was no dashboard to track each team's and each participant's progress.
  <br>
  👉 So instead, I decided to use their team page by merging their pull requests.

I've defined the overall business process workflow in the following sequence diagrams. All I needed is to sort out those limitations stated above. To me, it was Power Platform and GitHub Actions with minimal coding efforts and maximum outcomes.


## The Plans for Process Automation ##

The limitations above have become opportunities to experiment with the new process automation!

* The event itself uses the GitHub repository, meaning all PRs and issues can be handled by [GitHub Actions][gha].
* [Microsoft 365][m365] services like [Microsoft Forms][spo forms] and [Microsoft Lists][spo lists] are used for data input and storage.
* [Power Automate][pau] is one of the [Power Platform][pp] services and is used for the core of process automation.

So, the GitHub repository and Microsoft 365 services are fully integrated with GitHub Actions workflows and Power Automate workflows. As a result, I was able to save a massive amount of time and money with them.


## The Result &ndash; Participant Registration ##

The first automation process I worked on was about storing data. The participant details need to be saved in [Microsoft Lists][spo lists]. When a participant enters their details through [Microsoft Forms][spo forms], then a [Power Automate][pau] workflow is triggered to process the registration details. At the same time, the workflow calls a GitHub Actions workflow to create a team page for the participant. Here's the simple sequence diagram describing this process.

![Registration Sequence Diagram][image-02]

The overall process is divided into two parts &ndash; one to process participant details in the Power Automate workflow, and the other to process the details in the GitHub Actions workflow.


### Power Automate Workflow ###

Let's have a look at the Power Automate part. When a participant registers through [Microsoft Forms][spo forms], the form automatically triggers a Power Automate workflow. The workflow checks the email address whether the participant has already registered or not. If the email doesn't exist, the participant details are stored to [Microsoft Lists][spo lists].

![Registration Flow 1][image-05]

Then it generates a team page. Instead of creating it directly from the Power Automate workflow, it builds the page content and sends it to the GitHub Actions workflow. The [`workflow_dispatch`][gha events workflowdispatch] event is triggered for this action.

![Registration Flow 2][image-06]

Finally, the workflow sends a confirmation email. In terms of the name, participants may register themselves with English names or Korean names. Therefore, I need logic to check the participant's name. If the participant name is written in English, it should be `[Given Name] [Surname]` (with a space; eg. Justin Yoo). If it's written in Korean, it should be `[Surname][Given Name]` (without a space; eg. 유저스틴). The red-boxed actions are responsible for identifying the participant's name. It may be simplified by adopting a custom connector with an Azure Functions endpoint.

![Registration Flow 3][image-07]


### GitHub Actions Workflow ###

As mentioned above, the Power Automate workflow calls a GitHub Actions workflow to generate a team page. Let's have a look. The `workflow_dispatch` event takes the input details from Power Automate, and they are `teamName` and `content`.

https://gist.github.com/justinyoo/29b678fecea0c2acdbe829644a55cbb5?file=01-on-team-page-requested-1.yaml&highlights=4,6,10

As GitHub [Marketplace][gha marketplace] has various types of Actions, I can simply choose one to create the team page, commit the change and push it back to the repository.

https://gist.github.com/justinyoo/29b678fecea0c2acdbe829644a55cbb5?file=01-on-team-page-requested-2.yaml&highlights=2,11-14,17

Now, I've got the registration process fully automated. Let's move on.


## The Result &ndash; Challenges Update ##

In this HackaLearn event, each participant was required to complete six challenges. Every time they finish one challenge, they MUST update their team page and create a PR to reflect their progress. As there are not many differences between the challenges, I'm going to use the Social Media Challenge as an example.

Here's the simple sequence diagram describing the process.

![Social Media Challenge Sequence Diagram][image-03]

1. After the participant posts a post to their social media, they update their team page and raise a PR. Then, a GitHub Actions workflow labels the PR with `review-required` and assigns a reviewer.
2. The assigned reviewer checks the social media post whether it's appropriately hashtagged with `#hackalearn` and `#hackalearnkorea`.
3. Once confirmed, the reviewer adds the `review-completed` label to the PR. Then another GitHub Actions workflow automatically removes the `review-required` label from the PR.
4. The reviewer completes the review by leaving a comment of `/socialsignoff`, and the comment triggers another GitHub Actions workflow. The workflow calls the Power Automate workflow that updates the record on [Microsoft Lists][spo lists] with the challenge progress.
5. The Power Automate workflow calls back to another GitHub Actions workflow to add `record-updated` and `completed-social` labels to the PR and remove the `review-completed` labels from it.
6. If there is an issue while updating the record, the GitHub Actions workflow adds the `review-required` label so that the assigned reviewer starts review again.


### GitHub Actions Workflow ###

As described above, there are five GitHub Actions workflow used to handle this request.


#### Challenge Update PR ####

The GitHub Actions workflow is triggered by the participant requesting a new PR. The event triggered is [`pull_request_target`][gha events pullrequesttarget], and it's only activated when the changes occur under the `teams` directory.

https://gist.github.com/justinyoo/29b678fecea0c2acdbe829644a55cbb5?file=02-on-challenge-submitted-1.yaml&highlights=4,10

If the PR is created later than the due date and time, the PR should not be accepted. Therefore, A PowerShell script is used to check the due date automatically. Since the PR's `created_at` value is the UTC value, it should be converted to the Korean local time, included in the PowerShell script.

https://gist.github.com/justinyoo/29b678fecea0c2acdbe829644a55cbb5?file=02-on-challenge-submitted-2.yaml&highlights=12-16

If the PR is over the due, it's immediately rejected and closed.

https://gist.github.com/justinyoo/29b678fecea0c2acdbe829644a55cbb5?file=02-on-challenge-submitted-3.yaml&highlights=2,10,25

If it's before the due, label the PR, leave a comment and randomly assign a reviewer.

https://gist.github.com/justinyoo/29b678fecea0c2acdbe829644a55cbb5?file=02-on-challenge-submitted-4.yaml&highlights=2,9,24


#### Challenge Review Completed ####

The assigned reviewer confirms the challenge and labels the result. This labelling action triggers the following GitHub Actions workflow.

https://gist.github.com/justinyoo/29b678fecea0c2acdbe829644a55cbb5?file=03-on-challenge-labelled-1.yaml&highlights=6,7


#### Challenge Review Approval ####

Commenting like `/socialsignoff` for the social media post challenge automatically triggers the following GitHub Actions workflow, with the event of [`issue_comment`][gha events issuecomment].

https://gist.github.com/justinyoo/29b678fecea0c2acdbe829644a55cbb5?file=04-on-challenge-review-commented-1.yaml&highlights=4-6

The first step of this workflow is to check whether the commenter is the assigned reviewer, then find out which challenge is approved. The `review-completed` label MUST exist on the PR, and the commenter MUST be in the reviewer list (`secrets.PR_REVIEWERS`).

https://gist.github.com/justinyoo/29b678fecea0c2acdbe829644a55cbb5?file=04-on-challenge-review-commented-2.yaml&highlights=16-18

If all conditions are met, the workflow takes one action based on the type of the challenge. Each action calls a Power Automate workflow to update the record on [Microsoft Lists][spo lists], send a confirmation email, and calls back to another GitHub Actions workflow.

https://gist.github.com/justinyoo/29b678fecea0c2acdbe829644a55cbb5?file=04-on-challenge-review-commented-3.yaml&highlights=2,9,16,23,30,37


#### Challenge Complete or Further Review ####

This GitHub Actions workflow completes the challenge, triggered by a Power Automate workflow through the [`workflow_dispatch`][gha events workflowdispatch] event. Power Automate sends values of `prId`, `labelsToAdd`, `labelsToRemove` and `isMergeable`.

https://gist.github.com/justinyoo/29b678fecea0c2acdbe829644a55cbb5?file=05-on-challenge-completed-1.yaml&highlights=4,6,10,14,18

The first action is to add labels to the PR and remove labels from the PR.

https://gist.github.com/justinyoo/29b678fecea0c2acdbe829644a55cbb5?file=05-on-challenge-completed-2.yaml&highlights=8

And finally, this action merges the PR. If there's an error on the Power Automate workflow side, the `isMeargeable` value MUST be `false`, meaning it won't execute the merge action.

https://gist.github.com/justinyoo/29b678fecea0c2acdbe829644a55cbb5?file=05-on-challenge-completed-3.yaml&highlights=8-9


### Power Automate Workflow ###

The challenge approval workflow calls this Power Automate workflow. Firstly, it checks the type of challenges. If no challenge is identified, it does nothing.

![Challenge Update Flow 1][image-08]

If the challenge is linked to the registered participant's GitHub ID, update the record on [Microsoft Lists][spo lists]; otherwise, do nothing.

![Challenge Update Flow 2][image-09]

Finally, it sends a confirmation email using a different email template based on the number of challenges completed.

![Challenge Update FLow 3][image-10]


## Other Power Automate Workflows ##

Previously described Power Automate workflows are triggered by GitHub Actions for integration. However, there are other workflows only for management purposes. As most processes are similar to each other, I'm not going to describe them all. Instead, it's the total number of workflows that I used for the event, which is 15 in total.

![List of Power Automate Workflows][image-04]


## Stats ##

Now all my business processes are fully automated. As an operator, I can focus on questions and PR reviews, but nothing else.

Here are some numbers related to this HackaLearn event.

* `14`: Number of days for HackaLearn
* `171`: Total number of participants
* `62`: Total number of participants who completed Cloud Skills Challenge
* `21`: Total number of teams who uploaded social media posts
* `17`: Total number of teams who completed building Azure Static Web Apps
* `16`: Total number of teams who completed provided their GitHub repository
* `20`: Total number of teams who published their blog post as a retrospective
* `13`: Total number of teams who completed all six challenges


## Side Events ##

During the event, we ran live hands-on workshop for [GitHub Actions][gha] and [Azure Static Web Apps][az swa] led by a [GCE][gce] and an [MLSA][mlsa] respectively.

* Live Hands-on: GitHub Actions (in Korean)

  https://youtu.be/e_elLW6uNSc

  <br>

* Live Hands-on: Azure Static Web Apps (in Korean)

  https://youtu.be/Hxkv6AjAisY

  <br>

* Live Hands-on: Azure Static Web Apps with Headless CMS (in Korean)

  https://youtu.be/x3j3mDblqMY

  <br>

---

So far, I summarised what I've learnt from this event and what I've done for workflow automation, using [GitHub Actions][gha], [Microsoft 365][m365] and [Power Automate][pau]. Although there are lots of spaces to improve, I managed to run the online hackathon event with a fully automated process. I can now do it again in the future.

Specially thanks to [MLSAs][mlsa] and [GCEs][gce] to review all the PRs, and mentors who answered questions from participants. Without them, regardless of the fully automated workflows, this event wouldn't be successfully running.


[image-01]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/08/running-hackathon-by-yourself-with-gha-m365-and-pp-01-en.png
[image-02]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/08/registration-en.png
[image-03]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/08/challenge-social-en.png
[image-04]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/08/running-hackathon-by-yourself-with-gha-m365-and-pp-04.png
[image-05]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/08/running-hackathon-by-yourself-with-gha-m365-and-pp-05.png
[image-06]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/08/running-hackathon-by-yourself-with-gha-m365-and-pp-06.png
[image-07]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/08/running-hackathon-by-yourself-with-gha-m365-and-pp-07.png
[image-08]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/08/running-hackathon-by-yourself-with-gha-m365-and-pp-08.png
[image-09]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/08/running-hackathon-by-yourself-with-gha-m365-and-pp-09.png
[image-10]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/08/running-hackathon-by-yourself-with-gha-m365-and-pp-10.png


[hackalearn]: https://github.com/devrel-kr/HackaLearn/blob/main/README.en.md

[build2021]: https://mybuild.microsoft.com/home?WT.mc_id=power-39037-juyoo

[az swa]: https://docs.microsoft.com/azure/static-web-apps/overview?WT.mc_id=power-39037-juyoo
[az swa learn]: https://docs.microsoft.com/learn/paths/azure-static-web-apps/?WT.mc_id=power-39037-juyoo

[mlsa]: https://studentambassadors.microsoft.com/?WT.mc_id=power-39037-juyoo
[gce]: https://githubcampus.expert

[m365]: https://www.microsoft.com/microsoft-365?WT.mc_id=power-39037-juyoo

[spo]: https://www.microsoft.com/microsoft-365/sharepoint/collaboration?WT.mc_id=power-39037-juyoo
[spo lists]: https://www.microsoft.com/microsoft-365/microsoft-lists?WT.mc_id=power-39037-juyoo
[spo forms]: https://www.microsoft.com/microsoft-365/online-surveys-polls-quizzes?WT.mc_id=power-39037-juyoo

[gha]: https://github.com/features/actions
[gha marketplace]: https://github.com/marketplace?type=actions
[gha events workflowdispatch]: https://docs.github.com/en/github-ae@latest/actions/reference/events-that-trigger-workflows#workflow_dispatch
[gha events pullrequesttarget]: https://docs.github.com/en/github-ae@latest/actions/reference/events-that-trigger-workflows#pull_request_target
[gha events issuecomment]: https://docs.github.com/en/github-ae@latest/actions/reference/events-that-trigger-workflows#issue_comment

[pp]: https://powerplatform.microsoft.com/?WT.mc_id=power-39037-juyoo
[pau]: https://flow.microsoft.com/?WT.mc_id=power-39037-juyoo
